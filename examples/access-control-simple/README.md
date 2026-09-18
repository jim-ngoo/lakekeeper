## Authentication to Authorization POC

This example models an existing Lakekeeper deployment that already uses Keycloak for authentication and later enables OpenFGA authorization without replacing its catalog database or Iceberg data.

The default stack uses Lakekeeper's `allowall` authorization backend. Authentication is still enforced: Lakekeeper validates Keycloak issuer, audience, signature, and token lifetime, but every authenticated principal is authorized. OpenFGA is added only through `docker-compose-authz.yaml` during the cutover.

> This is a local POC. The committed client secrets and passwords are intentionally insecure and must not be reused outside this example.

### 1. Start the AuthN-only stack

From this directory:

```bash
docker compose up -d
docker compose ps
```

For a completely fresh run, remove the example's containers and volumes first. This deletes all POC catalog and object-storage data:

```bash
docker compose down -v
```

Open Jupyter at [http://localhost:8888](http://localhost:8888), then run `03-AuthN-to-AuthZ-POC.ipynb` through section 7. The notebook bootstraps Lakekeeper, creates the warehouse and sample table, and verifies:

| Check | Expected result |
|---|---|
| No, malformed, or expired token | `401 Unauthorized` |
| Technical-client token | Catalog access succeeds |
| Trino-client token | Catalog access succeeds |
| Peter and Anna tokens | Catalog access succeeds while authorization is disabled |
| PyIceberg query | Returns the three sample rows |
| Trino queries | `SHOW`, `SELECT`, aggregation, and filtering succeed |

Useful endpoints:

* Jupyter: [http://localhost:8888](http://localhost:8888)
* Lakekeeper UI: [http://localhost:8181](http://localhost:8181)
* Swagger UI: [http://localhost:8181/swagger-ui/](http://localhost:8181/swagger-ui/)
* Keycloak: [http://localhost:30080](http://localhost:30080), using `admin` / `admin`

The example users are `peter` / `iceberg` and `anna` / `iceberg`.

The Lakekeeper UI's Preview tab reads Iceberg files directly from Silo in the browser. This example exposes Silo at [http://localtest.me:9000](http://localtest.me:9000), maps that hostname to the Docker host for in-network clients, and enables CORS. After changing from an older `http://silo:9000` or `http://silo.localhost:9000` endpoint, run `docker compose up -d --force-recreate`, then rerun notebook section 4 to update the existing warehouse storage profile in place.

### 2. Enable OpenFGA authorization

Stop writes during this procedure. Keep the existing Postgres and Silo volumes so the exercise proves an in-place cutover.

```bash
docker compose stop lakekeeper

docker compose -f docker-compose.yaml -f docker-compose-authz.yaml up -d openfga

# Install Lakekeeper's authorization model in OpenFGA.
docker compose -f docker-compose.yaml -f docker-compose-authz.yaml run --rm migrate

# Rebuild OpenFGA's structural hierarchy from the existing catalog.
docker compose -f docker-compose.yaml -f docker-compose-authz.yaml run --rm \
	--entrypoint /home/nonroot/lakekeeper migrate \
	openfga reconcile --mode add-missing

# Existing ownership, grants, and bootstrap roles cannot be reconstructed.
docker compose -f docker-compose.yaml -f docker-compose-authz.yaml run --rm \
	--entrypoint /home/nonroot/lakekeeper migrate \
	reopen-bootstrap --yes

docker compose -f docker-compose.yaml -f docker-compose-authz.yaml up -d lakekeeper
docker compose -f docker-compose.yaml -f docker-compose-authz.yaml ps
```

Resume the notebook at section 9. It bootstraps the new authorization store with the same technical identity, verifies Peter, Anna, and Trino are initially denied, then applies and tests these grants:

| Principal | Grant | Expected result |
|---|---|---|
| Peter | Warehouse `describe`, table `select` | PyIceberg read succeeds |
| Anna | None initially | PyIceberg read is denied |
| Trino service client | Warehouse `select` | Trino read succeeds |
| Anna | Table `select` in the final step | Only that table becomes readable |

The simple example gives Trino one service identity, so Trino cannot distinguish Peter from Anna. The notebook verifies end-user authorization directly with PyIceberg and verifies Trino authorization as the `trino` service principal. Use the [advanced access-control example](../access-control-advanced/) when a shared query engine must enforce end-user identity.

### Production procedure

Before applying this to the real deployment:

1. Back up Lakekeeper Postgres and OpenFGA independently, and record the current Lakekeeper image and environment.
2. Quiesce catalog writes for the migration window.
3. Deploy OpenFGA v1.11 or later with durable storage and authentication.
4. Add `LAKEKEEPER__AUTHZ_BACKEND=openfga` and the required `LAKEKEEPER__OPENFGA__*` settings without changing OIDC settings.
5. Run `lakekeeper migrate`, then `lakekeeper openfga reconcile --mode add-missing`.
6. Run `lakekeeper reopen-bootstrap --yes`, start Lakekeeper, and bootstrap as the intended initial administrator.
7. Recreate required project roles, ownership, and grants. Reconcile restores hierarchy only.
8. Run positive and negative access tests before restoring writes.

### Rollback

To return this POC to authenticated `allowall` mode while retaining its catalog and OpenFGA data:

```bash
docker compose up -d --force-recreate lakekeeper
docker compose ps
```

Do not use `docker compose down -v` for rollback because it deletes the POC data. A production rollback must restore the prior Lakekeeper configuration and follow the database/storage recovery plan captured before migration.

### Troubleshooting

```bash
docker compose logs keycloak lakekeeper
docker compose -f docker-compose.yaml -f docker-compose-authz.yaml logs openfga migrate lakekeeper
docker compose -f docker-compose.yaml -f docker-compose-authz.yaml config
```

* `401` means authentication failed. Check issuer, audience, expiry, Keycloak reachability, and the matching Lakekeeper error ID in logs.
* `403` after the cutover means authentication succeeded but OpenFGA denied the action. Check bootstrap and grants.
* A CORS error in the UI Preview tab means the browser cannot reach the warehouse's S3 endpoint or Silo is not returning CORS headers. Confirm [http://localtest.me:9000](http://localtest.me:9000) is reachable, then rerun section 4 to repair an older stored endpoint.
* Missing objects after the cutover usually mean reconcile did not run against the same Postgres catalog.
* Missing privileges after reconcile are expected: ownership, grants, and role assignments must be recreated.

### Review and commit

```bash
git status --short
git diff -- examples/access-control-simple
git add examples/access-control-simple
git commit -m "docs: add staged authn to authz poc"
```
