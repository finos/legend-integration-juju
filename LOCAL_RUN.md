# How to run Legend locally using Charmed Operators
This tutorial will cover how to use Juju and Charmed Operators (https://juju.is/) to deploy an instance of the [FINOS Legend](https://www.finos.org/legend) stack on a Kubernetes cluster running on your laptop. We will use [Microk8s](https://microk8s.io/) to create the cluster and Juju to deploy and manage the Legend applications. 

This deployment can be used for evaluation and demonstration purposes, as well as local development of new Legend features or other FINOS *Charmed* applications. The same instructions, however, can be used in a variety of [clouds and Kubernetes flavours](https://juju.is/docs/olm/clouds) supported by Juju. 


The steps in this tutorial have been tested on [Ubuntu 20.04](https://releases.ubuntu.com/focal/) machine with an Internet connection. You will need a [GitLab](https://gitlab.com/) account with persmissions to create applications. 

The deployment is composed by the following steps:

_Create a Kubernetes cluster_
- [Install Microk8s](#Install-Microk8s)

_Prepare the cluster for Legend_
- [Install Juju](#Install-Juju)
- [Bootstrap Juju](#Bootstrap-Juju)
- [Add the Juju Legend Model](#Add-the-Juju-Legend-Model)

_Deploy the Charmed Operators_
- [Deploy the Legend Bundle](#Deploy-the-Legend-Bundle)
- [Setup and Configure GitLab](#Setup-and-Configure-GitLab)
- [Monitor Juju status](#Monitor-Juju-status)

_Configure the Legend Stack_
- [Authorize the Legend Gitlab Application](#Authorize-the-Legend-Gitlab-Application)
- [Authenticate user on Legend SDLC](#Authenticate-user-on-Legend-SDLC)

- [Use Legend](#Use-Legend)

- [Destroy setup](#Destroy-setup) (optional)

## Install Microk8s
[Microk8s](https://microk8s.io/) is a *micro* Kubernetes distribution that creates a fully functional Kubernetes cluster in your laptop; you can install it on Linux running:
``` bash
sudo snap install microk8s --classic
```
Instructions for other operating systems are also available at https://microk8s.io/docs/install-alternatives.

The following commands will 
- add an alias for kubectl;
- create a new shell session with active membership to the microk8s group (you can also achieve this by logging out, and logging back in again);
- enable the dns, storage and ingress add-ons;
  - This operation will take a few moments to complete. The `--wait-ready` flag waits for microK8s to be ready.  

``` bash
sudo snap alias microk8s.kubectl kubectl
source ~/.profile
sudo usermod -a -G microk8s $USER
sudo chown -f -R $USER ~/.kube
newgrp microk8s
microk8s status --wait-ready
```

After a short while, you should see the follwoing:

TODO --wait-ready does not wait
``` bash
$ microk8s status --wait-ready
microk8s is running
high-availability: no
  datastore master nodes: 127.0.0.1:19001
  datastore standby nodes: none
addons:
  enabled:
    dns                  # CoreDNS
    ha-cluster           # Configure high availability on the current node
    ingress              # Ingress controller for external access
    storage              # Storage class; allocates storage from host directory
```


On Mac, assuming you have XCode, Python 3.9 (with brew, you can run `brew link --overwrite python@3.9`) and [HomeBrew](brew.sh) installed, you can simply run `brew install ubuntu/microk8s/microk8s`.


## Install Juju
To install Juju, you can follow [the instructions in the docs](https://juju.is/docs/olm/installing-juju) or simply install the Juju snap with `sudo snap install juju --classic`; on MacOS, you can use brew with `brew install juju`; run `juju status` to check if everything is up.

## Bootstrap Juju
In Juju terms, "bootstrap" means "create a Juju controller", which is the part of Juju that runs in your cluster and controls the applications.

```bash
juju bootstrap microk8s
```

## Add the Juju Legend Model
[Juju models](https://juju.is/docs/olm/models) are a logical grouping of applications and infrastructure that work together to deliver a service or product. In Kubernetes terms, models are effectively namespaces. Models are fundamental concepts in Juju and implement service isolation, access control, repeatability and boundaries.

You can add a new model with
``` bash
juju add-model <model-name>
```

We will use `finos-legend` as the <model-name>.

## Deploy the Legend Bundle
When you deploy an application with Juju, the installation code in the charmed operator will run and set up all the resources and environmental variables needed for the application to run properly. In the case of this tutorial, we are deploying a *bundle*, which describes applications to be deployed and relationships between them.

Deploy the [finos-legend-bundle](https://github.com/finos/finos-legend-bundle) using 

TODO: Remove `--channel=edge` once `stable` is released. 
``` bash
juju deploy finos-legend-bundle --channel=edge
```

TODO: Temporary:
```
juju deploy nginx-ingress-integrator ingress-studio
juju deploy nginx-ingress-integrator ingress-sdlc
juju deploy nginx-ingress-integrator ingress-engine
  
juju relate finos-legend-engine-k8s ingress-engine
juju relate finos-legend-studio-k8s ingress-studio
juju relate finos-legend-sdlc-k8s ingress-sdlc
```
  
In another terminal, you can check the deployment status and the integration code running using `watch --color juju status --color`.

You'll notice that the Unit `finos-legend-gitlab-integrator-k8s/0` will get to `blocked` status; this is expected, as you'll need to [Setup and Configure GitLab](#Setup-and-Configure-GitLab).

## Setup and Configure GitLab
To run Legend, you need to either host a GitLab instance, or use GitLab.com; the type of installation really depends on user's requirements, there is a section in `DEPLOY_GITLAB.md` that talks about that (TODO).

If this is your first experience with Legend, we suggest you starting with GitLab.com as per the instructions below. If, instead, you're interested to test a Legend deployment with a local GitLab, please follow instructions on `DEPLOY_GITLAB.md`.

### GitLab.com
1. Create an account on gitlab.com
2. Create an application:
- Login GitLab.com, click your profile picture on the right upper corner and click "Preferences". 
- Click "Applications" on the left menu. 
- Create a new application with the following information:
  - name
  - enter a dummy URI in the redirect URL field (http://dummy will work for now) 
  - enable the following scopes: 
    - API
    - Open ID
    - Profile
- Click "Save Application".

3. Copy the <Application ID> and <Secret> information.

In your terminal, run the following command with the infromation you just copied
```bash
juju config finos-legend-gitlab-integrator-k8s client-id="<Application ID>" client-secret="<Secret>"
```

Retrieve the GitLab URI from the GitLab Integrator charm using  
```bash
juju show-unit finos-legend-gitlab-integrator-k8s/0 | grep legend-gitlab-redirect-uris
```
  
Go back to the "Applications" page on GitLab.com and edit the application you created. Replace the dummy URI with the URIs retrieved from the command above and save the application. It should look something like the following:
  
``` bash
http://10.1.122.204:6060/callback
http://10.1.122.248:7070/api/auth/callback
http://10.1.122.248:7070/api/pac4j/login/callback
http://10.1.122.255:8080/studio/log.in/callback
```  
  
TODO: Add `http://10.1.122.248:7070/api/pac4j/login/callback` to the charm's list of URLs  

## Monitor Juju status
Run `watch --color juju status --color` to see the applications reacting to the configuration change. As a result of this change, your FINOS Legend deployment should complete, the output should look like this.

```bash
$ watch --color juju status --color
Model  Controller  Cloud/Region        Version  SLA          Timestamp
lma    microk8s    microk8s/localhost  2.9.16   unsupported  14:19:18+01:00

App                                 Version  Status  Scale  Charm                               Store     Channel  Rev  OS          Address         Message
finos-legend-db-k8s                          active      1  finos-legend-db-k8s                 charmhub  edge      13  kubernetes  10.152.183.135
finos-legend-engine-k8s                      active      1  finos-legend-engine-k8s             charmhub  edge      13  kubernetes  10.152.183.11
finos-legend-gitlab-integrator-k8s           active      1  finos-legend-gitlab-integrator-k8s  charmhub  edge      24  kubernetes  10.152.183.102
finos-legend-sdlc-k8s                        active      1  finos-legend-sdlc-k8s               charmhub  edge      36  kubernetes  10.152.183.42
finos-legend-studio-k8s                      active      1  finos-legend-studio-k8s             charmhub  edge      14  kubernetes  10.152.183.141
mongodb-k8s                                  active      1  mongodb-k8s                         charmhub  edge       6  kubernetes  10.152.183.210

Unit                                   Workload  Agent  Address       Ports  Message
finos-legend-db-k8s/0*                 active    idle   10.1.252.83
finos-legend-engine-k8s/0*             active    idle   10.1.252.98
finos-legend-gitlab-integrator-k8s/0*  active    idle   10.1.252.71
finos-legend-sdlc-k8s/0*               active    idle   10.1.252.122
finos-legend-studio-k8s/0*             active    idle   10.1.252.78
mongodb-k8s/0*                         active    idle   10.1.252.116
```

Note that all `Workload`,  `Status` are `active`; you can now go ahead and [authorize the Legend user against GitLab](#Authorize-Legend-user-against-Gitlab).

## Authorize the Legend Gitlab Application

### A notes on your browser settings and cookies
The Legend stack rely on cookies to authenticate user across the applications. A recent change on how [samesite cookies](https://web.dev/samesite-cookies-explained/] are handled by Google Chrome and Mozilla Firefox can block the authentication process. To avoid problems with cookies, before starting the authentication process:
  
- make sure you use the private mode
- allow samesite cookies in your browser
- clear the browser's cache 
- [if you are using Firefox](https://support.mozilla.org/en-US/kb/enhanced-tracking-protection-firefox-desktop#w_how-to-tell-when-firefox-is-protecting-you), disable Enchanced Tracking Protection for the Studio and SDLC pages.
  
Using `juju status`, grab the `Address` of the `Unit` called `finos-legend-studio-k8s/0*` and point your browser to `http://<STUDIO-IP>:8080`; you should be redirected to a GitLab App authorization page.

![Authorize Legend Charm](./images/authorize-legend-charm.png)

Click on `Authorize` and you should be redirected back to Legend Studio. Note the `Unauthorized` pop-up dialog box in the lower right corner. ![Unauthorized User](./images/unauthorized-studio-charm.png) This is expected, since the Legend user needs to [authenticate against the SDLC component]((#authorize-the-user-to-use-legend)).

## Authenticate user on Legend SDLC
As before, using `juju status`, grab the `Address` of the `Unit` called `finos-legend-sdlc-k8s/0*` and point your browser to `http://<SDLC_IP>:7070/api/auth/authorize`; you should get redirected again to GitLab, as follows. Click on `Authorize` and you should be redirected to Legend Studio.

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
