# Auth-coverage ADO pipelines for `AzureRMAuth`

These pipelines exercise each authentication path of
`plugins/module_utils/azure_rm_common.py::AzureRMAuth` end-to-end against a
real Azure subscription. They run the minimal `auth_smoke` integration
target (create resource group → read & assert via `_info` → delete) so that
any failure is attributable to the auth code path, not module logic.

All pipelines are **on-demand only** (`trigger: none`, `pr: none`). An ADO
admin must register one Pipeline per YAML file (Pipelines → New →
Existing YAML).

## Files

| File | Auth path exercised |
|------|---------------------|
| `template-auth-pipeline.yml`            | Shared template — not registered as a pipeline. |
| `pipeline-sp-secret-env.yml`            | SP + secret via env vars (baseline; mirrors today's PR pipeline). |
| `pipeline-sp-secret-params.yml`         | SP + secret via module params (no AZURE_* env vars). |
| `pipeline-sp-certificate.yml`           | SP + X.509 certificate (`CertificateCredential`). |
| `pipeline-oidc-token.yml`               | OIDC / federated identity, in-memory token. |
| `pipeline-oidc-token-file.yml`          | OIDC / federated identity, token read from file. |
| `pipeline-msi.yml`                      | Managed identity (`auth_source=msi`). Self-hosted runner required. |
| `pipeline-cli.yml`                      | Azure CLI (`auth_source=cli`). |
| `pipeline-credential-file.yml`          | `~/.azure/credentials` default profile (`auth_source=credential_file`). |
| `pipeline-credential-file-profile.yml`  | `~/.azure/credentials` + named profile selection. |
| `pipeline-auth-source-env.yml`          | Explicit `auth_source=env`. |
| `pipeline-auto-profile-param.yml`       | `auto` precedence: `profile` module param drives `_get_profile`. |
| `pipeline-auto-default-credfile.yml`    | `auto` precedence: falls through to `~/.azure/credentials [default]`. |
| `pipeline-auto-cli-fallback.yml`        | `auto` precedence: falls all the way through to Azure CLI. |
| `pipeline-precedence-params-over-env.yml` | `auto` precedence: module params override env vars. |

`ad_user`/`password` is intentionally not covered — Azure discourages it
and it cannot satisfy MFA-required tenants.

## How params-based pipelines work

The `auth_smoke` target's `tasks/main.yml` reads optional `AUTH_SMOKE_*`
env vars (namespaced so AzureRMAuth's automatic `AZURE_*` env pickup
cannot satisfy auth on its own) and surfaces them into
`module_defaults: group/azure:`. So a "module params" pipeline just sets
`AUTH_SMOKE_CLIENT_ID`, `AUTH_SMOKE_SECRET`, `AUTH_SMOKE_TENANT`,
`AUTH_SMOKE_SUBSCRIPTION_ID`, `AUTH_SMOKE_AUTH_SOURCE`, `AUTH_SMOKE_PROFILE`
in `extraEnv` and the target does the rest. No `runme.yml` wrapper is
needed (and would be ignored by `ansible-test integration`, which only
honours `runme.sh`).

## ADO variables / secrets required

Define a variable group (e.g. `ansible-azcollection-auth`) and link it to
each pipeline. Mark all `_SECRET`/credential values as secret.

| Variable | Used by | Notes |
|----------|---------|-------|
| `AZURE_CLIENT_ID`         | most pipelines | SP appId for the secret-based SP |
| `AZURE_SECRET`            | most pipelines | SP client secret |
| `AZURE_TENANT`            | most pipelines | tenant id |
| `AZURE_SUBSCRIPTION_ID`   | all pipelines  | target subscription |
| `SUBSCRIPTION_FULL_NAME`  | template, `pipeline-cli.yml` | ADO Azure service connection name used for cleanup and `az login` |
| `AZURE_CLIENT_ID_CERT`    | `pipeline-sp-certificate.yml` | SP appId that has the certificate registered |
| `AZURE_CERT_PEM`          | `pipeline-sp-certificate.yml` | secret variable containing the full PEM (private key + cert) |
| `AZURE_CERT_THUMBPRINT`   | `pipeline-sp-certificate.yml` | hex thumbprint of the cert |
| `OIDC_SERVICE_CONNECTION` | `pipeline-oidc-token.yml`, `pipeline-oidc-token-file.yml` | Name of an ADO Azure service connection configured with **workload-identity federation** so that `AzureCLI@2 + addSpnToEnvironment: true` exposes `idToken`. |
| `MSI_POOL_NAME`           | `pipeline-msi.yml` | Name of a self-hosted ADO agent pool whose agents have a managed identity assigned. |
| `MSI_CLIENT_ID`           | `pipeline-msi.yml` | (optional) client_id of the user-assigned MI; leave empty for system-assigned. |

The SP behind `AZURE_CLIENT_ID`/`AZURE_SECRET` and the cert SP and the OIDC
SP and the MSI all need at minimum **Contributor** on the target
subscription so RG create/delete works.

## Local sanity-checking the YAML

```bash
# Lint the static structure (no live ADO required)
yq eval '.' tests/utils/ado/auth/pipeline-*.yml > /dev/null
yq eval '.' tests/utils/ado/auth/template-auth-pipeline.yml > /dev/null
```

## Adding a new auth pipeline

1. Add a new `pipeline-<name>.yml` next to the others.
2. `extends:` `template-auth-pipeline.yml`.
3. Provide whatever `preAuthSteps` and `extraEnv` configure exactly that
   one auth path. Do **not** modify `auth_smoke` itself.
4. Document any new ADO variables in this README.
5. Ask the ADO admin to register the new pipeline.
