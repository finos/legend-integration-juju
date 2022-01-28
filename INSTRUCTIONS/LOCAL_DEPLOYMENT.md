# How to run Legend locally using Charmed Operators
This tutorial will cover how to use Juju and Charmed Operators to deploy an instance of the [FINOS Legend](https://www.finos.org/legend) stack on your local workstation, using [Microk8s](https://microk8s.io/). It can be used for evaluation and demonstration purposes, as well as  development of new Legend features or other FINOS *Charmed* applications.

These steps may be carried out on any Linux distribution that has a [Snap](https://snapcraft.io/) package manager installed. The steps in this tutorial have been tested on [Ubuntu 20.04](https://releases.ubuntu.com/focal/). Following the steps in this tutorial does require that your Linux host has a working Internet connection.

The deployment is composed by the following steps:
- [Install Microk8s](#Install-Microk8s)
- [Install Juju](#Install-Juju)
- [Bootstrap Juju](#Bootstrap-Juju)
- [Add the Juju Legend Model](#Add-the-Juju-Legend-Model)
- [Deploy the Legend Bundle](#Deploy-the-Legend-Bundle)
- [Setup and Configure GitLab](#Setup-and-Configure-GitLab)
- [Monitor Juju status](#Monitor-Juju-status)
- [Authorize the Legend Gitlab Application](#Authorize-the-Legend-Gitlab-Application)
- [Authenticate user on Legend SDLC](#Authenticate-user-on-Legend-SDLC)
- [Use Legend](#Use-Legend)
- [Destroy setup](#Destroy-setup) (optional)

## Install Microk8s
[Microk8s](https://microk8s.io/) is a *micro* Kubernetes distribution that runs locally; you can install it on Linux running the following commands:
```bash
sudo snap install microk8s --classic
sudo snap alias microk8s.kubectl kubectl
source ~/.profile
sudo usermod -a -G microk8s $USER
sudo chown -f -R $USER ~/.kube
newgrp microk8s
```

On Mac, assuming you have XCode, Python 3.9 (with brew, you can run `brew link --overwrite python@3.9`) and [HomeBrew](brew.sh) installed, you can simply run `brew install ubuntu/microk8s/microk8s`.

To configure and start `microk8s`, simply run:

```bash
microk8s enable dns storage ingress
microk8s status --wait-ready
```

Below is the expected output; before moving forward, wait to see that `dns`, `ingress` and `storage` are listed as `enabled`.

```bash
microk8s is running
high-availability: no
  datastore main nodes: 127.0.0.1:19001
  datastore standby nodes: none
addons:
  enabled:
    dns                  # CoreDNS
    ha-cluster           # Configure high availability on the current node
    ingress              # Ingress controller for external access
    storage              # Storage class; allocates storage from host directory
...
```

## Install Juju
To install Juju, you can follow [the instructions in the docs](https://juju.is/docs/olm/installing-juju) or simply install a Juju with the command line `sudo snap install juju --classic`; on MacOS, you can use brew with `brew install juju`; run `juju status` to check if everything is up.

If you're interested to know how to run Juju on your cloud of choice, checkout [the official docs](https://juju.is/docs/olm/clouds); you can always run `juju clouds` to check your configured clouds. In the instructions below, we will always use `microk8s`, but you can replace it with the name of the cloud you're using.

## Bootstrap Juju
In Juju terms, "bootstrap" means "create a Juju controller", which is the part of Juju that runs in your cluster and controls the applications.

```bash
juju bootstrap microk8s finos-legend-controller
```

You are free to use any name other than `finos-legend-controller` but it must be consistent through the rest of this tutorial.

## Add the Juju Legend Model
[Juju models](https://juju.is/docs/olm/models) are a logical grouping of applications and infrastructure that work together to deliver a service or product. In Kubernetes terms, models are effectively namespaces. Models are fundamental concepts in Juju and implement service isolation, access control, repeatability and boundaries.

You can add a new model with
``` bash
juju add-model finos-legend
```

## Deploy the Legend Bundle
When you deploy an application with Juju, the installation code in the charmed operator will run and set up all the resources and environmental variables needed for the application to run properly. In the case of this tutorial, we are deploying a *bundle*, which describes applications to be deployed and relationships between them.

Deploy the [finos-legend-bundle](https://github.com/finos/finos-legend-bundle) using 
``` bash
juju deploy finos-legend-bundle --trust --channel=edge
```

In another terminal, you can check the deployment status and the integration code running using `watch --color juju status --color`.

You'll notice that the Unit `gitlab-integrator/0` will get to `blocked` status; this is expected, as you'll need to [Setup and Configure GitLab](#Setup-and-Configure-GitLab).

## Setup and Configure GitLab
To run Legend, you need to either run a GitLab instance somewhere, or use GitLab.com; the type of installation really depends on user's requirements, there is a secion in `DEPLOY_GITLAB.md` that talks about that (TODO).

If this is your first experience with Legend, we suggest you starting with GitLab.com as per the instructions below. If, instead, you're interested to test a Legend deployment with a local GitLab, please follow instructions on `DEPLOY_GITLAB.md`.

### GitLab.com
1. Create an account on gitlab.com
2. Create an application:
- Login GitLab.com, click your profile picture on the right upper corner and click "Preferences". 
- Click "Applications" on the left menu. 
- Create a new application with the following information:
  - name
  - enter a dummy URI in the redirect URL field (dummy.org will work for now) 
  - enable the following scopes: 
    - API
    - Open ID
    - Profile
- Click "Save Application".

3. Copy the "Application ID" and "Secret" information.

In your terminal, run the following command with the infromation you jsut copied
```bash
juju config gitlab-integrator gitlab-client-id="<Application ID>" gitlab-client-secret="<Secret>"
```

Retrieve the GitLab URI from the GitLab Integrator charm using  
```bash
juju show-unit gitlab-integrator/0 | grep legend-gitlab-redirect-uris
```
  
Go back to the "Applications" page on GitLab.com and edit the application you created. Replace the dummy URI with the URIs retrieved from the command above and save the application. 

## Monitor Juju status
Run `juju status` to see the applications reacting to the configuration change. As a result of this change, your FINOS Legend deployment should complete, the output should look like this.

```bash
Model  Controller  Cloud/Region        Version  SLA          Timestamp
lma    microk8s    microk8s/localhost  2.9.16   unsupported  14:19:18+01:00

App                Version  Status  Scale  Charm                               Store     Channel  Rev  OS          Address         Message
gitlab-integrator           active      1  finos-legend-gitlab-integrator-k8s  charmhub  edge      25  kubernetes  10.152.183.65
ingress-engine              active      1  nginx-ingress-integrator            charmhub  stable    24  kubernetes  10.152.183.41
ingress-sdlc                active      1  nginx-ingress-integrator            charmhub  stable    24  kubernetes  10.152.183.188
legend-db                   active      1  finos-legend-db-k8s                 charmhub  edge      13  kubernetes  10.152.183.219
legend-engine               active      1  finos-legend-engine-k8s             charmhub  stable    13  kubernetes  10.152.183.97
legend-ingress              active      1  nginx-ingress-integrator            charmhub  stable    24  kubernetes  10.152.183.63
legend-sdlc                 active      1  finos-legend-sdlc-k8s               charmhub  stable    36  kubernetes  10.152.183.238
legend-studio               active      1  finos-legend-studio-k8s             charmhub  stable    14  kubernetes  10.152.183.66
mongodb                     active      1  mongodb-k8s                         charmhub  edge       8  kubernetes  10.152.183.13

Unit                  Workload  Agent  Address      Ports  Message
gitlab-integrator/0*  active    idle   10.1.123.61
ingress-engine/0*     active    idle   10.1.123.46         Ingress with service IP(s): 10.152.183.23
ingress-sdlc/0*       active    idle   10.1.123.50         Ingress with service IP(s): 10.152.183.158
ingress-studio/0*     active    idle   10.1.123.62         Ingress with service IP(s): 10.152.183.206
legend-db/0*          active    idle   10.1.123.63
legend-engine/0*      active    idle   10.1.123.45
legend-sdlc/0*        active    idle   10.1.123.43
legend-studio/0*      active    idle   10.1.123.48
mongodb/0*            active    idle   10.1.123.21
```

Note that all `Workload`,  `Status` are `active`; you can now go ahead and [authorize the Legend user against GitLab](#Authorize-Legend-user-against-Gitlab).

## Authorize the Legend Gitlab Application
Using `juju status`, grab the `Address` of the `Unit` called `legend-studio/0*` and point your browser to `http://<STUDIO-IP>:8080`; you should be redirected to a GitLab App authorization page.

![Authorize Legend Charm](./images/authorize-legend-charm.png)

Click on `Authorize` and you should be redirected back to Legend Studio. Note the `Unauthorized` pop-up dialog box in the lower right corner. ![Unauthorized User](./images/unauthorized-studio-charm.png) This is expected, since the Legend user needs to [authenticate against the SDLC component]((#authorize-the-user-to-use-legend)).

## Authenticate user on Legend SDLC
As before, using `juju status`, grab the `Address` of the `Unit` called `legend-sdlc/0*` and point your browser to `http://<SDLC_IP>:7070/api/auth/authorize`; you should get redirected again to GitLab, as follows. Click on `Authorize` and you should be redirected to Legend Studio.

![Authorize User](./images/authorize-charm-user.png)

## Use Legend
From `http://<STUDIO_IP>:8080/studio` you should be able tostart using Legend Studio!

![Authorized Legend](./images/authorized-studio-charm.png)

## Destroy setup
To remove all deployed Legend applications, remove any data and reset Juju to the state it was before Legend was deployed, you can destroy the controller (created during the bootstrapping process). **This is a non-reversible operation - all your data created in studio will be lost!**

```bash
juju destroy-controller -y --destroy-all-models --destroy-storage finos-legend-controller
```

To also remove Juju and MicroK8s, you can run:
``` bash
sudo snap remove juju --purge
sudo snap remove microk8s --purge
```

On MacOS, use `brew remove juju ubuntu/microk8s/microk8s`.
