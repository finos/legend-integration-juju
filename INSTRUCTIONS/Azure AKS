TODO: Headers

Using Juju to drive Kubernetes workloads on AKS requires installing some prerequisite software:
TODO: Links
- Juju client
- Azure CLI
- Kubernetes client

## Install the Juju client

You will need to install Juju. If you're on Linux, we recommend using `snap`:

```bash
sudo snap install juju --classic
```

We also provide other installation methods as described on our detailed [installation documentation](TODO).

### Install the Azure CLI

The [Azure CLI](https://docs.microsoft.com/en-us/cli/azure/) is a cross-platform command-line utility for interacting with Azure. For installation instructions, please visit the [Azure CLI web pages](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli).

Install the Kubernetes client

`kubectl` is the command-line client for directly interacting with a Kubernetes cluster. The Azure CLI provides a sub-command that will install `kubectl` for you:

```bash
az aks install-cli
```

## Provision a Kubernetes cluster on Azure</h2></a>

This guide will assume that your Azure account has sufficient access to an Azure Subscription to allow the creation of Resource Groups and Azure Kubernetes Service clusters.

Azure resources can be created from the Azure CLI, Azure Portal or using Azure Powershell. This guide will focus on the creation of a cluster using the Azure CLI. For information on how deploy using the Azure Portal see the [Azure Documentation](https://docs.microsoft.com/en-us/azure/aks/kubernetes-walkthrough-portal) or go directly to the portal to [create your cluster](https://portal.azure.com/#create/microsoft.aks).

[note]
Please refer to the Azure documentation for authoritative information on how to manage services and accounts with Azure.
[/note]

### Log into the Azure CLI

To log in from the command-line, execute:

```bash
az login
```

This will open a browser window (or provide an authentication link) that will allow you to authenticate with Microsoft Azure. Once logged in, confirm you have access to a subscription by running:

```bash
az account list -o table
```

## Create the Kubernetes cluster with the Azure CLI


Now that we've created a Resource Group in the correct region, we can provision a new Azure Kubernetes Service cluster.

### Create a resource group

A "Resource Group" is a link between services and a geographic region for billing and authentication purposes. To create a Resource Group for your project, provide a region and a human-readable name to the `az group create` command.

```bash
az group create -l <region> -n <resource-group>
```

TODO: Fold the section below
"Help selecting an Azure Region"

Each Azure region has different capabilities. In particular, different  Kubernetes versions are supported within each region. To get the list of regions, execute:

```bash
juju regions azure
```

The Azure CLI can inspect regions and enumerate supported Kubernetes versions.

```bash
az aks get-versions -l <region> -o table
```

### Provision the AKS cluster with the Azure CLI

The `az aks create` command provisions a cluster. The command below will create a cluster in the specified Resource Group, generating a new SSH keypair for the underlying nodes:

```bash
az aks create -g <resource-group> -n <k8s-cluster-name> --generate-ssh-keys
```

### Access the cluster

The Azure CLI is able to configure `kubectl` on your behalf to access your new AKS cluster:

```bash
az aks get-credentials -g <resource-group> -n <k8s-cluster-name>
```

Verify that you have access by enumerating the nodes in the cluster:

```bash
kubectl get nodes
```

## Connect Juju to your Kubernetes cluster


There are three distinct stages to configuring your AKS cluster for management with Juju:

- Register the cluster as a Kubernetes cloud

- Deploy a Juju controller to the cluster

- Create a model to house our applications

### Register the cluster as a Kubernetes cloud

In Juju's vocabulary, a [cloud](https://juju.is/docs/clouds) is a space that can host deployed workloads. To register a Kubernetes cluster with Juju, execute the `juju add-k8s` command. The `<cloud>` argument is your specified name for the cloud, and will be used by Juju to refer to the cluster.

```bash
juju add-k8s --aks --resource-group <resource-group> --cluster-name <k8s-cluster> <cloud>
```

### Create a controller

Juju, as an active agent system, requires a [_controller_](https://juju.is/docs/controllers) to get started. Controllers are created with the `juju bootstrap` command that takes as arguments  a cloud's name and, optionally, also our intended name for the controller.

```bash
juju bootstrap <cloud>
```

### Create a model

A model is a workspace for inter-related applications. They correspond (roughly) to a Kubernetes namespace. To create one, use `juju add-model` with a name of your choice:

```bash
juju add-model <model-name>
```

## Deploy workloads

You're now able to deploy workloads into your model. Workloads are provided by charms (and sets of charms called bundles). You can find a list of Kubernetes charms [on Charmhub](https://charmhub.io/?platform=kubernetes)!

## Clean up

The following instructions allow you to undo everything we just did!

TODO: format the below as a note

The following commands are destructive, and will destroy all running workloads that were deployed with Juju, then remove the Resource Group and AKS cluster. Ensure there is nothing in the Resource Group you need before proceeding!

[/note]

Remove the Juju controller and any workloads deployed:

```bash
juju kill-controller -y -t0 <controller>
```

Delete the Azure Resource Group (and therefore the cluster)

```bash
az group delete -g <resource-group>
```
