[![FINOS - Incubating](https://cdn.jsdelivr.net/gh/finos/contrib-toolbox@master/images/badge-incubating.svg)](https://finosfoundation.atlassian.net/wiki/display/FINOS/Incubating)

## Juju and Charmed Operators

The [Juju Charmed Operator Lifecycle Manager (OLM)](https://juju.is/docs/olm) is a hybrid-cloud application management and orchestration system for installation and day 2 operations. It helps deploy, configure, scale, integrate, maintain, and manage Kubernetes native, container-native and VM-native applications—and the relations between them.

A charmed operator (also known, more simply, as a “charm”) encapsulates a single application and all the code and know-how it takes to operate it, such as how to combine and work with other related applications or how to upgrade it. Charms are programmed to understand a single application, its operations, and its potential to communicate or integrate with other applications. A charm defines and enables the channels by which applications connect.

### Cloud vendor agnostic

The instructions cover the deployment of FINOS Legend charmed operators on a host PC along with a locally deployed Gitlab instance. However these charms can be deployed on any cloud and should be able to use any Gitlab instance with requisite permissions.

### Multi-hybrid cloud

The Legend applications stack was deployed as a _bundle_ in the same cloud. [Juju](https://juju.is/) allows, however, for you to deploy each application on a different cloud and then integrate the stack across your estate.

### Offline installation

We assume your host had a functioning Internet connection. However it is also possible to [deploy charmed operators offline](https://juju.is/docs/olm/working-offline).

## Installation
To get started, you can checkout the [local run documention](LOCAL_RUN.md), which will walk you through and explain all the different deployment steps to run a local Legend instance, and will point you to docs for alternative deployments, such as clouds, barebone installations and other.

### Related Repositories

- [legend-juju-bundle](https://github.com/finos/legend-juju-bundle)
- [legend-juju-gitlab-integrator](https://github.com/finos/legend-juju-gitlab-integrator)
- [legend-juju-db-operator](https://github.com/finos/legend-juju-db-operator)
- [legend-juju-libs](https://github.com/finos/legend-juju-libs)
- [legend-juju-studio-operator](https://github.com/finos/legend-juju-studio-operator)
- [legend-juju-sdlc-server-operator](https://github.com/finos/legend-juju-sdlc-server-operator)
- [legend-juju-engine-server-operator](https://github.com/finos/legend-juju-engine-server-operator)

## Help and Support

Feel free to [create an issue](https://github.com/finos/legend-integration-juju/issues/new/choose) or submit a Pull Request to this repository in order to contribute; make sure to read the Legend [Contribution Guide](https://github.com/finos/legend/blob/master/CONTRIBUTING.md) first.

You can also use chat to the contributors to this Legend integration via the [FINOS Legend Slack Channel](finos-lf.slack.com) or the [Canonical Mattermost channel](https://chat.charmhub.io/charmhub/channels/charmed-legend).

## Roadmap

Visit our [roadmap](https://github.com/finos/legend#roadmap) to know more about the upcoming features.

## Contributing

Visit Legend [Contribution Guide](https://github.com/finos/legend/blob/master/CONTRIBUTING.md) to learn how to contribute to Legend on Juju.

## License

Copyright (c) 2021-present, Canonical

Distributed under the [Apache License, Version 2.0](http://www.apache.org/licenses/LICENSE-2.0).

SPDX-License-Identifier: [Apache-2.0](https://spdx.org/licenses/Apache-2.0)
