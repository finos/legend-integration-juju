# Deploy Legend on AWS EKS using Juju
This tutorial will cover how to use Juju and Charmed Operators to deploy an instance of the [FINOS Legend](https://www.finos.org/legend) stack on Amazon EKS.

### Audience
This document can be used for evaluation of the Legend stack running on EKS. Please contact FINOS if you would like to run Legend in a production enviroment.  

### Prerequisites 
This document assumes that you already have installed the Juju CLI, AWS CLI, ``kubectl``, and ``eksctl`` as described in th prerequisites section of [this page](https://juju.is/docs/olm/amazon-elastic-kubernetes-service-(amazon-eks)#heading--prerequisites). 

## Create and setup an AWS cluster

### Create the cluster
The command below will create an EKS cluster description file.
``` yaml
cat << EOF > eks-cluster.yaml
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig

metadata:
  name: finos-legend
  region: eu-west-2

nodeGroups:
  - name: ng-1
    instanceType: m5.large
    desiredCapacity: 3
    volumeSize: 80
EOF
```

Finally, you can create the cluster by running
``` bash
eksctl create cluster -f ./eks-cluster.yaml
```
The cluster creation can take up to 25 minutes to complete. 

> ⚠️ If the previous command fails with: `Cannot create cluster 'legend-acct-juju' because us-east-1e, the targeted availability zone, does not currently have sufficient capacity to support the cluster....`, then:
> 1. Access CloudFormation Web Console and delete the stack; wait for completion
> 2. Try again
> If you still encounter issues, try using a different AWS region.


After the EKS cluster was deployed, we need to generate our `kubeconfig` file, which contains the necessary details to connect to our newly created cluster. To do so, run the following:

``` bash
eksctl utils write-kubeconfig --cluster finos-legend --region eu-west-2
kubectl config rename-context $(kubectl config current-context) finos-legend
```

### Set up Ingress

To make sure the Legend application are accessible outside the cluster, we need to install the Nginx Ingress resources and set up an AWS LoadBalancer. For this, run:
```
RAW_INGRESS_URL="https://raw.githubusercontent.com/finos/legend-integration-juju/92284746aec10e1d74bfb0ac762112da5586b288/INSTRUCTIONS/AWS%20EKS"
kubectl apply -f "$RAW_INGRESS_URL/eks_ingress.yaml"
sleep 10
ALB_FQDN="$(kubectl -n nginx-ingress get svc nginx-ingress -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')"
```

## Install Juju
To install Juju, you can follow [the instructions in the docs](https://juju.is/docs/olm/installing-juju) or simply install a Juju with the command line `sudo snap install juju --classic`; on MacOS, you can use brew with `brew install juju`; run `juju status` to check if everything is up.

If you're interested to know how to run Juju on your cloud of choice, checkout [the official docs](https://juju.is/docs/olm/clouds); you can always run `juju clouds` to check your configured clouds. In the instructions below, we will always use `microk8s`, but you can replace it with the name of the cloud you're using.



## Bootstrap Juju

In Juju terms, "bootstrap" means "create a Juju controller", which is the part of Juju that runs in your cluster and controls the applications. You can bootstrap Juju to the EKS cluster by running
``` bash
juju bootstrap finos-legend
```
Running `juju status` will now show the Juju Controller in your cloud.

### Create a Model
[Juju models](https://juju.is/docs/olm/models) are a logical grouping of applications and infrastructure that work together to deliver a service or product. In Kubernetes terms, models are effectively namespaces. Models are fundamental concepts in Juju and implement service isolation, access control, repeatability and boundaries.

You can add a new model with

``` bash
juju add-model finos-legend
```

## Deploy the Legend Bundle
When you deploy an application with Juju, the installation code in the charmed operator will run and set up all the resources and environmental variables needed for the application to run properly. In the case of this tutorial, we are deploying a bundle -- which not only describes a set of applications we want deployed, but also the relations between them.

Deploy the finos-legend-bundle in the finos-legend model using the command line :
``` bash
juju deploy finos-legend-bundle --trust --channel=edge
```
The `--trust` will allow the Nginx charm to have access to the EKS cluster. 

In another terminal window, you can see the applications being deployed and the integration code running
``` bash
watch --color juju status --color
```
You'll notice that the Unit `gitlab-integrator/0` will get to `blocked` status; this is expected, as you'll need to [Setup and Configure GitLab](#Setup-and-Configure-GitLab).

## Gitlab.com Application setup
To run Legend, you need to either run a GitLab instance somewhere, or use GitLab.com; the type of installation really depends on user's requirements, there is a secion in `DEPLOY_GITLAB.md` that talks about that (TODO).

If this is your first experience with Legend, we suggest you starting with GitLab.com as per the instructions below. If, instead, you're interested to test a Legend deployment with a local GitLab, please follow instructions on `DEPLOY_GITLAB.md`.

#### 1. Create a GitLab Application

1. Create an account on gitlab.com
2. Create an application:
- Login GitLab.com, click your profile picture on the right upper corner and click "Preferences". 
- Click "Applications" on the left menu. 
- Create a new application with the following information:
  - name
  - Check the `Confidential` checkbox 
  - Enter the following `Redirect URIs`:
    ``` bash
    http://legend-studio/studio/log.in/callback
    http://legend-engine/callback
    http://legend-sdlc/api/auth/callback
    http://legend-sdlc/api/pac4j/login/callback
    ```
  - enable the following scopes: 
    - API
    - Open ID
    - Profile
3. Click `Save Application`.

Note that the Redirect URIs will have to be replaced manually if you decide to use [multiple Amazon Load Balancers](#using-multiple-aws-load-balancers) or  configure them with external host names.

On the following page, make a note of the `Application ID` and `Secret`. 

#### 2. Pass the application details to the Legend stack

In your terminal run the following command to pass the `Application ID` and `Secret` to the Legend stack
``` bash
juju config gitlab-integrator gitlab-client-id="<Application ID>" gitlab-client-secret="<Secret Id> "
```

Run `watch --color juju status --color` to see the applications reacting to the configuration change. As a result of this change, your FINOS Legend deployment should complete, the output should look like this:

![image](https://user-images.githubusercontent.com/5586487/152023319-47887089-310b-434b-8c3c-cf8c913fbc99.png)


## Accessing the Legend Studio dashboard

FINOS Legend can be accessed using two different methods. Choose one that best fits your needs:

- [Direct connection through /etc/hosts](#direct_connection_through_/etc/hosts) (recommended for testing purposes)
- [Bring your own DNS hostnames](#bring_your_own_dns_hostnames)

### Direct connection through /etc/hosts

These instructions need to be replicated in all systems acessing the Legend applications. It's primarily recommended for testing purposes.

In order to access FINOS Legend through this method, we will need to configure the local system's `/etc/hosts` (or ``C:\Windows\System32\drivers\etc\hosts`` file on Windows). You will need to know the `IP` through which we can access the applications:

``` bash
dig +short $ALB_FQDN | tail -n 1
```

Using the `IP` retrieve with the command above, add the follwoing lines to your `/etc/hosts` file

```
<IP>      legend-studio
<IP>      legend-sdlc
<IP>      legend-engine
```

The following command should return without errors
``` bash
curl -H "Host: legend-studio" the_ip_above
```
You can now proceed to [Authenticate the user and the application](#authenticate_the_user_and_the_application).

### Bring your own DNS hostnames

You can use your own DNS hostnames to access the FINOS Legend Applications. The applications will have to be configured to respond to those names:

```
juju config legend-studio external-hostname="studio.your.domainname"
juju config legend-sdlc external-hostname="sdlc.your.domainname"
juju config legend-engine external-hostname="engine.your.domainname"
```
You can now proceed to [Authenticate the user and the application](#authenticate_the_user_and_the_application).

### Authorize the GitLab user and application

> ⚠️ Browser settings and cookies
> 
> The Legend stack rely on cookies to authenticate user across the applications. A recent change on how [samesite cookies](https://web.dev/samesite-cookies-explained/] are handled by Google Chrome and Mozilla Firefox can block the authentication process. To avoid problems with cookies, before starting the authentication process:
>  
> - make sure you use the private mode
> - allow samesite cookies in your browser
> - clear the browser's cache 
> - [if you are using Firefox](https://support.mozilla.org/en-US/kb/enhanced-tracking-protection-firefox-desktop#w_how-to-tell-when-firefox-is-protecting-you), disable Enchanced Tracking Protection for the Studio and SDLC pages.

In your browser, enter the following URLs to authorize the user and applications: 

[http://legend-sdlc/api/auth/authorize](http://legend-sdlc/api/auth/authorize) - Click `Authorize`. You should see the text `Success`. 

[http://legend-engine](http://legend-studio) - Click `Authorize`. You should be redirected to Legend Studio.

If the process was sucessful, you will be able to see the Legend Studio dashboard on [http://legend-studio/studio/-/setup](http://legend-studio/studio/-/setup)! 🎉

## Destroy setup

To remove all deployed Legend applications, remove any data and reset Juju to the state it was before Legend was deployed, you can destroy the controller (created during the bootstrapping process). This is a non-reversible operation--all your data created in studio will be lost!

``` bash
juju destroy-controller -y --destroy-all-models --destroy-storage finos-legend
```

If you want to destroy the entire EKS Cluster, you can run:

``` bash
eksctl delete cluster finos-legend --region eu-west-2
```

If you've destroyed the EKS Cluster before destroying the Juju Controller on it, you can simply run:

``` bash
juju unregister finos-legend
```
