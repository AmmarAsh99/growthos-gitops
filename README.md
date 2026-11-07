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

## Later releases and migrations

Update the frontend, backend, and migration image tags to the intended release. Applying a completed Job again does not rerun it, and its Pod template cannot normally be updated in place. After confirming the previous Job has finished and its logs are no longer needed, recreate it:

```powershell
kubectl delete job db-push -n growthos --ignore-not-found
kubectl apply -f migration-job.yaml
kubectl wait --for=condition=complete job/db-push -n growthos --timeout=300s
```

After success, apply the application Deployments again. Plan migrations to remain compatible with the running application during upgrades.

## GitOps setup next

Configure Argo CD or Flux to reconcile the intended manifests, excluding `secrets.yaml.example`. Provision secrets before workloads. Configure controller-specific migration ordering and Job recreation before enabling automated releases; the manual commands above do not define that automation. If workloads already exist in another namespace, plan their migration before syncing these manifests, since adding a namespace does not move existing resources.
