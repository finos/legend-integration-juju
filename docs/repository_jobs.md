# FINOS Legend Juju EKS deployment - Periodic refresh

The [Refresh EKS FINOS Legend deployment](../.github/workflows/scheduled.yaml) job will run periodically for a configured Juju EKS deployment and will refresh the Legend Engine, Studio, and SDLC charms on the Staging environment to their latest revisions (including the Legend docker images associated to the revisions). This job requires a few Secrets to be defined in the project repository (``Settings > Secrets > Actions > Repository secrets``):

- ``KUBECONFIG_B64``: The EKS kubeconfig file in base64 format. The kubeconfig file is typically found in ``~/.kube/config``, and it can be transformed into base64 by running: ``cat ~/.kube/config | base64``.
- ``CONTROLLERS_B64``: The Juju ``controllers.yaml`` file in base64 format. The file is typically found in ``~/.local/share/juju/controllers.yaml``. The file will contain information about all the locally known Juju Controllers, but only the one containing the Legend model is needed. To transform it into base64, run: ``cat ~/.local/share/juju/controllers.yaml | base64``.
- ``CONTROLLER_PASSWORD``: The Juju Controller's admin password. It can be typically found in ``~/.local/share/juju/accounts.yaml``. This password will be used to authenticate into the Juju Controller.

The action will also send an email in case of failure, so it needs email related secrets to be set up as well: `MAILGUN_API_KEY`, `MAILGUN_DOMAIN`, `EMAIL_TO`.

Once the above repository secrets have been set, the job should succeed. A manual job can also be triggered by going to ``Actions > Refresh EKS FINOS Legend deployment > Run workflow``.

> **NOTE**: This action is automatically disabled once the [Switch EKS FINOS Legend Prod and Staging environments](../.github/workflows/switch_env.yaml) action has run. This allows the possiblity to switch back if something went wrong with the new Production environment. Otherwise, this daily action would refresh the staging environment to the latest version.


# Renew the CA-verified Certificate

The [Renew the CA-verified Certificate](../.github/workflows/renew_certificate.yaml) job runs periodically for a configured Juju EKS deployment and renew the TLS certificates for the EKS FINOS Legend [Staging environment](https://staging.legend.finos.org/) and the [Production environment](https://juju-acct.legend.finos.org).

This job uses the same Secrets defined above.


# Switch EKS FINOS Legend Prod and Staging environments

Currently, we have 2 Legend environments, the [Staging environment](https://staging.legend.finos.org/) and the [Production environment](https://juju-acct.legend.finos.org). The Staging environment is refreshed periodically, and it will have deployed the latest version of FINOS Legend. Monthly, these environments will be switched, promoting the Staging environment to Production and Staging the other environment. This will be triggered through the [Switch EKS FINOS Legend Prod and Staging environments](../.github/workflows/switch_env.yaml) manual action.

This job uses the same Secrets defined above.
