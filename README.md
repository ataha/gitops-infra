# gitops-infra

GitOps repository for deploying and managing **Veeam Kasten K10** and **PostgreSQL** on a DigitalOcean Kubernetes (DOKS) cluster using **ArgoCD**.

Everything in this repo is declarative. Pushing a commit is how you install, configure, upgrade, and uninstall workloads. No `helm install` or `kubectl apply` by hand (except the one-time bootstrap commands documented below).

---

## Repository Structure

```
gitops-infra/
├── apps/
│   ├── sealed-secrets/          # Sealed Secrets controller (cluster infra)
│   │   └── application.yaml
│   ├── kasten/                  # Kasten K10 Helm install
│   │   ├── application.yaml
│   │   └── values.yaml
│   ├── kasten-config/           # Kasten configuration (profile, policies, secrets)
│   │   ├── application.yaml
│   │   ├── sealed-secret.yaml        # S3 credentials (encrypted)
│   │   ├── sealed-dr-secret.yaml     # DR passphrase (encrypted)
│   │   ├── profile.yaml              # k10-backups Location Profile
│   │   └── policy.yaml               # K10 self-DR policy
│   └── postgresql/              # PostgreSQL workload + its backup policy
│       ├── application.yaml          # Two ArgoCD Apps in one file
│       ├── sealed-postgres-secret.yaml  # DB credentials (encrypted)
│       ├── values.yaml               # Bitnami Helm values
│       └── policy-postgresql.yaml    # Kasten backup policy for postgresql namespace
```

---

## Prerequisites

Before running anything in this repo, the following must be in place:

| Requirement | Notes |
|---|---|
| DOKS cluster running | With a CSI driver that supports volume snapshots |
| ArgoCD installed | In the `argocd` namespace |
| `kubectl` configured | Pointing at your DOKS cluster (`doctl kubernetes cluster kubeconfig save <n>`) |
| `kubeseal` CLI installed | For creating SealedSecrets (`brew install kubeseal` on macOS) |
| AWS S3 bucket | Created and accessible, with versioning + SSE-S3 enabled |
| AWS IAM user | With scoped S3 permissions (see IAM Policy section below) |
| VolumeSnapshotClass annotated | See One-Time Cluster Prep section |

### VS Code Extensions (recommended)

- **YAML** (`redhat.vscode-yaml`) — schema validation
- **Kubernetes** (`ms-kubernetes-tools.vscode-kubernetes-tools`) — live cluster view
- **GitLens** (`eamodio.gitlens`) — Git history as audit trail

---

## ArgoCD Applications Overview

There are **five** ArgoCD Applications deployed from this repo, in bootstrap order:

| App name | Manages | Namespace |
|---|---|---|
| `sealed-secrets` | Sealed Secrets controller | `kube-system` |
| `kasten` | Kasten K10 Helm release | `kasten-io` |
| `kasten-config` | S3 profile, DR policy, credentials | `kasten-io` |
| `postgresql-resources` | DB credentials Secret + Kasten backup policy | `postgresql` |
| `postgresql` | Bitnami PostgreSQL Helm release | `postgresql` |

Inside `kasten-config`, resources are applied in sync-wave order:

| Wave | Resource | Purpose |
|---|---|---|
| 10 | `sealed-secret` → `k10-backups-secret` | S3 credentials for Kasten |
| 10 | `sealed-dr-secret` → `k10-dr-secret` | Kasten DR encryption passphrase |
| 20 | `profile.yaml` | Points Kasten at the S3 bucket |
| 50 | `policy.yaml` | Hourly K10 self-DR backup + export |

Inside `postgresql-resources`:

| Wave | Resource | Purpose |
|---|---|---|
| 0 | `sealed-postgres-secret` → `postgres-credentials` | DB passwords |
| 20 | `policy-postgresql.yaml` | Hourly PostgreSQL backup + export |

---

## One-Time Cluster Prep

These commands are run once per cluster before bootstrapping ArgoCD Applications. They are not in Git because they are per-cluster bootstrap steps, not declarative config.

```bash
# 1. Annotate the DOKS snapshot class so Kasten recognises it
kubectl annotate volumesnapshotclass do-block-storage \
  k10.kasten.io/is-snapshot-class=true --overwrite

# 2. Run Kasten preflight check (optional but recommended)
curl https://docs.kasten.io/tools/k10_primer.sh | bash
```

---

## Bootstrap

Run once to register all Applications with ArgoCD. After this, ArgoCD manages everything.

```bash
# 1. Sealed Secrets controller (must be first)
kubectl apply -f apps/sealed-secrets/application.yaml

# 2. Wait for the controller to be ready before creating any SealedSecrets
kubectl -n kube-system rollout status deployment/sealed-secrets

# 3. Kasten K10
kubectl apply -f apps/kasten/application.yaml

# 4. Kasten configuration (profile, DR policy, credentials)
kubectl apply -f apps/kasten-config/application.yaml

# 5. PostgreSQL
kubectl apply -f apps/postgresql/application.yaml
```

From this point on, every change goes through Git. Push a commit, ArgoCD reconciles it.

---

## Adding or Rotating Secrets

Never commit plaintext secrets. All secrets are encrypted with Sealed Secrets before committing.

### Rotate the S3 credentials

```bash
kubectl create secret generic k10-backups-secret \
  --namespace kasten-io \
  --from-literal=aws_access_key_id='AKIA...' \
  --from-literal=aws_secret_access_key='...' \
  --dry-run=client -o yaml \
| kubeseal --format=yaml \
    --controller-namespace=kube-system \
    --controller-name=sealed-secrets \
> apps/kasten-config/sealed-secret.yaml

git add apps/kasten-config/sealed-secret.yaml
git commit -m "Rotate S3 credentials"
git push
```

### Rotate the PostgreSQL password

```bash
kubectl create secret generic postgres-credentials \
  --namespace postgresql \
  --from-literal=postgres-password='...' \
  --from-literal=password='...' \
  --from-literal=replication-password='...' \
  --dry-run=client -o yaml \
| kubeseal --format=yaml \
    --controller-namespace=kube-system \
    --controller-name=sealed-secrets \
> apps/postgresql/sealed-postgres-secret.yaml

git add apps/postgresql/sealed-postgres-secret.yaml
git commit -m "Rotate PostgreSQL credentials"
git push
```

---

## Upgrading Kasten K10

1. Check the current chart version on Artifact Hub (artifacthub.io/packages/helm/kasten/k10)
2. Edit `apps/kasten/application.yaml`, change `targetRevision`
3. Commit and push — ArgoCD reconciles the upgrade

---

## Uninstalling

### Remove PostgreSQL

```bash
kubectl -n argocd delete application postgresql
kubectl -n argocd delete application postgresql-resources
kubectl delete namespace postgresql
```

### Remove Kasten (keep S3 backups intact)

Before uninstalling, save these three things or DR restore becomes impossible:
- DR passphrase (should already be in your password manager)
- Cluster ID (K10 dashboard → Settings → Disaster Recovery)
- S3 bucket name, region, and IAM credentials

```bash
kubectl -n argocd delete application kasten-config
kubectl -n argocd delete application kasten
kubectl -n kasten-io delete pvc --all
kubectl delete namespace kasten-io
```

### Reinstall from DR backup

After reinstalling Kasten via ArgoCD (which recreates the DR passphrase Secret from the SealedSecret in Git), go to the K10 dashboard → Settings → Disaster Recovery, enter the saved Cluster ID and passphrase to discover restore points in S3, then restore the catalog.

---

## IAM Policy

The AWS IAM user for Kasten's S3 access needs only these permissions:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "BucketLevel",
      "Effect": "Allow",
      "Action": ["s3:ListBucket", "s3:GetBucketLocation"],
      "Resource": "arn:aws:s3:::<your-bucket-name>"
    },
    {
      "Sid": "ObjectLevel",
      "Effect": "Allow",
      "Action": [
        "s3:PutObject",
        "s3:GetObject",
        "s3:DeleteObject",
        "s3:AbortMultipartUpload",
        "s3:ListMultipartUploadParts"
      ],
      "Resource": "arn:aws:s3:::<your-bucket-name>/*"
    }
  ]
}
```

---

## Critical Values to Keep Outside This Repo

Store these in a password manager — not in Git:

| Secret | Where to find it | Why it matters |
|---|---|---|
| Kasten DR passphrase | Your password manager | Without it, DR backups are unrecoverable |
| Kasten Cluster ID | K10 dashboard → Settings → DR | Required to discover restore points on a new cluster |
| AWS S3 access key + secret | AWS IAM | Required to read the S3 bucket during restore |
| Sealed Secrets master key | See below | Without it, SealedSecrets in Git cannot be decrypted on a new cluster |

### Back up the Sealed Secrets master key

```bash
kubectl -n kube-system get secret \
  -l sealedsecrets.bitnami.com/sealed-secrets-key \
  -o yaml > sealed-secrets-master-key-backup.yaml
```

Store this file in a password manager or encrypted drive. Do not commit it to Git.

---

## Kasten Dashboard Access

```bash
kubectl --namespace kasten-io port-forward service/gateway 8080:8000

TOKEN_NAME=$(kubectl get secret -n kasten-io | grep k10-k10-token | cut -d " " -f 1)
kubectl get secret -n kasten-io $TOKEN_NAME -o jsonpath="{.data.token}" | base64 -d; echo
```

Open http://127.0.0.1:8080/k10/ and paste the token.

---

## Backup Verification (Fire Drill)

```bash
# 1. Insert test data
kubectl -n postgresql exec -it postgresql-0 -- \
  psql -U postgres -d appdb \
  -c "CREATE TABLE dr_test (id serial, val text); INSERT INTO dr_test (val) VALUES ('pre-backup');"

# 2. Trigger manual backup: K10 dashboard → Policies → postgresql-backup → Run Once
#    Wait for the Action to complete

# 3. Delete the test data
kubectl -n postgresql exec -it postgresql-0 -- \
  psql -U postgres -d appdb -c "DROP TABLE dr_test;"

# 4. Restore: K10 dashboard → Applications → postgresql → Restore
#    Select the restore point from step 2

# 5. Verify
kubectl -n postgresql exec -it postgresql-0 -- \
  psql -U postgres -d appdb -c "SELECT * FROM dr_test;"
```

---

## Ownership

| Component | Managed by | Change process |
|---|---|---|
| Cluster infrastructure | Platform team | PR to `apps/sealed-secrets/` |
| Kasten install + S3 profile + DR policy | Platform team | PR to `apps/kasten/` and `apps/kasten-config/` |
| PostgreSQL workload + its backup policy | App team | PR to `apps/postgresql/` |

All changes go through GitHub PRs. ArgoCD reconciles from `main` branch only.
