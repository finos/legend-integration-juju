[![FINOS - Incubating](https://cdn.jsdelivr.net/gh/finos/contrib-toolbox@master/images/badge-incubating.svg)](https://finosfoundation.atlassian.net/wiki/display/FINOS/Incubating)

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

### Related Repositories
- [finos-legend-gitlab-integrator](https://github.com/canonical/finos-legend-gitlab-integrator)
- [finos-legend-db-operator](https://github.com/canonical/finos-legend-db-operator)
- [finos-legend-libs](https://github.com/canonical/finos-legend-libs)
- [finos-legend-studio-operator](https://github.com/canonical/finos-legend-studio-operator)
- [finos-legend-sdlc-server-operator](https://github.com/canonical/finos-legend-sdlc-server-operator)
- [finos-legend-engine-server-operator](https://github.com/canonical/finos-legend-engine-server-operator)
- [https://github.com/canonical/finos-legend-bundle](https://github.com/canonical/finos-legend-bundle)

## Help and Support
Feel free to [create an issue] or submit a Pull Request to this repository in order to contribute; make sure to read the Legend [Contribution Guide](https://github.com/finos/legend/blob/master/CONTRIBUTING.md) first.

You can also use chat to the contributors to this Legend intergation via the [FINOS Legend Slack Channel](finos-lf.slack.com) or the [Canonical Mattermost Forum](https://chat.charmhub.io/charmhub/channels/charmed-legend).

## Roadmap

Visit our [roadmap](https://github.com/finos/legend#roadmap) to know more about the upcoming features.

## Contributing

Visit Legend [Contribution Guide](https://github.com/finos/legend/blob/master/CONTRIBUTING.md) to learn how to contribute to Legend.

## License

Copyright (c) 2021-present, Canonical

Distributed under the [Apache License, Version 2.0](http://www.apache.org/licenses/LICENSE-2.0).

SPDX-License-Identifier: [Apache-2.0](https://spdx.org/licenses/Apache-2.0)


