## Usage

This chart is expected to be used with the [Crossplane provider for AWS Secrets Manager](https://marketplace.upbound.io/providers/upbound/provider-aws-secretsmanager/v1.20.1)
and consumed via an ArgoCD Application of type helm, created via
[Crossplane provider-kubernetes Object](https://github.com/crossplane-contrib/provider-kubernetes/blob/main/examples/object/object-watching.yaml)
```yaml
apiVersion: kubernetes.crossplane.io/v1alpha2
kind: Object
metadata:
  name: foo
  annotations:
    uptest.upbound.io/post-assert-hook: testhooks/validate-watching.sh
    uptest.upbound.io/timeout: "60"
spec:
  watch: true
  references:
  # Use patchesFrom to patch field from other k8s resource to object defined in .spec.forProvider.manifest
  - patchesFrom:
      apiVersion: v1
      kind: Secret
      name: bar
      namespace: default
      fieldPath: metadata.resourceVersion # field that changes every time the object is modified
    toFieldPath: spec.source.helm.valuesObject.nameSuffix
  forProvider:
    manifest:
      apiVersion: argoproj.io/v1alpha1
      kind: Application
      metadata:
        name: bar
        namespace: argocd
      spec:
        project: default
        source:
          chart: aws-secrets-manager-secret
          repoURL: https://baburciu.github.io/aws-secrets-manager-secret/
          targetRevision: 0.2.2
          helm:
            valuesObject:
              fullnameOverride: ekscert
              region: us-east-1
              secret:
                key: key
                name: bar
                namespace: default
        destination:
          server: https://kubernetes.default.svc
          namespace: default
        syncPolicy:
          automated:
            prune: true
            selfHeal: true
          syncOptions:
            - CreateNamespace=true
            - RespectIgnoreDifferences=true
  providerConfigRef:
    name: provider-kubernetes
---
apiVersion: v1
kind: Secret
metadata:
  name: bar
  namespace: default
stringData:
  key: some-value
```
With this applied
```shell
$ kg object.kubernetes.crossplane.io/bar-aws-secrets-manager-secret -n default
NAME                             KIND          PROVIDERCONFIG        SYNCED   READY   AGE
bar-aws-secrets-manager-secret   Application   provider-kubernetes   True     True    9s
 $ kg application bar-aws-secrets-manager-secret -n argocd
NAME                             SYNC STATUS   HEALTH STATUS
bar-aws-secrets-manager-secret   Synced        Healthy
 $ kg application bar-aws-secrets-manager-secret -n argocd -o yaml | yq .status.resources
- group: secretsmanager.aws.upbound.io
  kind: Secret
  name: eks-cert-4623913
  status: Synced
  syncWave: 5
  version: v1beta1
- group: secretsmanager.aws.upbound.io
  kind: SecretVersion
  name: eks-cert-4623913
  status: Synced
  syncWave: -10
  version: v1beta1
 $ kvs bar -n default
key='some-value'
 $ kg secret bar -n default -o yaml | yq .metadata.resourceVersion
4623913
 $ echo some-value-changed-again-to-secret | base64
c29tZS12YWx1ZS1jaGFuZ2VkLWFnYWluLXRvLXNlY3JldAo=
 $ k edit secret bar -n default     # have changed the contents of the secret, like cert-manager would
secret/bar edited
 $ kvs bar -n default
key='some-value-changed-again-to-secret'
 $ kg secret bar -n default -o yaml | yq .metadata.resourceVersion
4624318
 $ kg application bar-aws-secrets-manager-secret -n argocd -o yaml | yq .status.resources
- group: secretsmanager.aws.upbound.io
  health:
    status: Missing
  kind: Secret
  name: eks-cert-4624318
  status: OutOfSync
  syncWave: 5
  version: v1beta1
- group: secretsmanager.aws.upbound.io
  health:
    message: Pending deletion
    status: Progressing
  kind: SecretVersion
  name: eks-cert-4623913
  requiresPruning: true
  status: OutOfSync
  version: v1beta1
- group: secretsmanager.aws.upbound.io
  kind: SecretVersion
  name: eks-cert-4624318
  status: Synced
  syncWave: -10
  version: v1beta1
 $
```

## Pushing chart

### Create Helm Chart Package and Registry

```shell
git checkout -b gh-pages

# Create a docs folder in your repo
mkdir docs

# Package chart to docs folder
helm package . --destination docs
### Successfully packaged chart and saved it to: docs/aws-secrets-manager-secret-0.2.2.tgz

# Create index file
helm repo index docs --url https://baburciu.github.io/aws-secrets-manager-secret/

# These commands create:
# - A .tgz package of your chart
# - An index.yaml file for the chart repository

# Commit and push
git add docs
### $ git add -f docs/aws-secrets-manager-secret-0.2.2.tg
git commit -m "Add Helm chart package"
git push

# Enable GitHub Pages in repo settings https://github.com/baburciu/aws-secrets-manager-secret/settings/pages, pointing to /docs folder
```
![alt text](image.png)

### ArgoCD application

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
spec:
  project: default
  source:
    chart: aws-secrets-manager-secret
    repoURL: https://baburciu.github.io/aws-secrets-manager-secret/
    targetRevision: 0.2.2
```
