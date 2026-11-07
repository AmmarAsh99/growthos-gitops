# GrowthOS GitOps

Kubernetes manifests for the GrowthOS frontend, API, PostgreSQL database, and database migrations. All resources use the `growthos` namespace. Files stay in the repository root.

## Files

| File | Purpose |
| --- | --- |
| `namespace.yaml` | Creates the GrowthOS namespace |
| `growthos-web.yaml` | Frontend Deployment and Service |
| `growthos-api.yaml` | API Deployment and Service |
| `postgres.yaml` | PostgreSQL StatefulSet, Service, and persistent storage |
| `migration-job.yaml` | Runs `npm run db:migrate` |
| `secrets.yaml.example` | Placeholder credentials to copy locally; never apply directly |

The PostgreSQL StatefulSet keeps its existing name, `postgress`, to avoid changing its identity and persistent volume claim names.

## Credentials

Real credentials belong in the ignored `secrets.local.yaml`, not in Git. The existing local credentials were preserved in that file during cleanup. For a fresh checkout, run this PowerShell command and replace every placeholder:

```powershell
Copy-Item secrets.yaml.example secrets.local.yaml
```

Use separate random JWT secrets. Keep the database fields and `DATABASE_URL` consistent; URL-encode special characters in the URL credentials.

If real credentials were previously pushed, rotate the database password and both JWT secrets. Removing the tracked file does not erase older Git commits. Coordinate removal from Git history separately if needed. For an existing database volume, changing the Kubernetes Secret alone does not change the PostgreSQL user's password: update the database password as well, then update the Secret and restart the API. JWT rotation invalidates existing tokens.

This setup provisions secrets manually outside Git. Before automating secret delivery, configure an external secret store or encrypted secrets with the chosen GitOps controller. Do not commit the local file or apply the example.

## Before deploying

- Configure `kubectl` for the intended cluster and check `kubectl config current-context`.
- Ensure the cluster has a default StorageClass capable of provisioning the database's 5 GiB claim, or set `storageClassName` in `postgres.yaml`.
- Ensure the cluster can pull the image tags in the manifests.
- Review `CORS_ORIGIN`, `COOKIE_SECURE`, and `NODE_ENV` in `growthos-api.yaml` for your environment. Secure cookies require an appropriate HTTPS setup; the current localhost/development settings are not a production configuration.
- Back up an existing database before applying schema migrations.

## Deploy manually

Use explicit filenames so the example and local secrets are never picked up accidentally by a directory-wide apply.

```powershell
kubectl apply -f namespace.yaml
kubectl apply -f secrets.local.yaml
kubectl apply -f postgres.yaml
kubectl rollout status statefulset/postgress -n growthos --timeout=300s
kubectl apply -f migration-job.yaml
kubectl wait --for=condition=complete job/db-push -n growthos --timeout=300s
```

Only continue after the migration succeeds:

```powershell
kubectl apply -f growthos-api.yaml -f growthos-web.yaml
kubectl rollout status deployment/growthos-api-deployment -n growthos --timeout=300s
kubectl rollout status deployment/growthos-web-deployment -n growthos --timeout=300s
kubectl get pods,services,pvc -n growthos
```

If migrations fail, inspect `kubectl logs job/db-push -n growthos` and resolve the failure before continuing. A wait timeout does not stop the Job.

The Services are cluster-internal. To inspect the frontend locally:

```powershell
kubectl port-forward service/web 8080:8080 -n growthos
```

Open http://localhost:8080. Full browser-to-API access also depends on the frontend's API URL/proxy configuration; ingress and TLS are not configured in this repository.

## Later releases and migrations (manual kubectl deployment)

Update the frontend, backend, and migration image tags to the intended release. Applying a completed Job again does not rerun it, and its Pod template cannot normally be updated in place. After confirming the previous Job has finished and its logs are no longer needed, recreate it:

```powershell
kubectl delete job db-push -n growthos --ignore-not-found
kubectl apply -f migration-job.yaml
kubectl wait --for=condition=complete job/db-push -n growthos --timeout=300s
```

After success, apply the application Deployments again. Plan migrations to remain compatible with the running application during upgrades.

## Deploy with Argo CD

Before the first sync, apply `namespace.yaml` and your ignored `secrets.local.yaml` manually as shown above. Push the manifest changes to GitHub. Create an Argo CD Application pointing to `https://github.com/AmmarAsh99/growthos-gitops.git`, revision `main`, path `.`, and destination namespace `growthos`. Select the intended cluster and start with manual sync. The `.example` file is not a deployable YAML manifest; do not configure Argo CD to include it.

The manifests define this order within the Sync phase:

| Wave | Resources | Gate before proceeding |
| --- | --- | --- |
| `-2` | Namespace | Namespace exists |
| `-1` | PostgreSQL Service and StatefulSet | PostgreSQL StatefulSet is healthy |
| `0` | Migration Job (Sync hook) | Migration succeeds |
| `1` | API and frontend Deployments and Services | Application rollout |

The migration uses a `Sync` hook so the database can be created first on an empty cluster. `BeforeHookCreation` deletes the previous Job before creating the next run; its logs remain available until that deletion. A failed migration stops the sync before the application updates are applied. Existing application Pods keep running during migrations, so schema changes must remain compatible with them.

Use a full Application sync. Selective sync skips hooks; do not enable `ApplyOutOfSyncOnly=true` for this Application. The migration hook can run on subsequent full syncs even without a new migration, so `db:migrate` must safely handle already-applied migrations. After the first successful sync, enable automatic sync if desired. Argo CD handles Job recreation; the manual Job deletion commands above are only for deployments performed directly with kubectl.

If workloads already exist in another namespace, plan their migration before syncing these manifests, since adding a namespace does not move existing resources. Sync waves are interpreted by Argo CD; `kubectl apply` alone does not enforce them.

Reference: [Argo CD sync phases and waves](https://argo-cd.readthedocs.io/en/latest/user-guide/sync-waves/).
