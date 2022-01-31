
# Deploy Legend locally (MicroK8s) using Juju 
This tutorial will cover how to use Juju and Charmed Operators to deploy an instance of the [FINOS Legend](https://www.finos.org/legend) stack on your local workstation, using [Microk8s](https://microk8s.io/). 

### Audience 
This document can be used for evaluation and demonstration purposes. This is also useful for development of new Legend features or other FINOS *Charmed* applications. 

### Prerequisites
These steps may be carried out on any Linux distribution that has a [Snap](https://snapcraft.io/) package manager installed. The steps in this tutorial have been tested on [Ubuntu 20.04](https://releases.ubuntu.com/focal/). 

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
To install Juju, you can follow [the instructions in the docs](https://juju.is/docs/olm/installing-juju) or simply install a Juju with the command line `sudo snap install juju --classic`; on MacOS, you can use brew with `brew install juju` (this will run an upgrade, if the `juju` brew formula is already installed); run `juju status` to check if everything is up.

If you're interested to know how to run Juju on your cloud of choice, checkout [the official docs](https://juju.is/docs/olm/clouds); you can always run `juju clouds` to check your configured clouds. In the instructions below, we will always use `microk8s`, but you can replace it with the name of the cloud you're using.

## Bootstrap Juju

In Juju terms, "bootstrap" means "create a Juju controller", which is the part of Juju that runs in your cluster and controls the applications. You can bootstrap Juju to the EKS cluster by running
``` bash
juju bootstrap microk8s finos-legend-controller
```

Running `juju status` will now show the Juju Controller in your cloud.

## Add the Juju Legend Model
[Juju models](https://juju.is/docs/olm/models) are a logical grouping of applications and infrastructure that work together to deliver a service or product. In Kubernetes terms, models are effectively namespaces. Models are fundamental concepts in Juju and implement service isolation, access control, repeatability and boundaries.

You can add a new model with
``` bash
juju add-model finos-legend
```

## Deploy the Legend Bundle
When you deploy an application with Juju, the installation code in the charmed operator will run and set up all the resources and environmental variables needed for the application to run properly. In the case of this tutorial, we are deploying a *bundle*, which describes applications to be deployed and relationships between them.

Deploy the finos-legend-bundle in the finos-legend model using the command line :
``` bash
juju deploy finos-legend-bundle --trust --channel=edge
```

In another terminal window, you can see the applications being deployed and the integration code running
``` bash
watch --color juju status --color
```

You'll notice that the Unit `gitlab-integrator/0` will get to `blocked` status; this is expected, as you'll need to [Setup and Configure GitLab](#Setup-and-Configure-GitLab).

## Setup and Configure GitLab
To run Legend, you need to either run a GitLab instance somewhere, or use GitLab.com; the type of installation really depends on user's requirements, there is a secion in `DEPLOY_GITLAB.md` that talks about that (TODO).

If this is your first experience with Legend, we suggest you starting with GitLab.com as per the instructions below. If, instead, you're interested to test a Legend deployment with a local GitLab, please follow instructions on `DEPLOY_GITLAB.md`.

If you are using gitlab.com, follow the following three steps.

#### 1. Create a GitLab Application

1. Create an account on gitlab.com
2. Create an application:
- Login GitLab.com, click your profile picture on the right upper corner and click "Preferences". 
- Click "Applications" on the left menu. 
- Create a new application with the following information:
  - name
  - Check the `Confidential` checkbox 
  - enter a dummy URI in the redirect URL field (http://dummy.org will work for now) 
  - enable the following scopes: 
    - API
    - Open ID
    - Profile
- Click "Save Application".

On the following page, make a note of the `Application ID` and `Secret`. 

#### 2. Pass the application details to the Legend stack

In your terminal run the following command to pass the `Application ID` and `Secret` to the Legend stack
``` bash
juju config gitlab-integrator gitlab-client-id="<Application ID>" gitlab-client-secret="<Secret Id> "
```

#### 3. Update the GitLab application URIs

Retrieve the GitLab URIs from the GitLab Integrator charm using  
```bash
juju show-unit gitlab-integrator/0 | grep legend-gitlab-redirect-uris
```
  
Go back to the "Applications" page on GitLab.com and edit the application you created. Replace the dummy URI with the URIs retrieved from the command above and save the application. 

Run `watch --color juju status --color` to see the applications reacting to the configuration change. As a result of this change, your FINOS Legend deployment should complete,

## Accessing the Legend Studio dashboard

> ⚠️ Browser settings and cookies
> 
> The Legend stack rely on cookies to authenticate user across the applications. A recent change on how [samesite cookies](https://web.dev/samesite-cookies-explained/] are handled by Google Chrome and Mozilla Firefox can block the authentication process. To avoid problems with cookies, before starting the authentication process:
>  
> - make sure you use the private mode
> - allow samesite cookies in your browser
> - clear the browser's cache 
> - [if you are using Firefox](https://support.mozilla.org/en-US/kb/enhanced-tracking-protection-firefox-desktop#w_how-to-tell-when-firefox-is-protecting-you), disable Enchanced Tracking Protection for the Studio and SDLC pages.

### Authorize the GitLab user and application

Run `juju status` on your terminal, grab the `Address` of the `Unit` called `legend-sdlc/0*` and point your browser to `http://<SDLC_IP>:7070/api/auth/authorize`; Click on `Authorize` and you should see the text `Authorized`. 

Similarly, run `juju status` on your terminal, grab the `Address` of the `Unit` called `legend-studio/0*` and point your browser to `http://<STUDIO-IP>:8080`. Click on `Authorize`. 

If the process was sucessful, you will be able to see the Legend Studio dashboard on `http://<STUDIO-IP>:8080/studio/-/setup`! 🎉


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
