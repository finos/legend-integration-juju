# Deploy Legend on AWS EKS using Juju
This tutorial will cover how to use Juju and Charmed Operators to deploy an instance of the [FINOS Legend](https://www.finos.org/legend) stack on Amazon EKS.

### Audience
This document can be used for evaluation of the Legend stack running on EKS. Please contact FINOS if you would like to run Legend in a production enviroment.

### Prerequisites
This document assumes that you already have installed the Juju CLI, AWS CLI, ``kubectl``, and ``eksctl`` as described in th prerequisites section of [this page](https://juju.is/docs/olm/amazon-elastic-kubernetes-service-(amazon-eks)#heading--prerequisites).

## Configure aws CLI
Before starting, make sure you're connected to the right AWS account, by typing `aws configure`; if you already have a profile set for your installation, you can type `aws configure --profile <my-profile-name>`; profiles are configured in `~/.aws/credentials`.

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

The ``region`` and ``instanceType`` can be customized, but make sure that that ``instanceType`` is available in that ``region``.

Finally, you can create the cluster by running
``` bash
eksctl create cluster -f ./eks-cluster.yaml
```
The cluster creation can take up to 25 minutes to complete.

> ⚠️ If the previous command fails with: `Cannot create cluster 'legend-acct-juju' because us-east-1e, the targeted availability zone, does not currently have sufficient capacity to support the cluster....`, then:
> 1. Access CloudFormation Web Console and delete the stack; wait for completion
> 2. Try again
> If you still encounter issues, try using a different AWS region.

If you are using Kubernetes v1.23 or newer, the Amazon Elastic Block Store (Amazon EBS) Container Storage Interface (CSI) driver is needed to manage the Amazon EBS persistent volumes. After deploying the EKS cluster, you can check the Kubernetes cluster version by running:

```
eksctl get cluster --region CLUSTER-REGION --name CLUSTER-NAME
```

The EBS CSI driver requires an ``iamserviceaccount`` account. The script below will create the service account and will add the CSI driver as an addon, allowing it to be updatable through ``eksctl``. Make sure that the ``CLUSTER_NAME``, ``REGION_NAME``, are correct, and ``ROLE_NAME`` is unique.

```
CLUSTER_NAME="finos-legend"
REGION_NAME="eu-west-2"
ROLE_NAME="AmazonEKS_EBS_CSI_DriverRole-finos"

eksctl utils associate-iam-oidc-provider --region="$REGION_NAME" --cluster="$CLUSTER_NAME" --approve

eksctl create iamserviceaccount \
  --name ebs-csi-controller-sa \
  --namespace kube-system \
  --cluster "$CLUSTER_NAME" \
  --region "$REGION_NAME" \
  --attach-policy-arn arn:aws:iam::aws:policy/service-role/AmazonEBSCSIDriverPolicy \
  --approve \
  --role-only \
  --role-name "${ROLE_NAME}"

# get the role ARN by using jq. Alternatively, the ARN can be fetched manually without using jq.
ARN=`eksctl get iamserviceaccount -o json --region "$REGION_NAME" --cluster "$CLUSTER_NAME" ebs-csi-controller-sa  | jq '.[0].status.roleARN'`

eksctl create addon --name aws-ebs-csi-driver --region "$REGION_NAME" --cluster "${CLUSTER_NAME}" --service-account-role-arn $ARN --force
```

You can read [here](https://docs.aws.amazon.com/eks/latest/userguide/ebs-csi.html) for more information about the EBS CSI driver and alternative methods to deploy it.

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
To install Juju, you can follow [the instructions in the docs](https://juju.is/docs/olm/installing-juju) or simply install a Juju with the command line `sudo snap install juju --classic`; on MacOS, you can use brew with `brew install juju`; run `juju status` to check if everything is up. Make sure that the version is `3.1.0` or higher, by running the command `juju version`.

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

The above command will deploy the latest application bundle published. You can deploy a specific version based on a [FINOS Legend release](https://github.com/finos/legend) by its year and month (newer than 2022.04.01):

```bash
juju deploy finos-legend-bundle --trust --channel=2022-04/edge
```

In another terminal window, you can see the applications being deployed and the integration code running
``` bash
watch --color juju status --color
```
You'll notice that the Unit `gitlab-integrator/0` will get to `blocked` status; this is expected, as you'll need to [Setup and Configure GitLab](#Setup-and-Configure-GitLab).

## Configure a hostname for FINOS Legend

In order for the FINOS Legend deployment to be publicly accessible, you are going to need a publicly available DNS hostname associated with the deployment. There are [alternative connection methods](#Alternative-methods-for-accessing-Legend-Studio-dashboard), but they should only be used for testing purposes and it's not recommended in production environments.

After obtaining your hostname, the Legend applications will have to be configured to respond to that hostname (all 3 Legend applications can be configured to have the same hostname):

```bash
HOST_NAME="your.hostname"
juju config legend-studio external-hostname="$HOST_NAME"
juju config legend-sdlc external-hostname="$HOST_NAME"
juju config legend-engine external-hostname="$HOST_NAME"
```

**NOTE**: If the hostname has been changed, you may need a new [TLS Certificate](#TLS-Configuration) for the new hostname, and the [gitlab.com's Callback URIs](#Gitlab.com-Application-setup) will have to be updated as well.

## TLS Configuration

In order to access FINOS Legend securely through HTTPS, you are going to need a TLS certificate. The certificate will depend on the [service hostname](#Accessing-the-Legend-Studio-dashboard) you are going to use for accessing Legend. Keep in mind that this will configure Ingress-level TLS termination, meaning that any and all requests will be sent to Legend encrypted until it reaches the Kubernetes Ingress. This will also automatically redirect any HTTP request to HTTPS, which means that you need to make sure that TLS termination does not occur at any point before reaching the NGINX Ingress Controller (for example, if you're using Cloudflare with Proxy enabled, make sure that the SSL/ TLS configuration is set to ``Full`` or ``Full (strict)``, otherwise the requests will be infinitely redirected).

Note that this **will not** work if you're using the Amazon Load Balancer hostname, as it is simply too long and cannot fit in the certificate Common Name (CN).

If you already have a certificate created, you simply have to register it into Kubernetes and configure Legend to use it. For more information on how to use it, see the [Enabling HTTPS](#Enabling-HTTPS) section.

#### Creating a Certificate

If you do not have a certificate yet, you could generate your own self-signed certificate instead. However, using a self-signed certificate will cause your browser to issue a Warning when accessing Legend because the certificate is not verified by a trusted Certificate Authority (CA), but you can still access Legend.

However, it is recommended to use TLS certificates verified by a trusted CA. In this case, The Kubernetes cluster will have to be reachable through a DNS hostname that you own because the CA will attempt to verify your ownership of that hostname through its challenges. If the challenges are not passed, then you won't have CA-verified certificates.

Below is a guide on how to obtain a TLS certificate approved by Let's Encrypt. We are going to use the ``certbot-k8s`` charm and the already deployed ``nginx-ingress-integrator`` charm in order to obtain it by solving an HTTP challenge. The ``certbot-k8s`` will setup an Ingress route for the Let's Encrypt HTTP Challenge and verifies that the is valid.

```bash
juju deploy --trust certbot-k8s --channel=edge
juju relate certbot-k8s legend-ingress
juju config legend-ingress rewrite-enabled=false

HOST_NAME="your-dns-resolvable-hostname"
OWN_EMAIL="your-email"
juju config certbot-k8s agree-tos=true email="$OWN_EMAIL" service-hostname="$HOST_NAME"
```

The charm should automatically generate the certificate and register it into Kubernetes for us to use. Its progress can be tracked by running ``juju status``. Once it's ready, running the following command should give you the Secret name; make sure that your DNS configuration is in place, otherwise the following command will fail.

```bash
juju run-action certbot-k8s/0 get-secret-name --wait
```

**_NOTE:_** If you're using juju v3.0.0 or newer, use ``juju run certbot-k8s/0 get-secret-name --wait=2m`` instead.

#### Enabling HTTPS

The step above has generated a certificate for us and registered it as a Kubernetes Secret. If you have your own certificate and you've skipped the step above, you can register it into the Kubernetes cluster using the following command:

```bash
SECRET_NAME="legend-tls"
kubectl create secret -n finos-legend tls "$SECRET_NAME" --cert=fullchain.pem --key=privkey.pem
```

We can now configure the ``nginx-ingress-integrator`` charm to use the certificate, and enable TLS on the Legend Applications as well:

```bash
juju config legend-ingress tls-secret-name="$SECRET_NAME"
juju config legend-engine enable-tls=true
juju config legend-sdlc enable-tls=true
juju config legend-studio enable-tls=true
```

After the config options above have been updated, the [gitlab.com's Callback URIs](#Gitlab.com-Application-setup) will have to be updated as well.

**NOTE**: If the hostname has to be changed, and you're [generating a new TLS certificate](#TLS-Configuration) for the new hostname, make sure you reset the ``tls-secret-name`` on the ``legend-ingress`` charm first; configuration options set on that charms apply to all of it relations. If not, ``certbot-k8s`` charm will encounter some issues when trying to solve the Let's Encrypt HTTP Challenge through HTTPS (there will be a mismatch between the TLS certficate currently in use and the new hostname). To do this, run:

```bash
juju config legend-ingress tls-secret-name=""
```

Alternatively, the ``certbot-k8s`` charm could be related to a different ``nginx-ingress-integrator`` charm.

## Gitlab.com Application setup
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
  - Enter the following `Redirect URIs`:
    ``` bash
    http://legend-host/studio/log.in/callback
    http://legend-host/callback
    http://legend-host/api/auth/callback
    http://legend-host/api/pac4j/login/callback
    ```
  - enable the following scopes:
    - API
    - Open ID
    - Profile
3. Click `Save Application`.

**Note**: The Redirect URIs above are based on the ``external-hostname`` configured for the FINOS Legend applications, the value of which will depend on your requirements. For more details, see the [Configure a hostname for FINOS Legend](#Configure-a-hostname-for-FINOS-Legend) section or the [alternative connection methods](#Alternative-methods-for-accessing-Legend-Studio-dashboard) for testing purposes. After the config option has been changed, the gitlab.com Callback URIs will have to be updated; you can get them by running:

```bash
juju run-action gitlab-integrator/0 get-redirect-uris --wait
```

**_NOTE:_** If you're using juju v3.0.0 or newer, use ``juju run gitlab-integrator/0 get-redirect-uris --wait=2m`` instead.

On the following page, make a note of the `Application ID` and `Secret`.

#### 2. Pass the application details to the Legend stack

In your terminal run the following command to pass the `Application ID` and `Secret` to the Legend stack
``` bash
app_id="<Application ID>"
secret_id="<Secret ID>"
juju config gitlab-integrator gitlab-client-id="$app_id" gitlab-client-secret="$secret_id"
```

Run `watch --color juju status --color` to see the applications reacting to the configuration change. As a result of this change, your FINOS Legend deployment should complete, the output should look like this:

![image](https://user-images.githubusercontent.com/5586487/152023319-47887089-310b-434b-8c3c-cf8c913fbc99.png)

### Authorize the GitLab user and application

> ⚠️ Browser settings and cookies
>
> The Legend stack rely on cookies to authenticate user across the applications. A recent change on how [samesite cookies](https://web.dev/samesite-cookies-explained/] are handled by Google Chrome and Mozilla Firefox can block the authentication process. To avoid problems with cookies, before starting the authentication process:
>
> - make sure you use the private mode
> - allow samesite cookies in your browser
> - clear the browser's cache
> - [if you are using Firefox](https://support.mozilla.org/en-US/kb/enhanced-tracking-protection-firefox-desktop#w_how-to-tell-when-firefox-is-protecting-you), disable Enchanced Tracking Protection for the Studio and SDLC pages.

The URLs below assumes the configured ``external-hostname`` is ``legend-host``. If you've used a different hostname, update accordingly. In your browser, enter the following URLs to authorize the user and applications:

[http://legend-host/api/auth/authorize](http://legend-host/api/auth/authorize) - Click `Authorize`. You should see the text `Success`.

[http://legend-host/engine](http://legend-engine/engine) - Click `Authorize`. You should be redirected to Legend Engine.

[http://legend-host](http://legend-host) - Click `Authorize`. You should be redirected to Legend Studio.

If the process was sucessful, you will be able to see the Legend Studio dashboard on [http://legend-host/studio/-/setup](http://legend-host/studio/-/setup)! 🎉

## Destroy setup

To remove all deployed Legend applications, remove any data and reset Juju to the state it was before Legend was deployed, you can destroy the controller (created during the bootstrapping process). This is a non-reversible operation--all your data created in studio will be lost!

``` bash
juju destroy-controller -y --destroy-all-models --destroy-storage finos-legend
```

If you want to destroy the entire EKS Cluster, you can run:

``` bash
# We also need to delete the nginx-ingress ingress, where the Amazon Load Balancer is allocated.
# If we don't remove it, it may be leaked into AWS, resulting in additional costs.
kubectl delete ns/nginx-ingress
eksctl delete cluster finos-legend --region eu-west-2
```

If you've destroyed the EKS Cluster before destroying the Juju Controller on it, you can simply run:

``` bash
juju unregister finos-legend
```

# Appendix

## Alternative methods for accessing Legend Studio dashboard

If you do not have a publicly available hostname, FINOS Legend can still be accessed in a few different ways. These ways are **not** recommended for production environments, but only for testing purposes. Choose one that best fits your needs. In all scenarios, we will have to configure the FINOS Legend applications to be served under the same hostname. For example, this would be needed for [direct connection through /etc/hosts](#Direct-connection-through-/etc/hosts):

```
HOST_NAME="legend-host"
juju config legend-studio external-hostname="$HOST_NAME"
juju config legend-sdlc external-hostname="$HOST_NAME"
juju config legend-engine external-hostname="$HOST_NAME"
```

The ``external-hostname`` will depend on the preffered method below. In any case, after the config option has been updated, the [gitlab.com's Callback URIs](#Gitlab.com-Application-setup) will have to be updated as well.

### Direct connection through /etc/hosts

These instructions need to be replicated in all systems acessing the Legend applications. It's primarily recommended for testing purposes. This method can have HTTPS enabled through self-signed certificates.

After the configuration above has been set, in order to access FINOS Legend through this method, we will need to configure our own ``/etc/hosts`` (or ``C:\Windows\System32\drivers\etc\hosts`` file on Windows) to point to it. We will need to know the IP through which we can access our applications:

``` bash
ALB_FQDN="$(kubectl -n nginx-ingress get svc nginx-ingress -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')"
ALB_IP="$(dig +short $ALB_FQDN | tail -n 1)"
echo $ALB_IP
```

Using the `IP` retrieved with the command above, add the follwoing lines to your `/etc/hosts` file:

```
the_ip_above      the_configured_hostname
```

The line above can de added directly by using this command:

```bash
echo "$ALB_IP $HOST_NAME" | sudo tee -a /etc/hosts
```

The following command should return without errors
``` bash
curl http://$HOST_NAME
```
You can now proceed to [Authenticate the user and the application](#authenticate_the_gitlab_user_and_application).

### Using The AWS Load Balancer

FINOS Legend is accessible through the configured AWS Load Balancer. Keep in mind that the Load Balancer name is too long to generate a certificate for. The Legend applications simply have to be configured to respond to its name:

```bash
ALB_FQDN="$(kubectl -n nginx-ingress get svc nginx-ingress -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')"
juju config legend-studio external-hostname="$ALB_FQDN"
juju config legend-sdlc external-hostname="$ALB_FQDN"
juju config legend-engine external-hostname="$ALB_FQDN"
```

You can now proceed to [Authenticate the user and the application](#authenticate_the_gitlab_user_and_application).
