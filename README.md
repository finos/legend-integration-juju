# Charmed FINOS Legend

Deploy the Legend stack to Kubernetes using Juju and charmed operators.

``` bash
juju deploy finos-legend-bundle
```

## Installation

Please refer to the [instructions page](legend-integration-juju/blob/main/INSTRUCTIONS.md) for a detailed guide on how to deploy the Legend stack on your Kubernetes cluster. If you don't have a cluster, we will show you how to get one with [MicroK8s](https://microk8s.io/). 

## Juju and Charmed Operators

The [Juju Charmed Operator Lifecycle Manager (OLM)](https://juju.is/docs/olm) is a hybrid-cloud application management and orchestration system for installation and day 2 operations. It helps deploy, configure, scale, integrate, maintain, and manage Kubernetes native, container-native and VM-native applications—and the relations between them.

A charmed operator (also known, more simply, as a “charm”) encapsulates a single application and all the code and know-how it takes to operate it, such as how to combine and work with other related applications or how to upgrade it. Charms are programmed to understand a single application, its operations, and its potential to communicate or integrate with other applications. A charm defines and enables the channels by which applications connect.

### Cloud vendor agnostic
The instructions cover the deployment of FINOS Legend charmed operators on a host PC along with a locally deployed Gitlab instance. However these charms can be deployed on any cloud and should be able to use any Gitlab instance with requisite permissions.

### Multi-hybrid cloud
The Legend applications stack was deployed as a *bundle* in the same cloud. [Juju](https://juju.is/) allows, however, for you to deploy each application on a different cloud and then integrate the stack across your estate.

### Offline installation
We assume your host had a functioning Internet connection. However it is also possible to [deploy charmed operators offline](https://juju.is/docs/olm/working-offline).


## Help and Support
- [Canonical Mattermost Forum](https://chat.charmhub.io/charmhub/channels/charmed-legend). Authors of the charmed operators discussed in this tutorial should be reachable here.
- [FINOS Legend Slack Channel](finos-lf.slack.com). FINOS developers should be reachable here.

