# Deploy Legend on AWS EKS using Juju
This tutorial will cover how to use Juju and Charmed Operators to deploy an instance of the [FINOS Legend](https://www.finos.org/legend) stack on your local workstation, using [AWS EKS](https://aws.amazon.com/eks/).

## Prerequisites
These steps may be carried out on any Linux distribution that has a [Snap](https://snapcraft.io/) package manager installed. The steps in this tutorial have been tested on [Ubuntu 20.04](https://releases.ubuntu.com/focal/). 

Follow prerequisite instructions on https://juju.is/docs/olm/amazon-elastic-kubernetes-service-(amazon-eks)

Make sure to use that:
1. Juju v2.9.25 or higher, check with `juju version`. Run `brew update` before installing it.
2. The IAM user has all permissions to run EKS operations; for testing purposes, grant Admin rights, then delete the key

## Create an AWS cluster (without a nodegroup)
```
eksctl create cluster \
--name legend-acct-juju \
--region us-west-2 \
--node-type ”t2.large” \
--without-nodegroup
```

If the previous command fails with: `Cannot create cluster 'legend-acct-juju' because us-east-1e, the targeted availability zone, does not currently have sufficient capacity to support the cluster....`, then:
1. Access CloudFormation Web Console and delete the stack; wait for completion
2. Try again

If you still encounter issues, try using a different AWS region.

## Create the nodegroup
```
eksctl create nodegroup \
--cluster legend-acct-juju \
--region us-west-2 \
--node-type t2.large \
--nodes 2
```

Make sure that the `node-type` is available in your region; access the AWS EC2 Web Console and Launch Instance; use the default AMI, then scroll the instance types and choose the one to use for the eksctl command. Now you can click on Cancel to quit the instance launch panel.

## Connect `kubectl` with the running cluster
```
aws eks --region us-west-2 update-kubeconfig --name legend-acct-juju

kubectl get nodes
# Should return 2 nodes
```

## Run Juju commands to deploy Legend
```
juju add-k8s legend-acct-juju
# k8s substrate "ec2/us-west-2" added as cloud "legend-acct-juju".

juju bootstrap legend-acct-juju
# Bootstrap complete, controller "legend-acct-juju-us-west-2" is now available in namespace "controller-legend-acct-juju-us-west-2"

juju add-model finos-legend-bundle
# Added 'finos-legend-bundle' model on legend-acct-juju/us-west-2 with credential 'legend-acct-juju' for user 'admin'

juju deploy finos-legend-bundle
# Deploy of bundle completed.
```

---

To cleanup:
```
eksctl delete cluster legend-acct-juju --region us-west-2
juju unregister legend-acct-juju-us-west-2
```
