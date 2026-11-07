# Local Kubernetes configuration

The real `k8s/secrets.yaml`, `growthos-api.yaml`, `growthos-web.yaml`, and `job.yaml`
are local, ignored files. Existing copies are preserved. These templates contain
placeholders only; do not apply the examples directory directly.

For a new checkout, copy each `*.example.yaml` to `k8s/`, removing `.example` from
the filename. Replace the registry namespace, image tag, public frontend origin,
and secret placeholders before applying. The database password inside
`DATABASE_URL` must be URL encoded and match `POSTGRES_PASSWORD`.

The image workflow uses the existing GitHub secrets `DOCKERHUB_USERNAME` and
`DOCKERHUB_TOKEN`. No new container environment variables are required.

Ignore rules do not remove earlier commits. The previously tracked secrets file
exists in Git history. Rotate real database credentials and JWT signing secrets
that were committed. Update the actual database password as well as the Kubernetes
Secret; changing a Secret alone does not change an initialized PostgreSQL password.
History cleanup, if needed, requires a coordinated rewrite of published branches
and tags. No history rewrite is performed by this change.
