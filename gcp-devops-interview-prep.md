# GCP DevOps Interview Prep

Coding questions and answers for the **Custom Application Architect / Google Cloud Engineer (CL9)** role, covering Terraform, Azure DevOps, GCP and multi-cloud integration.

---

## 1. Custom Terraform modules for GCP

### Q1. Write a reusable GCS bucket module with enterprise guardrails

Requirements: CMEK encryption, versioning, uniform bucket-level access, public access prevention and required labels.

**`modules/gcs-bucket/variables.tf`**

```hcl
variable "project_id" {
  type = string
}

variable "name" {
  type = string
}

variable "location" {
  type    = string
  default = "EU"
}

variable "kms_key_name" {
  type = string
}

variable "labels" {
  type = map(string)

  validation {
    condition = alltrue([
      for k in ["env", "owner", "cost-center"] : contains(keys(var.labels), k)
    ])
    error_message = "labels must include env, owner and cost-center."
  }
}

variable "retention_days" {
  type    = number
  default = 0
}

variable "iam_members" {
  description = "Map of role => list of members"
  type        = map(list(string))
  default     = {}
}
```

**`modules/gcs-bucket/main.tf`**

```hcl
resource "google_storage_bucket" "this" {
  project                     = var.project_id
  name                        = var.name
  location                    = var.location
  uniform_bucket_level_access = true
  public_access_prevention    = "enforced"
  force_destroy               = false
  labels                      = var.labels

  versioning {
    enabled = true
  }

  encryption {
    default_kms_key_name = var.kms_key_name
  }

  dynamic "lifecycle_rule" {
    for_each = var.retention_days > 0 ? [1] : []

    content {
      action {
        type = "Delete"
      }
      condition {
        age = var.retention_days
      }
    }
  }
}

resource "google_storage_bucket_iam_binding" "this" {
  for_each = var.iam_members

  bucket  = google_storage_bucket.this.name
  role    = each.key
  members = each.value
}
```

**`modules/gcs-bucket/outputs.tf`**

```hcl
output "name" {
  value = google_storage_bucket.this.name
}

output "url" {
  value = google_storage_bucket.this.url
}
```

**Key talking points**

- `validation` blocks enforce governance at plan time.
- `dynamic` blocks handle optional configuration.
- `for_each` over a map gives stable resource addresses. `count` recreates resources when list order changes.
- The GCS service agent needs `roles/cloudkms.cryptoKeyEncrypterDecrypter` on the KMS key. Grant it in the module or document it as a prerequisite.

---

### Q2. `count` vs `for_each`: refactor this code

```hcl
resource "google_project_service" "apis" {
  count   = length(var.apis)
  project = var.project_id
  service = var.apis[count.index]
}
```

**Answer:** With `count`, each resource is addressed by index (`apis[0]`). Removing an item from the middle of the list shifts every later item, so Terraform destroys and recreates them. With `for_each`, each resource is addressed by key (`apis["compute.googleapis.com"]`), so its address stays stable.

```hcl
resource "google_project_service" "apis" {
  for_each = toset(var.apis)

  project                    = var.project_id
  service                    = each.value
  disable_on_destroy         = false
  disable_dependent_services = false
}
```

To migrate existing state without recreating resources, use a `moved` block (Terraform 1.1+):

```hcl
moved {
  from = google_project_service.apis[0]
  to   = google_project_service.apis["compute.googleapis.com"]
}
```

Or use the CLI:

```bash
terraform state mv \
  'google_project_service.apis[0]' \
  'google_project_service.apis["compute.googleapis.com"]'
```

---

### Q3. Configure remote state with locking and keep environments separate

```hcl
terraform {
  required_version = ">= 1.6"

  required_providers {
    google = {
      source  = "hashicorp/google"
      version = "~> 6.0"
    }
  }

  backend "gcs" {
    bucket = "org-tfstate-prod"
    prefix = "landing-zone/network"
  }
}
```

- The GCS backend supports **native state locking**, so no lock table is needed.
- Turn on versioning for the state bucket so you can recover an earlier state.
- Use CMEK for the bucket, and give write access only to the pipeline service account.

**Per-environment state:** pass the bucket and prefix at init time.

```bash
terraform init \
  -backend-config="bucket=org-tfstate-${ENV}" \
  -backend-config="prefix=app/${COMPONENT}"
```

I prefer separate directories or tfvars per environment over `terraform workspace` for production. Workspaces share the same backend and code path, which makes it easy to apply to the wrong environment.

**Reading another stack's outputs:**

```hcl
data "terraform_remote_state" "network" {
  backend = "gcs"

  config = {
    bucket = "org-tfstate-prod"
    prefix = "landing-zone/network"
  }
}

# Usage:
# data.terraform_remote_state.network.outputs.subnet_self_link
```

---

### Q4. Landing zone: create a project in a folder, attach it to a Shared VPC and enable APIs

```hcl
resource "random_id" "suffix" {
  byte_length = 2
}

resource "google_project" "this" {
  name            = var.name
  project_id      = "${var.prefix}-${var.name}-${random_id.suffix.hex}"
  folder_id       = var.folder_id
  billing_account = var.billing_account
  labels          = var.labels
  deletion_policy = "PREVENT" # google provider v6
}

resource "google_project_service" "apis" {
  for_each = toset(var.apis)

  project = google_project.this.project_id
  service = each.value
}

resource "google_compute_shared_vpc_service_project" "attach" {
  count = var.shared_vpc_host_project != null ? 1 : 0

  host_project    = var.shared_vpc_host_project
  service_project = google_project.this.project_id

  depends_on = [google_project_service.apis]
}

resource "google_compute_subnetwork_iam_member" "net_user" {
  for_each = toset(var.subnet_users)

  project    = var.shared_vpc_host_project
  region     = var.region
  subnetwork = var.subnet_name
  role       = "roles/compute.networkUser"
  member     = each.value
}
```

**Follow-up: org policy as code**

```hcl
resource "google_org_policy_policy" "no_external_ip" {
  name   = "folders/${var.folder_id}/policies/compute.vmExternalIpAccess"
  parent = "folders/${var.folder_id}"

  spec {
    rules {
      deny_all = "TRUE"
    }
  }
}
```

---

### Q5. `google_project_iam_policy` vs `_binding` vs `_member`: which is safest?

| Resource | Behavior | Risk |
|---|---|---|
| `_policy` | Authoritative for the **entire** project IAM policy | Can lock everyone out, including the pipeline SA |
| `_binding` | Authoritative for **one role** | Removes other members of that role |
| `_member` | Non-authoritative, adds one member | Safest, but can drift |

Use `_member` for shared projects. Use `_binding` only when the module fully owns the role. Never mix `_binding` and `_member` for the same role, because each apply removes the other's changes.

---

## 2. Azure DevOps pipelines for Terraform on GCP

### Q6. Multi-stage pipeline: validate, plan, approve, apply with keyless authentication

**`azure-pipelines.yml`**

```yaml
trigger:
  branches:
    include: [main]
  paths:
    include: [infra/**]

pr:
  branches:
    include: [main]

variables:
  TF_VERSION: "1.9.5"
  WORKDIR: "infra/envs/$(ENV)"
  GCP_PROJECT_NUMBER: "123456789012"
  WIF_POOL: "ado-pool"
  WIF_PROVIDER: "ado-provider"
  GCP_SA: "tf-deployer@seed-project.iam.gserviceaccount.com"

stages:
  - stage: Validate
    jobs:
      - job: lint
        pool:
          vmImage: ubuntu-latest
        steps:
          - template: templates/install-terraform.yml
            parameters:
              version: $(TF_VERSION)

          - script: |
              terraform fmt -check -recursive
              cd $(WORKDIR)
              terraform init -backend=false
              terraform validate
            displayName: fmt + validate

          - script: |
              pip install checkov
              checkov -d infra --framework terraform --quiet --soft-fail-on LOW
            displayName: Security scan (Checkov)

  - stage: Plan
    dependsOn: Validate
    variables:
      ENV: prod
    jobs:
      - job: plan
        pool:
          vmImage: ubuntu-latest
        steps:
          - template: templates/gcp-auth.yml

          - script: |
              cd $(WORKDIR)
              terraform init -input=false
              terraform plan -input=false -out=tfplan -detailed-exitcode || ec=$?
              if [ "${ec:-0}" = "2" ]; then changes=true; else changes=false; fi
              echo "##vso[task.setvariable variable=HAS_CHANGES;isOutput=true]$changes"
              [ "${ec:-0}" = "1" ] && exit 1 || exit 0
            name: tfplan
            displayName: terraform plan

          - publish: $(WORKDIR)/tfplan
            artifact: tfplan-prod

  - stage: Apply
    dependsOn: Plan
    condition: >-
      and(
        succeeded(),
        eq(variables['Build.SourceBranch'], 'refs/heads/main'),
        eq(dependencies.Plan.outputs['plan.tfplan.HAS_CHANGES'], 'true')
      )
    variables:
      ENV: prod
    jobs:
      - deployment: apply
        environment: gcp-prod # approvals & checks configured on the environment
        pool:
          vmImage: ubuntu-latest
        strategy:
          runOnce:
            deploy:
              steps:
                - checkout: self

                - download: current
                  artifact: tfplan-prod

                - template: templates/gcp-auth.yml

                - script: |
                    cd $(WORKDIR)
                    terraform init -input=false
                    terraform apply -input=false $(Pipeline.Workspace)/tfplan-prod/tfplan
                  displayName: terraform apply
```

**`templates/gcp-auth.yml`**: keyless, no JSON key stored in ADO

```yaml
steps:
  - task: AzureCLI@2 # ADO service connection with workload identity federation
    displayName: Get OIDC token and configure GCP WIF
    inputs:
      azureSubscription: "ado-oidc-connection"
      scriptType: bash
      addSpnToEnvironment: true
      scriptLocation: inlineScript
      inlineScript: |
        echo "$idToken" > $(Agent.TempDirectory)/oidc.jwt

        cat > $(Agent.TempDirectory)/gcp-cred.json <<EOF
        {
          "type": "external_account",
          "audience": "//iam.googleapis.com/projects/$(GCP_PROJECT_NUMBER)/locations/global/workloadIdentityPools/$(WIF_POOL)/providers/$(WIF_PROVIDER)",
          "subject_token_type": "urn:ietf:params:oauth:token-type:jwt",
          "token_url": "https://sts.googleapis.com/v1/token",
          "credential_source": {
            "file": "$(Agent.TempDirectory)/oidc.jwt"
          },
          "service_account_impersonation_url": "https://iamcredentials.googleapis.com/v1/projects/-/serviceAccounts/$(GCP_SA):generateAccessToken"
        }
        EOF

        echo "##vso[task.setvariable variable=GOOGLE_APPLICATION_CREDENTIALS]$(Agent.TempDirectory)/gcp-cred.json"
```

**Why this design**

- **Plan artifact → apply:** the apply stage runs exactly the plan that was reviewed.
- **`-detailed-exitcode`:** the apply stage is skipped when there are no changes.
- **Environment approvals:** production requires manual approval.
- **Keyless authentication:** no long-lived service account keys to store or rotate.

**GCP side of the federation (Terraform)**

```hcl
resource "google_iam_workload_identity_pool" "ado" {
  workload_identity_pool_id = "ado-pool"
}

resource "google_iam_workload_identity_pool_provider" "ado" {
  workload_identity_pool_id          = google_iam_workload_identity_pool.ado.workload_identity_pool_id
  workload_identity_pool_provider_id = "ado-provider"

  attribute_mapping = {
    "google.subject" = "assertion.sub"
  }

  attribute_condition = "assertion.sub.startsWith('sc://myorg/myproject/')"

  oidc {
    issuer_uri        = "https://vstoken.dev.azure.com/<ORG_ID>"
    allowed_audiences = ["api://AzureADTokenExchange"]
  }
}

resource "google_service_account_iam_member" "wif" {
  service_account_id = google_service_account.tf.name
  role               = "roles/iam.workloadIdentityUser"
  member             = "principalSet://iam.googleapis.com/${google_iam_workload_identity_pool.ado.name}/*"
}
```

> Check the exact issuer and audience against your ADO organization's service connection details.

---

### Q7. Keep the pipeline DRY across dev, test and prod

Use a **stage template with parameters**.

**`templates/tf-stage.yml`**

```yaml
parameters:
  - name: env
    type: string
  - name: serviceConnection
    type: string
  - name: requireApproval
    type: boolean
    default: false

stages:
  - stage: Deploy_${{ parameters.env }}
    jobs:
      - deployment: tf
        ${{ if parameters.requireApproval }}:
          environment: gcp-${{ parameters.env }}-gated
        ${{ else }}:
          environment: gcp-${{ parameters.env }}
        pool:
          vmImage: ubuntu-latest
        strategy:
          runOnce:
            deploy:
              steps:
                - checkout: self
                - script: echo "Deploying to ${{ parameters.env }}"
```

**`azure-pipelines.yml`**

```yaml
stages:
  - template: templates/tf-stage.yml
    parameters:
      env: dev
      serviceConnection: gcp-dev

  - template: templates/tf-stage.yml
    parameters:
      env: prod
      serviceConnection: gcp-prod
      requireApproval: true
```

- Store templates in a separate repo and pin them with `resources.repositories` using `ref: refs/tags/v1.2.0`.
- Use variable groups linked to Azure Key Vault for non-GCP secrets.

---

### Q8. Version and consume Terraform modules across teams

```hcl
module "bucket" {
  source = "git::https://dev.azure.com/org/platform/_git/tf-modules//gcs-bucket?ref=v2.3.0"

  project_id   = var.project_id
  name         = "app-data-prod"
  kms_key_name = var.kms_key_name
  labels       = local.labels
}
```

- Use semantic version tags and follow the module layout: `README`, `examples/`, `versions.tf`.
- Generate docs with `terraform-docs`.
- Test with `terraform test` (1.6+) or Terratest.
- Publish a CHANGELOG. A major version bump signals a breaking change.

Authenticate to the ADO Git repo inside a pipeline:

```bash
git config --global \
  url."https://$(System.AccessToken)@dev.azure.com".insteadOf "https://dev.azure.com"
```

---

### Q9. Write a native `terraform test` for the bucket module

**`modules/gcs-bucket/tests/bucket.tftest.hcl`**

```hcl
variables {
  project_id   = "test-proj"
  name         = "unit-test-bucket"
  kms_key_name = "projects/p/locations/eu/keyRings/r/cryptoKeys/k"
  labels = {
    env         = "dev"
    owner       = "team"
    cost-center = "123"
  }
}

run "enforces_public_access_prevention" {
  command = plan

  assert {
    condition     = google_storage_bucket.this.public_access_prevention == "enforced"
    error_message = "Bucket must block public access"
  }
}

run "rejects_missing_labels" {
  command = plan

  variables {
    labels = {
      env = "dev"
    }
  }

  expect_failures = [var.labels]
}
```

---

## 3. Multi-cloud integration (Azure and AWS)

### Q10. HA VPN between GCP and AWS (GCP side)

```hcl
resource "google_compute_ha_vpn_gateway" "gw" {
  name    = "gcp-to-aws"
  region  = var.region
  network = var.network_id
}

resource "google_compute_router" "cr" {
  name    = "cr-aws"
  region  = var.region
  network = var.network_id

  bgp {
    asn = 64514
  }
}

resource "google_compute_external_vpn_gateway" "aws" {
  name            = "aws-peer"
  redundancy_type = "FOUR_IPS_REDUNDANCY"

  # 4 outside IPs from 2 AWS VPN connections
  dynamic "interface" {
    for_each = var.aws_tunnel_ips

    content {
      id         = interface.key
      ip_address = interface.value
    }
  }
}

resource "google_compute_vpn_tunnel" "t" {
  for_each = { for i, ip in var.aws_tunnel_ips : i => ip }

  name                            = "tunnel-${each.key}"
  region                          = var.region
  vpn_gateway                     = google_compute_ha_vpn_gateway.gw.id
  vpn_gateway_interface           = each.key < 2 ? 0 : 1
  peer_external_gateway           = google_compute_external_vpn_gateway.aws.id
  peer_external_gateway_interface = each.key
  shared_secret                   = var.psks[each.key] # sourced from Secret Manager
  router                          = google_compute_router.cr.id
  ike_version                     = 2
}

# Add google_compute_router_interface and google_compute_router_peer
# per tunnel (BGP link-local 169.254.x.x/30).
```

- Use HA VPN for 99.99% availability.
- Use Cloud Interconnect or Cross-Cloud Interconnect for high bandwidth.
- Plan non-overlapping CIDRs up front and advertise routes over BGP.
- Keep the pre-shared keys in Secret Manager, not in tfvars.

---

### Q11. Cloud Run calls AWS S3 and Azure without stored keys

**AWS side:** a GCP service account produces an OIDC ID token, and the workload exchanges it with AWS STS `AssumeRoleWithWebIdentity`.

```python
import boto3
import google.auth.transport.requests
from google.oauth2 import id_token


def aws_session(role_arn: str, audience: str = "sts.amazonaws.com") -> boto3.Session:
    """Exchange a Google-signed ID token for temporary AWS credentials."""
    request = google.auth.transport.requests.Request()
    token = id_token.fetch_id_token(request, audience)

    creds = boto3.client("sts").assume_role_with_web_identity(
        RoleArn=role_arn,
        RoleSessionName="gcp-cloudrun",
        WebIdentityToken=token,
    )["Credentials"]

    return boto3.Session(
        aws_access_key_id=creds["AccessKeyId"],
        aws_secret_access_key=creds["SecretAccessKey"],
        aws_session_token=creds["SessionToken"],
    )


s3 = aws_session("arn:aws:iam::111122223333:role/gcp-reader").client("s3")
```

The AWS role's trust policy:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": { "Federated": "accounts.google.com" },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "accounts.google.com:sub": "<GCP_SA_UNIQUE_ID>"
        }
      }
    }
  ]
}
```

**Azure side:** use Entra ID workload identity federation. Add a federated credential to an app registration with issuer `https://accounts.google.com` and the service account's unique ID as the subject. The workload then uses `ClientAssertionCredential` with the Google ID token.

---

### Q12. Trigger an Azure Data Factory pipeline when a file lands in GCS

```python
# Cloud Run function triggered by Eventarc (GCS object.finalized)
import functions_framework
import google.auth.transport.requests
import requests
from google.oauth2 import id_token

TENANT = "tenant-guid"
CLIENT = "app-client-id"
ADF_URL = (
    "https://management.azure.com/subscriptions/{sub}/resourceGroups/{rg}"
    "/providers/Microsoft.DataFactory/factories/{df}/pipelines/{p}"
    "/createRun?api-version=2018-06-01"
)


def azure_token() -> str:
    """Exchange a Google ID token for an Azure management-plane token."""
    request = google.auth.transport.requests.Request()
    gtoken = id_token.fetch_id_token(request, "api://AzureADTokenExchange")

    resp = requests.post(
        f"https://login.microsoftonline.com/{TENANT}/oauth2/v2.0/token",
        data={
            "client_id": CLIENT,
            "grant_type": "client_credentials",
            "client_assertion_type": "urn:ietf:params:oauth:client-assertion-type:jwt-bearer",
            "client_assertion": gtoken,
            "scope": "https://management.azure.com/.default",
        },
        timeout=10,
    )
    resp.raise_for_status()
    return resp.json()["access_token"]


@functions_framework.cloud_event
def on_file(event):
    data = event.data
    resp = requests.post(
        ADF_URL.format(sub="...", rg="...", df="...", p="ingest"),
        headers={"Authorization": f"Bearer {azure_token()}"},
        json={"bucket": data["bucket"], "object": data["name"]},
        timeout=30,
    )
    resp.raise_for_status()
    print("ADF runId:", resp.json()["runId"])
```

---

## 4. Enterprise AI integration

### Q13. A GCP app calls an AI model on Azure OpenAI (or AWS Bedrock)

**Design**

- Cloud Run with a dedicated service account.
- Egress through Direct VPC egress, then VPN/Interconnect to an Azure Private Endpoint.
- Authenticate with federated identity (see Q11), or keep an API key in Secret Manager as a fallback.
- Retries with backoff, a timeout and a circuit breaker.
- Log prompt metadata, never PII.

```python
import os
import time

import requests
from google.cloud import secretmanager

RETRYABLE = {429, 500, 502, 503}


def get_secret(name: str) -> str:
    client = secretmanager.SecretManagerServiceClient()
    return client.access_secret_version(name=name).payload.data.decode()


ENDPOINT = os.environ["AOAI_ENDPOINT"]  # private endpoint DNS
DEPLOYMENT = os.environ["AOAI_DEPLOYMENT"]
API_KEY = get_secret(os.environ["AOAI_KEY_SECRET"])


def chat(prompt: str, retries: int = 4) -> str:
    url = (
        f"{ENDPOINT}/openai/deployments/{DEPLOYMENT}"
        "/chat/completions?api-version=2024-06-01"
    )
    for attempt in range(retries):
        resp = requests.post(
            url,
            headers={"api-key": API_KEY},
            json={"messages": [{"role": "user", "content": prompt}]},
            timeout=30,
        )
        if resp.status_code in RETRYABLE:
            time.sleep(min(2**attempt, 20))  # exponential backoff
            continue
        resp.raise_for_status()
        return resp.json()["choices"][0]["message"]["content"]

    raise RuntimeError("AI endpoint unavailable after retries")
```

**Terraform for the Cloud Run service**

```hcl
resource "google_cloud_run_v2_service" "ai_app" {
  name     = "ai-gateway"
  location = var.region
  ingress  = "INGRESS_TRAFFIC_INTERNAL_LOAD_BALANCER"

  template {
    service_account = google_service_account.ai_app.email

    vpc_access {
      egress = "ALL_TRAFFIC"

      network_interfaces {
        network    = var.network
        subnetwork = var.subnet
      }
    }

    containers {
      image = var.image

      env {
        name  = "AOAI_KEY_SECRET"
        value = google_secret_manager_secret_version.aoai.name
      }
    }
  }
}
```

---

## 5. Python automation and SDLC

### Q14. Find resources missing required labels across a folder (CI governance check)

```python
import sys

from google.cloud import asset_v1

REQUIRED = {"env", "owner", "cost-center"}
ASSET_TYPES = [
    "storage.googleapis.com/Bucket",
    "compute.googleapis.com/Instance",
    "run.googleapis.com/Service",
]


def find_unlabeled(scope: str):
    """Yield (resource_name, missing_labels) for non-compliant resources."""
    client = asset_v1.AssetServiceClient()
    results = client.search_all_resources(
        request={"scope": scope, "asset_types": ASSET_TYPES}  # scope: "folders/123456"
    )
    for resource in results:
        missing = REQUIRED - set(resource.labels.keys())
        if missing:
            yield resource.name, sorted(missing)


if __name__ == "__main__":
    bad = list(find_unlabeled(sys.argv[1]))
    for name, missing in bad:
        print(f"{name}: missing {missing}")
    sys.exit(1 if bad else 0)  # non-zero exit fails the pipeline
```

---

### Q15. Detect drift nightly and alert

```yaml
schedules:
  - cron: "0 2 * * *"
    displayName: Nightly drift check
    branches:
      include: [main]
    always: true

trigger: none

jobs:
  - job: drift
    pool:
      vmImage: ubuntu-latest
    strategy:
      matrix:
        network:
          COMPONENT: network
        iam:
          COMPONENT: iam
    steps:
      - template: templates/gcp-auth.yml

      - script: |
          cd infra/envs/prod/$(COMPONENT)
          terraform init -input=false
          terraform plan -input=false -detailed-exitcode -lock=false
          ec=$?
          if [ $ec -eq 2 ]; then
            echo "##vso[task.logissue type=warning]Drift detected in $(COMPONENT)"
            exit 1
          fi
          exit $ec
        displayName: Detect drift
```

---

## 6. Rapid-fire scenarios

| Question | Strong answer |
|---|---|
| State is locked after a crashed pipeline | Confirm no run is active, then `terraform force-unlock <LOCK_ID>`. Find out why the lock wasn't released. |
| Import a manually created bucket | Terraform 1.5+ `import` block, then `terraform plan -generate-config-out=gen.tf` (see below). |
| A secret shows in plan output | Mark it `sensitive = true`, prefer `ephemeral` values (TF 1.10+). State still holds secrets, so encrypt it and restrict access. |
| Circular dependency between modules | Break it with an intermediate output, a separate layer, or `terraform_remote_state`. |
| Prevent deleting a production DB | `lifecycle { prevent_destroy = true }` plus `deletion_protection = true`. |
| Enforce policy before apply | Checkov/tfsec in CI, OPA/Conftest on `terraform show -json tfplan`, and Org Policies at runtime. |
| Least privilege for the pipeline SA | One SA per environment and layer, no Owner role, impersonation via WIF, IAM Recommender to trim roles. |
| Deploy containers to GKE/Cloud Run from ADO | Build → push to Artifact Registry → Binary Authorization attestation → deploy (Cloud Deploy or `gcloud run deploy`), canary via traffic splitting. |
| Git workflow for IaC | Trunk-based, plan output posted as a PR comment, CODEOWNERS reviewers, branch policies, build validation pipeline. |

**Import block example**

```hcl
import {
  to = google_storage_bucket.legacy
  id = "my-project/legacy-bucket-name"
}
```

```bash
terraform plan -generate-config-out=generated.tf
```

**Policy check on a plan with Conftest**

```bash
terraform plan -out=tfplan
terraform show -json tfplan > tfplan.json
conftest test tfplan.json --policy policy/
```

---

## Suggested prep focus

1. **Terraform modules (~35%):** `for_each`, `dynamic`, validation, `moved`/`import`, remote state, testing.
2. **Azure DevOps YAML (~30%):** templates, environments and approvals, artifacts, WIF service connections.
3. **Multi-cloud and AI integration (~20%):** HA VPN/Interconnect, identity federation, private endpoints.
4. **Python and SDLC (~15%):** Asset Inventory and Secret Manager SDKs, CI security scanning.
