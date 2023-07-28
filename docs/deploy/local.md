# Deploy Legend locally (MicroK8s) using Juju 
This tutorial will cover how to use Juju and Charmed Operators to deploy an instance of the [FINOS Legend](https://www.finos.org/legend) stack on your local workstation, using [Microk8s](https://microk8s.io/). 

## Prerequisites
These steps may be carried out on any Linux distribution that has a [Snap](https://snapcraft.io/) package manager installed. The steps in this tutorial have been tested on [Ubuntu 20.04](https://releases.ubuntu.com/focal/). 

## Install Microk8s
[Microk8s](https://microk8s.io/) is a *micro* Kubernetes distribution that runs locally; you can install it on Linux running the following commands:
```bash
sudo snap install microk8s --channel 1.27-strict/stable
sudo snap alias microk8s.kubectl kubectl
source ~/.profile
sudo usermod -a -G snap_microk8s $USER
sudo chown -f -R $USER ~/.kube
newgrp snap_microk8s
```

> On Mac, assuming you have XCode, Python 3.9 (with brew, you can run `brew link --overwrite python@3.9`) and [HomeBrew](brew.sh) installed, you can run 
> 
> ``` bash
> brew install multipass
> brew install ubuntu/microk8s/microk8s
> microk8s install
> ```

To configure and start `microk8s`, simply run:

```bash
sudo microk8s enable dns hostpath-storage ingress
microk8s status --wait-ready
```

Below is the expected output; before moving forward

> ⚠️ Wait for services
> 
> Make sure that `dns`, `ingress` and `storage` are listed as `enabled`. This can take a couple of minutes. 

``` bash
microk8s is running
high-availability: no
  datastore master nodes: 127.0.0.1:19001
  datastore standby nodes: none
addons:
  enabled:
    dns                  # (core) CoreDNS
    ha-cluster           # (core) Configure high availability on the current node
    helm                 # (core) Helm - the package manager for Kubernetes
    helm3                # (core) Helm 3 - the package manager for Kubernetes
    hostpath-storage     # (core) Storage class; allocates storage from host directory
    ingress              # (core) Ingress controller for external access
    storage              # (core) Alias to hostpath-storage add-on, deprecated
```

## Install Juju
To install Juju, you can follow [the instructions in the docs](https://juju.is/docs/juju/get-started-with-juju) or simply install a Juju with the command line `sudo snap install juju`; on MacOS, you can use brew with `brew install juju` (this will run an upgrade, if the `juju` brew formula is already installed). Since the Juju package is strictly confined, you also need to manually create a path `sudo mkdir -p ~/.local/share/juju` and then change the owner of the juju directory: `sudo chown -R $USER .local/share/juju/`; run `juju status` to check if everything is up. This guide was written using Juju 3.1.5. 

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

The above command will deploy the latest application bundle published. You can deploy a specific version based on a [FINOS Legend release](https://github.com/finos/legend) by its year and month (newer than 2022.04.01):

```bash
juju deploy finos-legend-bundle --trust --channel=2022-04/edge
```

In another terminal window, you can see the applications being deployed and the integration code running
``` bash
watch --color juju status --color
```

You'll notice that the Unit `gitlab-integrator/0` will get to `blocked` status; this is expected, as you'll need to [Setup and Configure GitLab](#Setup-and-Configure-GitLab).

## Configure a hostname for your Legend Applications

By default, the hostnames used by the Legend Applications will be their Juju deployed application names (for example, for Legend Studio it will be ``legend-studio``). However, you can configure them to be served under the same hostname by running:

```bash
my_hostname="legend-host"
juju config legend-engine external-hostname=$my_hostname
juju config legend-sdlc external-hostname=$my_hostname
juju config legend-studio external-hostname=$my_hostname
```

## Setup and Configure GitLab
To run Legend, you need to either run a GitLab instance somewhere, or use GitLab.com; the type of installation really depends on user's requirements, there is a secion in `DEPLOY_GITLAB.md` that talks about that (TODO).

If this is your first experience with Legend, we suggest you start with GitLab.com as per the instructions below. If, instead, you're interested to test a Legend deployment with a local GitLab, please follow instructions on `DEPLOY_GITLAB.md`.

If you are using gitlab.com, follow the following three steps.

#### 1. Create a GitLab Application

1. Create an account on gitlab.com
2. Create an application:
- Login to GitLab.com, click your profile picture on the right upper corner and click "Preferences". 
- Click "Applications" on the left menu. 
- Create a new application with the following information:
  - name
  - Check the `Confidential` checkbox 
  - Enter the following redirect URIs:
    ``` bash
    http://legend-host/engine/callback
    http://legend-host/api/auth/callback
    http://legend-host/api/pac4j/login/callback
    http://legend-host/studio/log.in/callback
    ``` 
  - enable the following scopes: 
    - API
    - Open ID
    - Profile
- Click "Save Application".

On the following page, make a note of the `Application ID` and `Secret`. 

#### 2. Pass the application details to the Legend stack

In your terminal run the following command to pass the `<Application ID>` and `<Secret ID>` to the Legend stack
``` bash
app_id="<Application ID>"
secret_id="<Secret ID>"
juju config gitlab-integrator gitlab-client-id="$app_id" gitlab-client-secret="$secret_id"
```

#### 3. Update the GitLab application URIs

The command below will retrieve the GitLab URIs from the GitLab Integrator charm
``` bash
juju run-action gitlab-integrator/0 get-redirect-uris --wait
```

**_NOTE:_** If you're using juju v3.0.0 or newer, use ``juju run gitlab-integrator/0 get-redirect-uris --wait=2m`` instead.

Go back to the "Applications" page on GitLab.com and edit the application you created. Replace the dummy URI with the URIs retrieved from the command above and save the application. 

Run `watch --color juju status --color` to see the applications reacting to the configuration change. As a result of this change, your FINOS Legend deployment should complete.

## Accessing the Legend Studio dashboard

> ⚠️ Browser settings and cookies
> 
> The Legend stack rely on cookies to authenticate user across the applications. A recent change on how [samesite cookies](https://web.dev/samesite-cookies-explained/) are handled by Google Chrome and Mozilla Firefox can block the authentication process. To avoid problems with cookies, before starting the authentication process:
>  
> - make sure you use the private mode
> - allow samesite cookies in your browser
> - clear the browser's cache 
> - [if you are using Firefox](https://support.mozilla.org/en-US/kb/enhanced-tracking-protection-firefox-desktop#w_how-to-tell-when-firefox-is-protecting-you), disable Enchanced Tracking Protection for the Studio and SDLC pages.

In order to access the local FINOS Legend Application, the following lines should be added to the `/etc/hosts` file:

``` bash
127.0.1.1 legend-host
```
Adding this line will allow you to access Legend directly in your browser through the user-friendly name. `legend-host` is the `external-hostname` [configured for the Legend Applications](#Configure-a-hostname-for-your-Legend-Applications). If you've provided a different name, use that instead.

> ⚠️ MacOS systems
> 
> In MacOS systems, MicroK8s runs in a Multipass VM. To have access to the services running innside the MicroK8s cluster, you will have to use the Multipass IP adress in your `/etc/hosts` file. 
> Run 
> ```bash
> ~ » multipass list
>  Name                    State             IPv4             Image
>  microk8s-vm             Running           <IP>             Ubuntu 18.04 LTS
>                                            10.1.255.65
> ```
> And add `<IP>` to your `/etc/hosts`:
> ``` bash
> <IP> legend-host
> ```

Adding those lines will allow you to access Legend directly in your browser through those user-friendly names.

### Authorize the GitLab user and application

Point your browser to `http://legend-host/api/auth/authorize`; Click on `Authorize` and you should see the text `Authorized`. 

Similarly, point your browser to `http://legend-host/engine`; Click on `Authorize`. 

Finally, point your browser to `http://legend-host/studio/-/setup`. Click on `Authorize`. 

If the process was sucessful, you will be able to see the Legend Studio dashboard on `http://legend-host/studio`! 🎉


## Destroy setup
To remove all deployed Legend applications, remove any data and reset Juju to the state it was before Legend was deployed, you can destroy the controller (created during the bootstrapping process). **This is a non-reversible operation - all your data created in studio will be lost!** If you would like to learn more about other less-destructive removal option, please check [this page](https://juju.is/docs/olm/removing-things). 

```bash
juju destroy-controller -y --destroy-all-models --destroy-storage finos-legend-controller
```

To also remove Juju and MicroK8s, you can run:
``` bash
sudo snap remove juju --purge
sudo snap remove microk8s --purge
```

On MacOS, use `brew remove juju ubuntu/microk8s/microk8s`.

# Other Operations

## Updating Legend applications

You can update and upgrade a charm automatically to the latest version by running ``juju upgrade-charm app-name --channel=edge``. For example, you can upgrade the Legend Studio charm by running: ``juju refresh legend-studio --channel=edge``.

However, if you want want to use a different Container Image than the charm has been published with, you will have to remove the application first, and add it manually, and then add the necessary relationships. You can see the currently created relations by running ``juju status --relations``.

Example: Let's say that you want to redeploy the Legend Studio with the latest ``finos/legend-studio:2.6.0`` image. For this, we need first remove the ``legend-studio`` charm:

``
juju remove-application legend-studio
``

You can see the status by running ``juju status``. After it was removed, redeploy it again, and also specifying the new Container Image as a resource:

``
juju deploy finos-legend-studio-k8s legend-studio --channel=edge --resource studio-image=finos/legend-studio:2.6.0
``

Afterwards, we need to add the necessary relations:

``` bash
juju relate legend-studio legend-db
juju relate legend-studio legend-engine
juju relate legend-studio legend-sdlc
juju relate legend-studio gitlab-integrator
juju relate legend-studio ingress-studio
```
