# KEDA on dev-eks-us-east-1

[KEDA](https://keda.sh) scales your apps up and down based on events, such as messages
waiting in a queue. This guide covers installing it on `dev-eks-us-east-1` with permission
to read from AWS, for example to scale on the length of an SQS queue.

Flux watches this repo and applies whatever is in it to the cluster. You don't run
`kubectl apply` yourself: to change the cluster, you change the files here and push.

For writing ScaledObjects once KEDA is running, see [../scaledobject-templates](../scaledobject-templates/README.md).

## Terms used in this guide

- **KEDA** - scales your apps up and down based on events, such as messages waiting in a queue.
- **Flux** - watches this Git repo and applies whatever is in it to the cluster. To change the cluster, you commit and push.
- **IRSA** (IAM Roles for Service Accounts) - the EKS feature that lets a Kubernetes service account act as an AWS IAM role, so no AWS keys are stored in the cluster.
- **Service account** - the identity a pod runs as inside Kubernetes. KEDA runs as `keda-operator`.
- **Sealed Secrets** - a tool that encrypts a Kubernetes Secret so it's safe to commit to Git. Only the controller running in the cluster can decrypt it.

## How it works

1. An IAM role in AWS says "the `keda-operator` service account in the `keda` namespace may use me".
2. KEDA's service account is tagged with that role's ARN (its AWS ID).
3. The ARN contains the AWS account ID, and this repo is public, so instead of writing it in plain text, it lives in an encrypted SealedSecret.
4. The Sealed Secrets controller decrypts it, and Flux installs KEDA using the value from it.

## The files

| File | What it does |
| --- | --- |
| `clusters/dev/dev-eks-us-east-1/keda.yaml` | Tells Flux to deploy the `keda/` folder next to it. `dependsOn: infrastructure` makes Flux wait for the `keda` HelmRepository and the Sealed Secrets controller, which `infrastructure` already installs. |
| `infrastructures/base/keda/helmrelease.yaml` | Installs the KEDA chart. Shared with `25c-shared`. Names KEDA's service account `keda-operator` (the IAM role only trusts this exact name). |
| `clusters/dev/dev-eks-us-east-1/keda/kustomization.yaml` | Pulls in the base HelmRelease above, plus the SealedSecret, and applies `patch.yaml`. |
| `clusters/dev/dev-eks-us-east-1/keda/patch.yaml` | This cluster's changes to base: creates the `keda` namespace, and reads the role ARN from the `keda-irsa` Secret instead of base's hand-made ConfigMap. |
| `clusters/dev/dev-eks-us-east-1/keda/sealedsecret.yaml` | The encrypted Secret. It's a placeholder until you do Step 2. |

The IAM role itself isn't in this repo. It's created in the `terraform-infra-v2` repo (Step 1).

## Before you start

You need:

- `kubectl` connected to `dev-eks-us-east-1`. Its API is private, so open the tunnel first:
  see `docs/access.md` in `terraform-infra-v2`. `kubectl --context dev-eks-us-east-1 get nodes` should work.
- the [`flux` CLI](https://fluxcd.io/flux/installation/#install-the-flux-cli)
- the [`kubeseal` CLI](https://github.com/bitnami-labs/sealed-secrets#kubeseal)
- the `aws` CLI, logged in to the dev account
- permission to apply changes in the `terraform-infra-v2` repo

## Step 1: create the IAM role

In the `terraform-infra-v2` repo, open `resources/us-east-1/dev/eks/iam.yaml` and add this
under `service_accounts:`:

```yaml
  dev-irsa-keda-operator-us-east-1:
    namespace_service_account: keda/keda-operator
    attached_policies: ["arn:aws:iam::aws:policy/AmazonSQSReadOnlyAccess"]
```

Then apply the eks stack.

`namespace_service_account` must stay `keda/keda-operator` (namespace/name), or the role won't
trust KEDA. The policy here lets KEDA read SQS; swap it for whatever your KEDA triggers need to read.

## Step 2: create the real SealedSecret

The Sealed Secrets controller is already running on the cluster. Check with:

```
kubectl --context dev-eks-us-east-1 get pods -n sealed-secrets
```

Find the AWS account ID:

```
aws sts get-caller-identity --query Account --output text
```

From the `clusters/dev/dev-eks-us-east-1/keda/` folder, download the cluster's public key.
It's used to encrypt, and it's safe to share:

```
kubeseal --context dev-eks-us-east-1 --fetch-cert \
  --controller-namespace=sealed-secrets \
  --controller-name=sealed-secrets-sealed-secrets > pub-cert.pem
```

The controller name looks doubled because Flux names Helm releases `<namespace>-<name>`.

Then encrypt the Secret. Replace `<ACCOUNT_ID>` with the account ID first:

```
cat <<YAML | kubeseal --format=yaml --cert=pub-cert.pem > sealedsecret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: keda-irsa
  namespace: flux-system
stringData:
  values.yaml: |
    serviceAccount:
      operator:
        annotations:
          eks.amazonaws.com/role-arn: arn:aws:iam::<ACCOUNT_ID>:role/dev-irsa-keda-operator-us-east-1
YAML
```

This overwrites the placeholder `sealedsecret.yaml`. Open it and check that `REPLACE_ME` is gone.
Delete `pub-cert.pem` afterwards; it doesn't belong in the repo.

## Step 3: commit and push

Flux only sees what's in Git, so commit the new file:

```
git add sealedsecret.yaml
git commit -m "Add KEDA IRSA SealedSecret"
git push
```

## Step 4: check that it worked

```
flux --context dev-eks-us-east-1 get kustomizations                 # infrastructure and keda should show Ready: True
flux --context dev-eks-us-east-1 get helmreleases -n flux-system    # keda should show Ready: True
kubectl --context dev-eks-us-east-1 get sa keda-operator -n keda -o yaml | grep role-arn
```

The last command should print the role ARN. If it does, KEDA can use the IAM role.

## Troubleshooting

**KEDA isn't installing.** Until Step 2 is done, this is expected: KEDA needs the `keda-irsa`
Secret, and the placeholder can't be decrypted. Run
`kubectl --context dev-eks-us-east-1 get secret keda-irsa -n flux-system`.
If it's missing, redo Step 2 and make sure you pushed.

**`kubectl` or `kubeseal` says "connection refused".** The tunnel to the cluster isn't open.
See `docs/access.md` in `terraform-infra-v2`.

**KEDA gets "access denied" from AWS.** Check that the role name in the ARN matches the one
Terraform created, and that the role's policy allows the actions your triggers need.
