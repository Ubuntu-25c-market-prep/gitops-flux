# Velero on dev-eks-us-east-1

[Velero](https://velero.io) backs up the cluster and restores it: a deleted namespace, a
broken upgrade, or the whole cluster rebuilt from scratch. This guide covers installing it
on `dev-eks-us-east-1` with permission to write to an S3 bucket and take EBS snapshots.

Flux watches this repo and applies whatever is in it to the cluster. You don't run
`kubectl apply` or `helm install` yourself: to change the cluster, you change the files
here and push.

## Terms used in this guide

- **Backup** - one run of Velero. Two halves: every Kubernetes object, written to S3 as
  YAML, and an EBS snapshot of every persistent volume. A restore needs both.
- **BackupStorageLocation** - the S3 bucket Velero writes the object half to.
- **VolumeSnapshotLocation** - where Velero takes the EBS snapshots (AWS, `us-east-1`).
- **Schedule** - a cron that creates Backups. Each Backup carries a `ttl`; Velero deletes
  it, and its snapshots, when the ttl runs out.
- **Restore** - takes a Backup and applies it back to the cluster, optionally into a
  different namespace.
- **IRSA** (IAM Roles for Service Accounts) - lets the `velero` service account act as an
  AWS IAM role, so no AWS keys are stored in the cluster.
- **Sealed Secrets** - encrypts a Kubernetes Secret so it's safe to commit to Git. Only the
  controller running in the cluster can decrypt it.

## How it works

1. `terraform-infra-v2` creates the S3 bucket, the IAM policy, and an IAM role that only
   the `velero` service account in the `velero` namespace can use.
2. A SealedSecret in this repo carries the role ARN and the bucket name. Both contain the
   AWS account id, and this repo is public, which is why they are encrypted.
3. Flux installs the Velero Helm chart, with the AWS plugin, and annotates the service
   account with the role ARN.
4. Every day at 03:00 UTC the `daily` Schedule backs up the whole cluster and keeps each
   backup for 30 days.

## The files

| File | What it does |
|---|---|
| `infrastructures/base/000-helm-repository/vmware-tanzu.yaml` | Where Flux downloads the Velero chart from |
| `infrastructures/base/velero/helmrelease.yaml` | The chart version, the AWS plugin and the service account. Shared by every cluster |
| `clusters/dev/dev-eks-us-east-1/velero.yaml` | Tells Flux to apply the folder below, after Sealed Secrets is running |
| `clusters/dev/dev-eks-us-east-1/velero/patch.yaml` | This cluster's snapshot region, Schedule and node placement |
| `clusters/dev/dev-eks-us-east-1/velero/velero-irsa-sealed.yaml` | The encrypted role ARN and bucket |

## Before you start

You need:

- the [`kubeseal` CLI](https://github.com/bitnami-labs/sealed-secrets#kubeseal)
- the [`velero` CLI](https://velero.io/docs/main/basic-install/#install-the-cli)
  (`brew install velero`), to check and drive backups once it's running
- the `aws` CLI, logged in to the dev account
- permission to apply changes in the `terraform-infra-v2` repo

## Step 1: create the AWS pieces

In `terraform-infra-v2`, as three pull requests, merged in this order. CI applies one
stack per merge, named by the `Path:` in the commit message:

| Order | Stack | Change |
|---|---|---|
| 1 | `resources/us-east-1/dev/s3` | a `velero` entry under `buckets` in `config.yaml` |
| 2 | `resources/us-east-1/dev/iam` | `policies/dev-VeleroPolicy-us-east-1.yaml` |
| 3 | `resources/us-east-1/dev/eks` | `dev-irsa-velero-us-east-1` under `service_accounts` in `iam.yaml`, `namespace_service_account: velero/velero` |

The order matters: the policy names the bucket, and the role attaches the policy.

## Step 2: create the real SealedSecret

```
kubeseal --context dev-eks-us-east-1 --fetch-cert \
  --controller-namespace=sealed-secrets \
  --controller-name=sealed-secrets-sealed-secrets > pub-cert.pem
```

```
cat <<YAML | kubeseal --format=yaml --cert=pub-cert.pem \
  > clusters/dev/dev-eks-us-east-1/velero/velero-irsa-sealed.yaml
apiVersion: v1
kind: Secret
metadata:
  name: velero-irsa
  namespace: flux-system
stringData:
  values.yaml: |
    serviceAccount:
      server:
        annotations:
          eks.amazonaws.com/role-arn: arn:aws:iam::<ACCOUNT_ID>:role/dev-irsa-velero-us-east-1
    configuration:
      backupStorageLocation:
        - name: default
          provider: aws
          bucket: dev-s3-us-east-1-velero-<ACCOUNT_ID>
          prefix: dev-eks-us-east-1
          default: true
          config:
            region: us-east-1
YAML
```

The annotation must land at `serviceAccount.server.annotations`. Anywhere else, Helm
accepts it and ignores it, and Velero fails with AccessDenied at the first backup.
`backupStorageLocation` is a list, and Helm replaces lists whole, so the full entry goes
in the secret, not just the bucket.

## Step 3: commit and push

Open a pull request. Once it merges, Flux picks it up on its next sync.

## Step 4: check that it worked

```
flux get helmreleases -n flux-system velero
kubectl -n velero get pods
velero backup-location get          # PHASE must be Available
velero schedule get                 # daily, 0 3 * * *
```

Then prove a restore works. Restore into a copy, never over the original:

```
velero backup create test-1 --include-namespaces monitoring --wait
velero backup describe test-1 --details   # lists the objects and EBS snapshot ids
velero restore create --from-backup test-1 \
  --namespace-mappings monitoring:monitoring-restore --wait
kubectl -n monitoring-restore get pvc     # new volumes, created from the snapshots
kubectl delete namespace monitoring-restore
velero backup delete test-1 --confirm
```

A backup you have never restored is not a backup.

## Troubleshooting

**`velero backup-location get` shows `Unavailable`.** Velero can't reach the bucket. Run
`kubectl -n velero logs deploy/velero` and look for AccessDenied. Check that the service
account has the `eks.amazonaws.com/role-arn` annotation
(`kubectl -n velero get sa velero -o yaml`) and that the role exists in IAM.

**A backup is `PartiallyFailed`.** `velero backup logs <name> | grep -i error`. A failed
EBS snapshot is usually a missing EC2 permission in `dev-VeleroPolicy-us-east-1`.

**A restored PVC stays `Pending`.** EBS snapshots restore into the same availability zone
the original volume was in. The pod needs a node in that zone.

**The HelmRelease waits on `velero-irsa`.** `velero-irsa-sealed.yaml` is still the placeholder,
or it was sealed against a different cluster's certificate. Redo Step 2.
