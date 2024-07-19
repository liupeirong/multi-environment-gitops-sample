# Introduction

This sample GitOps repo demonstrates a way to organize Kubernetes applications packaged in Helm charts for different environments
 such as dev, test, prod, and also in the case of industrial scenarios, for different manufacturing plants or retail stores,
 each requiring some levels of customization of configurations. It is inspired by this [flux2-kustomize-helm-example](https://github.com/fluxcd/flux2-kustomize-helm-example/tree/main)
 and its derived [gitops-flux2-kustomize-helm multi-tenant sample using Azure Arc GitOps extension](https://github.com/Azure/gitops-flux2-kustomize-helm-mt).
 What this repo adds is how to manage configurations
 when you have potentially a hundred plants or stores to deploy applications to.

When the number of environments reaches hundreds or thousands, just managing that many folders could be
 challenging. Other solutions leveraging a configuration databases may be necessary. This is beyond the scope of this sample.

## What's in the repo?

Samples in this repo mimic applications that need to be configured and
 deployed to on-site clusters in industrial scenarios where, for example,
 there are multiple regions, each region has multiple plants/sites, each
 site has multiple manufacturing lines. These assets and their attributes
 are stored in the [assets](./assets/) folder.

There are two sample apps in this repo:

- siteconf - assuming there's a Kubernetes cluster running in a plant that
 supports multiple manufacturing lines, this app shows how to create a
 site-level configuration that changes based on the lines configured on the
 cluster and each line's configuration. This sample only has a `ConfigMap`.
- busybox - this sample could run one instance per manufacturing line on a cluster,
 each in its own namespace. It deploys a pod with customized configurations
 for that line. It also demonstrates how to use Azure Arc Key Vault extension
 to manage secrets.

### Repo Structure

```txt
|- apps          # contains the base and customized config for each app 
|-   app1        # app1 corresponds to the helm chart in charts folder
|-     base      # base configuration for the app
|-     env1      # configuration override for environment 1 
|-     env2      # configuration override for environment 2
|-     ...
|-   app2
|-     base
|-     env1
|-     env2
|-     ...
|- assets         # attributes of assets that don't change by apps
|-                # just an example, you can define your own.
|-   regions.yaml 
|-   sites.yaml   
|-   lines.yaml
|- charts           # helm charts for apps
|-   chart_for_app1 # chart folder for app1
|-   chart_for_app2 # chart folder for app2
|-   ...
|- sample-manifests # sample manifests to create secrets outside of gitops
|-   acr-cred.yaml  # create secrets for Azure Container Registry
|-   akv-cred.yaml  # create secrets for Azure Key Vault
```

## How is configuration customized for each environment?

Flux, Kustomize, and Helm work together to manage configuration for
multiple environments in a scalable way:

- `charts\app\values.yaml` contains common or default configurations for an app.
 Flux requires this file to exist in the Helm chart folder.
- `apps\app\values.yaml` is optional and can also contain common or default
 configurations that you don't want to put in the chart.
- `apps\app\env\` can have any number of values yaml files to define configurations
 specific to the environment. They can override the default configurations or add
 new.

To put it all together,

- The Flux HelmRelease definition has a [valuesFiles](./apps/busybox/base/release.yaml)
 section where you can specify a list of values files.
- The [kustomization.yaml](./apps/busybox/dev/kustomization.yaml) in each environment
 appends the values files for its specific environment to the base `valuesFiles`.
- Helm substitutes the app template with the values provided in the `valuesFiles`
 in the order the values files are listed.

## Try it out

You can hydrate the helm templates on the local machine without deploying to a cluster.

1. Apply Kustomization. For example, from the root folder of the repo,
 run `kubectl kustomize apps\busybox\hou01` to kustomize `busybox` app for environment
 `hou0`.
1. From the above output, take the list of `valuesFiles` and provide them to this
 command `helm template charts\busybox -f <valuesFile1.yaml> -f <valuesFile2.yaml> ...`
1. Verify the final hydrated configuration is what's defined for the specific environment.

## Deploy using Azure Arc GitOps Extension

[Azure Arc GitOps extension](https://learn.microsoft.com/en-us/azure/azure-arc/kubernetes/tutorial-use-gitops-flux2?tabs=azure-cli)
 is based on Flux. Some of the differences include:

- Flux manifests are stored as resources in Azure instead of `.flux-system`
 in the GitOps repo.
- Azure provides a central view of Flux configuration objects such as
 `GitRepository`, `Kustomization`, `HelmRelease`, and their reconciliation
 in the Azure portal.
- Flux [multi-tenancy](https://learn.microsoft.com/en-us/azure/azure-arc/kubernetes/conceptual-gitops-flux2#multi-tenancy) is enabled by default.

You can [configure Arc GitOps to deploy your application either from the portal](https://learn.microsoft.com/en-us/azure/azure-arc/kubernetes/tutorial-use-gitops-flux2?tabs=azure-portal#apply-a-flux-configuration)
 or using az cli as the following:

```bash
az k8s-configuration flux create -g <resource-group> -c <arc-cluster> -n <configuration-name> --namespace busyboxns -t connectedClusters --scope namespace -u <url-to-this-repo> --branch main --kustomization name=<kustomization-name> path=./apps/busybox/<environment-name> prune=false --https-user <git-username> --https-key <git-password>

az k8s-configuration flux create -g <resource-group> -c <arc-cluster> -n <configuration-name> -t connectedClusters --scope cluster -u <url-to-this-cluster> --branch main --kustomization name=<kustomization-name> path=./apps/siteconf/<environment-name> prune=false --https-user <git-username> --https-key <git-password>
```

This will create `GitRepository` and `Kustomization` objects for Flux to
 reconcile. Note that in this sample, we scoped the configuration to `namespace`
 for busybox, meaning the Flux controller cannot access resources outside the
 specified namespace. The namespace itself, ex. `busyboxns` must already exist.
 On the other hand, we scoped the configuration for siteconf to `cluster`,
 and [a namespace is created](./apps/siteconf/base/kustomization.yaml#L5) during the deployment.

Also note that by default, Arc GitOps enables multi-tenancy. So the `GitRepository`s
 used by all the deployment manifests must be in this namespace. For example,
 `sourceRef` in [release.yaml](./apps/busybox/base/release.yaml#L16) cannot be in
 another namespace, and the `GitRepository` name, `busyboxcfg`, must match the
 `configuration-name` in the above `az k8s-configuration` command.

### Secrets Management

There are multiple ways to manage secrets in Kubernetes with Flux. None is easy.

_Azure Key Vault_

While Arc enabled cluster can install [Azure Key Vault extension](https://learn.microsoft.com/en-us/azure/azure-arc/kubernetes/tutorial-akv-secrets-provider) to store secrets,
 this extension works based on a CSI driver. So the workload will need to mount
 a CSI volume into a pod to access the secrets. This can't be used for secrets
 such as credentials for container registries that needs to pull container
 images to create a pod to begin with.

_SOPS_

While SOPS has Azure Key Vault integration and Flux supports SOPS, it requires
 Workload Identity to be enabled in the cluster. Workload Identity is not yet
 available for Arc enabled Kubernetes clusters at the time of this writing.

_Kubernetes Secrets_

This is simple but difficult to manage at scale.

In this sample, to keep things simple,

- for secrets that are used outside of pods, create them as Kubernetes secrets outside
 of Flux. For example,

  - [credential for pulling images from container registry](./sample-manifests/acr-cred.yaml)
  - [credential for a service principal to access Key Vault](./sample-manifests/akv-cred.yaml)

- for secrets accessed by pods, use _Azure Key Vault_ extension as shown
 in the busybox Helm chart.

### Troubleshooting Flux

To troubleshoot deployment failure, you can use these commands whether or
 not using Arc GitOps extension.

```bash
# to check a kustomization
kubectl describe kustomization <name> -n <namespace>
# to check a helmrelease, this could provide more info than get -o yaml
kubectl describe hr <release> -n <namespace>

# to check which reconcilation failed
flux get all -n <namespace> --status-selector ready=false
# to get logs of flux sources: gitRepository, helmchart...
flux get sources all -n <namespace>
# flux logs
flux logs -n <namespace> 

# to see why a release failed 
helm list -n <namespace>
helm history <release> -n <namespace>

# to force an immediate reconciliation
flux reconcile kustomization <name> --with-source -n <namespace>
flux reconcile hr <release> --with-source -n <namespace>

# If a deployment reached ProgressDeadlineExceeded due to bad imagePullCredential
# or missing resources like service account, it won't redeploy even after you
# fixed the missing resource.
# Manually changed/deleted resources won't be reconciled by HelmRelease.
# If you delete a deployment, you will need to run the following to get the
# deployment again.
flux suspend hr <release> -n <namespace>
flux resume hr <release> -n <namespace>
```
