# Producing `sealed-github-oauth.yaml`

This directory's `kustomization.yaml` references a file that is **not committed
yet**, because it cannot be produced without a GitHub OAuth App that does not
exist. Until it is added, `kustomize build` on this directory fails — which is
the intended behaviour: the alternative is installing Argo CD with a Dex that
crash-loops on a missing credential.

## 1 · Create the GitHub OAuth App

GitHub → the `Ubuntu-25c-market-prep` org → Settings → Developer settings →
OAuth Apps → New OAuth App.

| Field | Value |
|---|---|
| Application name | `Argo CD — u25c-shared` |
| Homepage URL | the Argo CD hostname, once `@istio` assigns one |
| Authorization callback URL | `https://<argocd-host>/api/dex/callback` |

**Own it at the organisation, not personally.** An OAuth App owned by an
individual leaves with that individual — which is the offboarding question
raised in `ops-program#52`.

The callback path is `/api/dex/callback`, not `/auth/callback`. Getting it wrong
produces a redirect_uri mismatch at login and nothing before it.

## 2 · Seal the credentials

```bash
kubectl create secret generic argocd-github-oauth \
  --namespace argocd \
  --from-literal=clientID='<client id>' \
  --from-literal=clientSecret='<client secret>' \
  --dry-run=client -o yaml \
| kubeseal \
  --controller-namespace sealed-secrets \
  --controller-name sealed-secrets-sealed-secrets \
  --format yaml \
> sealed-github-oauth.yaml
```

Then add the label Argo CD uses to find it, under `spec.template.metadata`:

```yaml
  template:
    metadata:
      name: argocd-github-oauth
      namespace: argocd
      labels:
        app.kubernetes.io/part-of: argocd
```

That label is what lets `argocd-cm` resolve `$argocd-github-oauth:clientSecret`.
Without it the reference silently stays a literal string and Dex refuses to
start.

`kubeseal` defaults to **strict** scope — bound to this name in this namespace —
which is what `ops-program#51` asks for. Do not widen it.

## 3 · Check what you are about to commit

```bash
grep -c 'encryptedData' sealed-github-oauth.yaml   # 1
grep -i 'clientSecret:' sealed-github-oauth.yaml   # only inside encryptedData
```

Every repository here is public. The ciphertext is safe to commit; the
`kubectl create secret` output above is **not**, so do not save it to a file.

## 4 · Delete this file

Once `sealed-github-oauth.yaml` exists, this README has served its purpose and
belongs in the PR description rather than in the tree.
