---
title: "DigitalOcean Kubernetes Challenge 2021"
date: 2021-12-11
tags: ["kubernetes", "digitalocean", "crossplane"]
---

## The [challenge](https://www.digitalocean.com/community/pages/kubernetes-challenge)

Deploy a solution for configuring Kubernetes "from the inside".

Install [Crossplane](https://crossplane.io), which is like Terraform but you manage the infra from inside Kubernetes, not from outside. Crossplane is an open source Kubernetes add-on that enables platform teams to assemble infrastructure from multiple vendors, and expose higher level self-service APIs for application teams to consume, without having to write any code.

## Initial thoughts

I have used Terraform for more than a year now and to provision infrastructures using plain Kubernetes manifests is a very refreshing idea to me. I feel that Crossplane definitely benefits organizations that are already heavily invested in Kubernetes. Having a unified language to manage both infrastructures and applications makes a lot of sense especially if developers in your team are comfortable working with Kubernetes. This reduces the overhead of learning something completely new like Terraform. That said, Crossplane is still a relatively young project at the time of writing this. There aren't as many providers and the managed resources under each of them are way lesser as compared to Terraform providers.

## Deploy Crossplane in DigitalOcean Kubernetes cluster

Ironically for this project, I have used Terraform to provision the DigitalOcean and Crossplane resources. All Terraform code shown can be found in [this repo](https://github.com/elroy-haw/crossplane-poc/tree/main/digitalocean). Note that nothing shown here is production-ready by any means e.g. I could have used a remote backend to store my Terraform state file but this was merely for a quick proof-of-concept.

### Deploy DigitalOcean resources

First, we will need to configure the DigitalOcean Terraform provider.

```hcl
provider "digitalocean" {
  token = var.do_token
}
```

I have used a variable to set the DigitalOcean API token (which can be created from the DigitalOcean portal). Create a `terraform.tfvars` file with `do_token=<your do token>`, in the same directory with the other `.tf` files will allow you to apply DigitalOcean resources successfully.

Next, let's start by defining a VPC. Local variables can be found in the repo aforementioned.

```hcl
resource "digitalocean_vpc" "vpc" {
  name        = format("%s-vpc", local.project_name)
  region      = local.region
  description = format("VPC for %s", local.project_name)
  ip_range    = local.ip_range
}
```

Last, we can now define a Kubernetes cluster in the VPC.

```hcl
data "digitalocean_kubernetes_versions" "versions" {
  version_prefix = "1.21."
}

resource "digitalocean_kubernetes_cluster" "cluster" {
  name          = format("%s-cluster", local.project_name)
  region        = local.region
  version       = data.digitalocean_kubernetes_versions.versions.latest_version
  vpc_uuid      = digitalocean_vpc.vpc.id
  auto_upgrade  = false
  surge_upgrade = false
  tags          = local.tags

  node_pool {
    name       = format("%s-default-worker-pool", local.project_name)
    size       = local.size
    node_count = 2
    auto_scale = false
    tags       = local.tags
  }
}
```

Here I have used the latest version of Kubernetes supported in DigitalOcean. Note that Crossplane version 1.5 (the one that I will be installing) requires a Kubernetes cluster to be v1.16.0 and above. Check [here](https://crossplane.io/docs/v1.5/reference/install.html) for Crossplane and Kubernetes compatible versions.

In the repo, I have also added an output for the cluster ID for convenience.

```hcl
output "cluster_id" {
  description = "DO Kubernetes cluster ID"
  value       = digitalocean_kubernetes_cluster.cluster.id
}
```

### Deploy Crossplane

First, we will need to configure the Helm Terraform provider.

```hcl
provider "helm" {
  kubernetes {
    host                   = digitalocean_kubernetes_cluster.cluster.kube_config[0].host
    token                  = digitalocean_kubernetes_cluster.cluster.kube_config[0].token
    cluster_ca_certificate = base64decode(digitalocean_kubernetes_cluster.cluster.kube_config[0].cluster_ca_certificate)
  }
}
```

Install the Crossplane Helm chart.

```hcl
resource "helm_release" "crossplane" {
  name             = "crossplane"
  chart            = "crossplane"
  repository       = "https://charts.crossplane.io/stable"
  version          = "1.5.1"
  namespace        = "crossplane-system"
  create_namespace = true
  values           = []
}
```

## Verify Crossplane is up and running

Now that the resources are deployed, let's check to see if they are working as intended.

### Add the DigitalOcean cluster configuration to kubeconfig

DigitalOcean CLI tool `doctl` can be used to update your kubeconfig file easily.

```bash
doctl kubernetes cluster kubeconfig save <paste the cluster_id output value here>
```

Now we can use `kubectl` to interact with the API server of the DigitalOcean Kubernetes cluster.

### Check Crossplane pods

The Crossplane pods are deployed in the `crossplane-system` namespace.

```bash
kubectl get pods -n crossplane-system
```

```bash
NAME                                      READY   STATUS    RESTARTS   AGE
crossplane-7cfcfb84c9-zj99x               1/1     Running   0          92s
crossplane-rbac-manager-cdcc7f487-hld26   1/1     Running   0          92s
```

## Install and configure Azure Provider

For this challenge, I have decided to go with the Azure Provider since I have some free credits lying around.

### Install Azure Provider

There are [multiple ways](https://crossplane.io/docs/v1.5/concepts/providers.html#installing-providers) to install the provider controllers and CRDs. Below uses the raw manifests way to install the Azure Provider.

```yaml
apiVersion: pkg.crossplane.io/v1alpha1
kind: ControllerConfig
metadata:
  name: debug-config
spec:
  args:
    - --debug
---
apiVersion: pkg.crossplane.io/v1
kind: Provider
metadata:
  name: provider-azure
spec:
  package: "crossplane/provider-azure:v0.18.0"
  controllerConfigRef:
    name: debug-config
```

Note that the `ControllerConfig` is not necessary. I added it to check out the Azure controller's debug logs.

The file above is located in the `azure/` folder of the aforementioned repo. Simply apply the manifests to the cluster and the core Crossplane controller will install the provider along with their CRDs.

### Configure Azure Provider

Each provider has its [own ways](https://crossplane.io/docs/v1.5/reference/configure.html) to set up the credentials for Crossplane to utilize when deploying resources on your behalf. The options also vary depending on which managed Kubernetes service Crossplane is deployed into e.g. for AWS EKS, you can opt for using IAM roles for service account instead of IAM user credentials.

For the Azure provider, a service principal has to first be created with the right role and permissions granted. Follow the instructions [here](https://crossplane.io/docs/v1.5/cloud-providers/azure/azure-provider.html) to prepare the base64 encoded secret value. Apply the following manifests to create the Azure provider configuration.

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: azure-account-creds
  namespace: crossplane-system
type: Opaque
data:
  credentials: <paste your base64 encoded Azure credentials here>
---
apiVersion: azure.crossplane.io/v1beta1
kind: ProviderConfig
metadata:
  name: default
spec:
  credentials:
    source: Secret
    secretRef:
      namespace: crossplane-system
      name: azure-account-creds
      key: credentials
```

Note that this will be the default Azure provider which will be used by the Azure Crossplane controller for creating any Azure managed resource without `providerConfigRef`.

## Create Azure resources

Two Azure resources will be created - a resource group and a PostgreSQL server. Apply the following manifests and the Azure Crossplane controller will create the resources.

### Resource group

```yaml
apiVersion: azure.crossplane.io/v1alpha3
kind: ResourceGroup
metadata:
  name: sqlserverpostgresql-rg
spec:
  location: East US
```

### PostgreSQLServer

```yaml
apiVersion: database.azure.crossplane.io/v1beta1
kind: PostgreSQLServer
metadata:
  name: sqlserverpostgresql41241
spec:
  forProvider:
    administratorLogin: myadmin
    resourceGroupNameRef:
      name: sqlserverpostgresql-rg
    location: East US
    sslEnforcement: Disabled
    version: "9.6"
    sku:
      tier: GeneralPurpose
      capacity: 2
      family: Gen5
    storageProfile:
      storageMB: 20480
  writeConnectionSecretToRef:
    namespace: crossplane-system
    name: sqlserverpostgresql-conn
```

Note that the `metadata.name` has to be unique.

## Ending thoughts

This post covers the very basics of using and deploying Crossplane. There are more that I am planning to explore in the near future, which includes:

- Using GitOps approach to manage the managed and composite resources
- Create composite resources for production-ready EKS/AKS clusters with bare minimum add-ons
- Experiment with AWS Provider's managed resources

Thanks for reading this post till the end!
