# CI

CI is managed through the [OpenShift CI](https://docs.ci.openshift.org/) system (Prow + ci-operator). The job configuration lives in [openshift/release](https://github.com/openshift/release/tree/master/ci-operator/config/openshift-online/rosa-hyperfleet).

## Jobs

| Job                                                                                                                                                                               | Schedule                 | Description                                                                                                                                      |
| --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------ |
| [`check-docs`](https://prow.ci.openshift.org/job-history/gs/test-platform-results/pr-logs/directory/pull-ci-openshift-online-rosa-hyperfleet-main-check-docs)                     | Pre-submit               | Checks markdown formatting with [Prettier](https://prettier.io/)                                                                                 |
| [`terraform-validate`](https://prow.ci.openshift.org/job-history/gs/test-platform-results/pr-logs/directory/pull-ci-openshift-online-rosa-hyperfleet-main-terraform-validate)     | Pre-submit               | Runs `terraform validate` on all root modules                                                                                                    |
| [`helm-lint`](https://prow.ci.openshift.org/job-history/gs/test-platform-results/pr-logs/directory/pull-ci-openshift-online-rosa-hyperfleet-main-helm-lint)                       | Pre-submit               | Lints Helm charts                                                                                                                                |
| [`check-rendered-files`](https://prow.ci.openshift.org/job-history/gs/test-platform-results/pr-logs/directory/pull-ci-openshift-online-rosa-hyperfleet-main-check-rendered-files) | Pre-submit               | Verifies rendered deploy files are up to date                                                                                                    |
| [`on-demand-e2e`](https://prow.ci.openshift.org/job-history/gs/test-platform-results/pr-logs/directory/pull-ci-openshift-online-rosa-hyperfleet-main-on-demand-e2e)               | Pre-submit (manual)      | End-to-end: provisions ephemeral environment using PR rosa-hyperfleet branch, runs tests, tears down. Trigger with `/test on-demand-e2e` on a PR |
| [`nightly-ephemeral`](https://prow.ci.openshift.org/job-history/gs/test-platform-results/logs/periodic-ci-openshift-online-rosa-hyperfleet-main-nightly-ephemeral)                | Daily at 04:00 UTC       | End-to-end: provisions ephemeral environment using `main` rosa-hyperfleet branch, runs tests, tears down                                         |
| [`nightly-integration`](https://prow.ci.openshift.org/job-history/gs/test-platform-results/logs/periodic-ci-openshift-online-rosa-hyperfleet-main-nightly-integration)            | Daily at 04:00 UTC       | Runs e2e tests against a standing integration environment                                                                                        |
| [`nightly-stage`](https://prow.ci.openshift.org/job-history/gs/test-platform-results/logs/periodic-ci-openshift-online-rosa-hyperfleet-main-nightly-stage)                        | Daily at 04:00 UTC       | Runs e2e tests against a standing stage environment                                                                                              |
| `nightly-m6i` (planned)                                                                                                                                                           | Mon/Wed/Fri at 05:00 UTC | Nightly ephemeral with `m6i.large` instance types — validates general-purpose Intel machines                                                     |
| `nightly-c6i` (planned)                                                                                                                                                           | Tue/Thu/Sat at 05:00 UTC | Nightly ephemeral with `c6i.xlarge` instance types — validates compute-optimized Intel machines                                                  |

> **Note:** `nightly-m6i` and `nightly-c6i` are pending periodic job definitions in [openshift/release](https://github.com/openshift/release). Scripts and override files are ready in this repo.

## Load Testing (Planned)

Load testing scripts are implemented but not yet wired into the Prow workflow. The `rosa-hyperfleet-load-test` step needs to be added to the `rosa-hyperfleet-ephemeral-e2e` workflow in [openshift/release](https://github.com/openshift/release).

- **Entrypoint**: `ci/nightly-load-test.sh` (will run as a Prow step after e2e, before teardown)
- **Scripts**: `ci/load-test/scripts/platform-api-load.js` (API throughput), `ci/load-test/scripts/hcp-lifecycle-load.js` (concurrent HCP creation)
- **Results**: JSON summaries saved to `${ARTIFACT_DIR}/load-test-results/` (visible in Prow artifacts)
- **Baseline comparison**: `ci/load-test/compare-baseline.py` checks for regressions against an S3-stored baseline

### Machine-Type Overrides

The `ci/nightly-machine-type.sh` script provisions ephemeral environments with non-default EC2 instance types. Override files in `ci/nightly-overrides/machine-types/` are injected via `--provision-override-file`:

```bash
# Run locally with a specific machine type
MACHINE_TYPE_OVERRIDE=m6i-large.yaml ./ci/nightly-machine-type.sh
```

To add a new machine type, create a YAML file in `ci/nightly-overrides/machine-types/` with `regional_cluster.node_instance_types` and `management_cluster_defaults.node_instance_types` overrides.

## Cross-Component E2E Testing

Component repos (e.g., `rosa-hyperfleet-api`) can run the e2e test suite against an ephemeral environment with their PR-built image deployed. See [Enabling Pre-Merge E2E for Component Repos](../docs/adding-component-pre-merge.md) for the full workflow, architecture, and SOP for onboarding new repos.

## Client Testing (Planned)

The e2e test runner (`ci/e2e-tests.sh`) validates cluster lifecycle through two CLI clients. Infrastructure provisioning via Terraform is validated separately in pre-submit checks.

### ROSACTL CLI (`make test-e2e-cli`)

Tests the HyperFleet-native CLI (`rosactl`) built from `rosa-hyperfleet-cli`. The CLI binary is cloned and built from `CLI_REPO`/`CLI_REF` before the test run. Runtime: ~30-35 min.

Tests are ordered by Ginkgo labels and executed in this sequence:

| Phase     | Label            | What it does                                                                       |
| --------- | ---------------- | ---------------------------------------------------------------------------------- |
| `setup`   | `help`           | Verifies `rosactl --help` output                                                   |
| `setup`   | `login`          | Logs in to the Platform API (`rosactl login --url`)                                |
| `setup`   | `vpc-create`     | Creates cluster VPC via CloudFormation (VPC, subnets, NAT/IGW, security groups)    |
| `setup`   | `vpc-list`       | Lists VPCs and verifies the created one appears                                    |
| `setup`   | `iam-create`     | Creates operator IAM roles via CloudFormation (8 operator roles + worker role)      |
| `setup`   | `iam-list`       | Lists IAM stacks and verifies the created one appears                              |
| `setup`   | `account-add`    | Registers the customer AWS account with the Platform API                           |
| `create`  | `hcp-create`     | Creates an HCP cluster via the Platform API                                        |
| `setup`   | `oidc-create`    | Creates a managed OIDC config                                                      |
| `monitor` | `cluster-status` | Polls `/clusters/{id}` until `status.phase` is non-empty, then waits for `Ready`   |
| `monitor` | `kubeconfig`     | Generates a kubeconfig and verifies API server connectivity                        |
| `monitor` | `nodepool-create`| Creates an additional nodepool via CLI                                              |
| `monitor` | `nodepool-list`  | Lists nodepools and verifies both default and created pools appear                 |
| `monitor` | `dns-verify`     | Validates DNS resolution and TLS certificate for the KAS endpoint                  |
| `monitor` | `nodepools-wait` | Waits for all nodepools to report ready                                            |
| `monitor` | `hcp-metrics`    | Queries Thanos for `hcp:hostedcluster_available` metric                            |
| `cleanup` | `nodepool-delete`| Deletes the extra nodepool                                                         |
| `cleanup` | `hcp-delete`     | Deletes the HCP cluster                                                            |
| `cleanup` | `cluster-query`  | Polls until the cluster is fully deleted                                           |
| `cleanup` | `oidc-delete`    | Deletes the OIDC config                                                            |
| `cleanup` | `vpc-delete`     | Deletes the VPC CloudFormation stack                                               |
| `cleanup` | `iam-delete`     | Deletes the IAM CloudFormation stack                                               |

`DeferCleanup` handlers fire on failure to prevent resource leaks. A `PRE_CLEANUP_HOOK` collects environment logs to S3 before HCP deletion so the HCP namespace is captured in diagnostics.

### ROSA CLI (`make test-e2e-rosa-cli`)

To enable ROSA CLI tests in CI, set `E2E_SKIP_ROSA_CLI=false` in the job's environment variables.
Tests the upstream `rosa` CLI's HyperFleet integration by cloning `openshift/rosa` at the `hyperfleet-v2` branch. Gated by `ROSA_LABEL_FILTER` (e.g. `hyperfleet-sanity`) and `ROSA_TEST_PROFILE`. Uses `credential_process` auto-refresh to avoid STS TTL expiry during long runs. Runtime: ~25-30 min.

The `hyperfleet-sanity` test exercises the following sequence:

1. **Login and verification** — `rosa login --hyperfleet-url`, `rosa whoami --output json` (verifies V2 API URL and region)
2. **VPC and networking** — Creates VPC, public/private subnets, Internet Gateway, NAT Gateway, route tables, worker security group, and a private hosted zone for PrivateLink DNS
3. **OIDC config** — `rosa create oidc-config --managed` and `rosa list oidc-config`; verifies the IAM OIDC provider exists in the customer account
4. **Operator roles** — `rosa create operator-roles --hosted-cp` creates 8 roles (ingress, cloud-controller-manager, ebs-csi, image-registry, network-config, control-plane-operator, node-pool-management, worker) with federated trust policies
5. **Cluster creation** — `rosa create cluster` with subnet and OIDC config binding; `rosa describe cluster --output json` to fetch cluster ID
6. **Readiness polling** — Waits for cluster state to become `Ready` via the hyperfleet API client (not CLI polling)
7. **Cleanup** — Deletes cluster (`rosa delete cluster`), operator roles (`rosa delete operator-roles`), OIDC config (`rosa delete oidc-config`), hosted zone records, security groups, route tables, NAT Gateway, EIP, Internet Gateway, subnets, and VPC

### Terraform Tests

TODO

### Parallel Execution

The ROSACTL CLI and ROSA CLI suites can run in parallel since they create clusters with independent names and do not share state.

On trying to execute the 2 tests to run in parallel there were issues when running in background via the shell script
   - The `cluster-status` test can fail with an empty `status.phase` after 30 seconds when the hyperfleet-operator's adapter reconciliation never starts.

## Possible solutions

In order to run tests in parallel using prow, the ci-operator config in [openshift/release](https://github.com/openshift/release)

1. The file `ci-operator/config/openshift-online/rosa-hyperfleet-api/openshift-online-rosa-hyperfleet-api-main.yaml` to be updated with 2 jobs running simultaneously triggered by a cron job on prow.

3. Create 2 separate scripts and then maybe in the shell script (ci-operator/step-registry/rosa-hyperfleet/e2e/rosa-hyperfleet-e2e-commands.sh) in the release repo could run the shell scripts in background. The only other thing is to make sure if the logs are captured correctly.

## Build Image

The CI image is built from [ci/Containerfile](ci/Containerfile) and includes all required tools:

| Tool      | Purpose                                       |
| --------- | --------------------------------------------- |
| Terraform | Infrastructure provisioning                   |
| Helm      | Kubernetes chart templating and linting       |
| AWS CLI   | AWS account and resource management           |
| Python/uv | Ephemeral provider and scripting              |
| Prettier  | Markdown formatting checks (`check-docs` job) |
| yq        | YAML processing                               |
| promtool  | Prometheus rule validation and unit testing   |

These tools are available in all CI job containers and can be used in scripts run by CI jobs.

## Ephemeral Environment

The [ci/ephemeral-provider/main.py](ci/ephemeral-provider/main.py) script manages ephemeral environments for CI testing. It supports three modes — provision, teardown (`--teardown`), and resync (`--resync`) — designed to run as separate CI steps with tests in between.

1. Creates a CI-owned git branch from the source repo/branch
2. Bootstraps the pipeline-provisioner pointing at the CI branch
3. Pushes rendered deploy files to trigger pipelines via GitOps
4. Waits for RC/MC pipelines to provision infrastructure
5. (Separate CI step) Runs the testing suite against the provisioned environment
6. Tears down infrastructure via GitOps (`delete: true` in config.yaml)
7. Destroys the pipeline-provisioner
8. CI branch is retained for post-run troubleshooting (delete manually via `git push ci --delete <branch>`)

### Running locally

See [Provisioning a Development Environment](../docs/development-environment.md) for the full guide on running ephemeral environments from your local machine via Make targets.

### Triggering the E2E Job Manually

1. Obtain an API token by visiting <https://oauth-openshift.apps.ci.l2s4.p1.openshiftapps.com/oauth/token/request>
2. Log in with `oc login`
3. Start the job:

```bash
# Trigger nightly-ephemeral
curl -X POST \
    -H "Authorization: Bearer $(oc whoami -t)" \
    'https://gangway-ci.apps.ci.l2s4.p1.openshiftapps.com/v1/executions/' \
    -d '{"job_name": "periodic-ci-openshift-online-rosa-hyperfleet-main-nightly-ephemeral", "job_execution_type": "1"}'

# Trigger nightly-integration
curl -X POST \
    -H "Authorization: Bearer $(oc whoami -t)" \
    'https://gangway-ci.apps.ci.l2s4.p1.openshiftapps.com/v1/executions/' \
    -d '{"job_name": "periodic-ci-openshift-online-rosa-hyperfleet-main-nightly-integration", "job_execution_type": "1"}'

# Trigger nightly-stage
curl -X POST \
    -H "Authorization: Bearer $(oc whoami -t)" \
    'https://gangway-ci.apps.ci.l2s4.p1.openshiftapps.com/v1/executions/' \
    -d '{"job_name": "periodic-ci-openshift-online-rosa-hyperfleet-main-nightly-stage", "job_execution_type": "1"}'
```

4. Copy the `id` from the response and check the execution to get the Prow URL:

```bash
curl -X GET \
    -H "Authorization: Bearer $(oc whoami -t)" \
    'https://gangway-ci.apps.ci.l2s4.p1.openshiftapps.com/v1/executions/<id>'
```

Open the `job_url` from the response to watch the job in Prow.

## Accessing Live Job Logs

When a Prow job is running (e.g. `on-demand-e2e`), you can watch its logs in real time:

1. Open the Prow job page (e.g. from the PR status check link or the job history -- see jobs table above).

2. In the build log output, look for a line like:
   ```
   INFO[2026-03-10T11:41:49Z] Using namespace https://console.xxxxx.ci.openshift.org/k8s/cluster/projects/ci-op-XXXXXXXX
   ```
3. Click the namespace link to open the OpenShift console for the CI cluster where the job pods are running. From there you can inspect pod logs, events, and resources in real time.

> **Note:** Access to the namespace is restricted to the person who triggered the job (i.e. the PR author for pre-submit jobs). There is no configuration option to grant access to additional users.

## AWS Profiles

All CI scripts and the ephemeral provider use a standard set of named AWS CLI profiles. The profile names are the same across CI and local development so that downstream consumers (Terraform, boto3, e2e tests) work identically in both contexts.

| Profile        | Account          | Purpose                              |
| -------------- | ---------------- | ------------------------------------ |
| `rrp-central`  | Central          | Pipeline-provisioner, SSM parameters |
| `rrp-rc`       | Regional Cluster | API Gateway auth, regional infra     |
| `rrp-mc`       | Management       | Management cluster provisioning      |
| `rrp-customer` | Customer         | HCP creation e2e tests               |

Not every job needs every profile. The table below shows which profiles each job type requires:

| Job type              | Required profiles                 |
| --------------------- | --------------------------------- |
| `nightly-ephemeral`   | `rrp-central`, `rrp-rc`, `rrp-mc` |
| `on-demand-e2e`       | `rrp-central`, `rrp-rc`, `rrp-mc` |
| `nightly-integration` | `rrp-rc`, `rrp-customer`          |
| `nightly-stage`       | `rrp-rc`, `rrp-customer`          |

### How profiles are loaded

**CI (Prow):** Each Vault secret contains a pre-built `aws_config` file with the profiles that job type needs. Prow mounts it at `/var/run/rosa-credentials/aws_config`. Scripts source [`ci/setup-aws-profiles.sh`](setup-aws-profiles.sh) which sets `AWS_CONFIG_FILE` to point at this file.

**Local development:** The dev scripts (`scripts/dev/ephemeral-env.sh`, `scripts/dev/int-env.sh`) read account IDs from `rosa-hyperfleet-internal` (or a custom path via `RRP_ACCOUNTS_DEV`/`RRP_ACCOUNTS_INT`) and obtain STS credentials via SAML, writing them to a config file with the same `rrp-*` profile names so containers see an identical interface.

### CI Vault secrets

CI jobs still use Vault-mounted credentials. These are managed in [Vault](https://vault.ci.openshift.org/ui/vault/secrets/kv/kv/list/selfservice/cluster-secrets-rosa-regional-platform-int/).

Vault **path** (where you edit) and **Prow secret name** (what ci-operator mounts) differ. Each Vault entry sets `secretsync/target-name` to the Prow name; jobs reference that name in [openshift/release](https://github.com/openshift/release/tree/master/ci-operator/config/openshift-online/rosa-hyperfleet).

| Prow secret (ci-operator)                  | Vault path                               | Used by                              | Key fields                                                                                                           |
| ------------------------------------------ | ---------------------------------------- | ------------------------------------ | -------------------------------------------------------------------------------------------------------------------- |
| `rosa-regional-platform-ephemeral-creds`   | `rosa-regional-platform-ephemeral-creds` | `nightly-ephemeral`, `on-demand-e2e` | `aws_config` — `rrp-central`, `rrp-rc`, `rrp-mc` profiles; API/ZOA URLs come from terraform outputs during provision |
| `rosa-regional-platform-integration-creds` | `integration-creds`                      | `nightly-integration`                | `aws_config`, `api_url`, `rhobs_api_url`, `zoa_rc_api_url`, `zoa_mc_api_url`                                         |
| `rosa-hyperfleet-stage-creds`              | `stage-creds`                            | `nightly-stage`                      | same keys as integration                                                                                             |

Each `aws_config` field contains a complete AWS CLI config file with only the profiles that job type requires. URL fields (`api_url`, `rhobs_api_url`, `zoa_rc_api_url`, `zoa_mc_api_url`) are configured for integration and stage; update them after a redeploy (same process as `rhobs_api_url` in [`docs/sop/rebuild-integration.md`](../docs/sop/rebuild-integration.md)).

## AWS Account Cleanup (Janitor)

The ephemeral tests create AWS resources across multiple accounts. Teardown relies on `terraform destroy`, which can fail and leak resources. To clean up leaked resources, a CloudFormation-based [aws-nuke-cf](https://github.com/openshift-online/aws-nuke-cf) stack is deployed into each AWS account. It runs aws-nuke on a schedule using an in-account IAM role.

See [ci/janitor/README.md](janitor/README.md) for the nuke configuration, preservation rules, and deployment instructions.
