[![FINOS - Incubating](https://cdn.jsdelivr.net/gh/finos/contrib-toolbox@main/images/badge-incubating.svg)](https://github.com/finos/community/blob/master/governance/Software-Projects/Project-Lifecycle.md#incubating-projects)

# Legend Charmed Operator - aka Legend Charm Bundle
The Legend Charm Bundle provides a simple, efficient and enterprise-ready way to deploy and orchestrate a Legend instance in any type of environments, from developers workstations, up to production. The bundle includes several Charms, one for each Legend component.

Read more about Juju and Charms, access tutorials to deploy locally or in the cloud of your choice in minutes, and how you could support this initiative.

## What is Juju
The [Juju Charmed Operator Lifecycle Manager (OLM)](https://juju.is/docs/olm) is a hybrid-cloud application management and orchestration system for installation and day 2 operations. It helps deploy, configure, scale, integrate, maintain, and manage Kubernetes native, container-native and VM-native applications and the relations between them.

A charmed operator (also known, more simply, as a “charm”) encapsulates a single application and all the code and know-how it takes to operate it, such as how to combine and work with other related applications or how to upgrade it. Charms are programmed to understand a single application, its operations, and its potential to communicate or integrate with other applications. A charm defines and enables the channels by which applications connect.

### Cloud vendor agnostic
The instructions cover the deployment of FINOS Legend charmed operators on a host PC along with a locally deployed Gitlab instance. However these charms can be deployed on any cloud and should be able to use any Gitlab instance with requisite permissions.

### Multi-hybrid cloud
The Legend applications stack was deployed as a _bundle_ in the same cloud. [Juju](https://juju.is/) allows, however, for you to deploy each application on a different cloud and then integrate the stack across your estate.

### Offline installation
We assume your host had a functioning Internet connection. However it is also possible to [deploy charmed operators offline](https://juju.is/docs/olm/working-offline).

## Installation
This repository hosts the following tutorials to get started with Legend Charm from the ground up:
- The [local run documention](docs/deploy/local.md) guides you through the entire setup to run Legend in your local workstation, using microk8s; the documentation was tested on Ubuntu and MacOS workstations, but it should work on Windows workstations too
- The [AWS EKS documentation](docs/deploy/aws-eks.md) explains how to run Legend on a AWS EKS cluster from scratch.

If you'd like to see the cloud of your choice in this list, do not hesitate to [create an issue](https://github.com/finos/legend-integration-juju/issues)

## Versioning
The Legend Charm Bundle is composed by several charms, which are hosted in multiple github repositories (see below).

Bundles and charms can be released through 2 different channels: **latest/stable** and **latest/edge**; the stable bundle will use stable charm versions, and similarly edge bundles will use edge charm versions.

Both latest bundle and charm versions are aligned with the latest Legend release version,as specified by the [manifest.json](https://github.com/finos/legend/tree/master/releases) files included in Legend releases.

Bundles and charms using the previous Legend releases can be deployed from their respective channels based on the [official Legend releases](https://github.com/finos/legend/tree/master/releases)(newer than 2022-04-01``). For example, to deploy the ``2022-04-05`` release of Juju, the channel ``2022-04/edge`` can be used.

Visit https://charmhub.io/finos-legend-bundle to know more about Legend Bundle versions available.

## Legend Charm and Bundle release automation process

The repositories for each charm in the Legend bundle have been configured to build and publish a new charm revision whenever a new Pull Request merges on their respective repositories through GitHub actions. These charm revisions will also be released on the ``latest/edge`` channel with the latest associated Legend image revision.

The [legend-juju-bundle](https://github.com/finos/legend-juju-bundle) repository has a GitHub action configured to check periodically if there is a new [official Legend release](https://github.com/finos/legend/tree/master/releases). If there is a new release, the GitHub action will do the following steps:

- upload the Legend image versions specified in that release's ``manifest.json`` file into Charmhub.
- create a new release for the Legend Engine, SDLC, and Studio charms on the ``latest/edge`` and ``yyyy-mm/edge`` channels (where ``yyyy-mm`` is the year and month of the Legend release).
- create a new ``release-yyyy-mm-dd`` branch in the [legend-juju-bundle](https://github.com/finos/legend-juju-bundle) repository.
- update the ``bundle.yaml`` file, having the Legend Engine, SDLC, and Studio charms use the ``yyyy-mm/edge`` channels and create a commit.
- upload the new ``bundle.yaml`` file into Charmhub on the ``yyyy-mm/edge`` track.

Each repository has included in its ``README.md`` file and docs information about how their GitHub actions have been configured and what repositories secrets they require. In summary, they all require a Charmhub authentication token that has the permission to upload resources for the respective charms.

## FINOS Legend Juju acceptance environment

The [Juju acceptance environment](https://juju-acct.legend.finos.org/) has been deployed on EKS using the [EKS deployment guide](docs/deploy/aws-eks.md). This repository has been configured with a [GitHub action](.github/workflows/scheduled.yaml) that will update the Legend Engine, SDLC and Studio charms to their latest revisions (and thus, latest image versions).

For more information on how the GitHub action has been set up, see [here](docs/periodic_job.md).

### Related Repositories

You can browse all repositories related with the Legend Juju Bundle via https://github.com/topics/legend-juju

## Help and Support
Feel free to [create an issue](https://github.com/finos/legend-integration-juju/issues/new/choose) or submit a Pull Request to this repository in order to contribute; make sure to read the Legend [Contribution Guide](https://github.com/finos/legend/blob/main/CONTRIBUTING.md) first.

You can also use chat to the contributors to this Legend integration via the [FINOS Legend Slack Channel](finos-lf.slack.com) or the [Canonical Mattermost channel](https://chat.charmhub.io/charmhub/channels/charmed-legend).

## Roadmap
Visit our [roadmap](https://github.com/finos/legend#roadmap) to know more about the upcoming features.

## Contributing
Visit Legend [Contribution Guide](https://github.com/finos/legend/blob/main/CONTRIBUTING.md) to learn how to contribute to Legend on Juju.

## License
Copyright (c) 2021-present, Canonical

Distributed under the [Apache License, Version 2.0](http://www.apache.org/licenses/LICENSE-2.0).

SPDX-License-Identifier: [Apache-2.0](https://spdx.org/licenses/Apache-2.0)
