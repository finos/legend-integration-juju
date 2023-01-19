# FINOS Legend Juju EKS deployment - Periodic refresh

The [Refresh EKS FINOS Legend deployment](../.github/workflows/scheduled.yaml) job will run periodically for a configured Juju EKS deployment and will refresh the Legend Engine, Studio, and SDLC charms to their latest revisions (including the Legend docker images associated the revisions). This job requires a few Secrets to be defined in the project repository (``Settings > Secrets > Actions > Repository secrets``):

- ``KUBECONFIG_B64``: The EKS kubeconfig file in base64 format. The kubeconfig file is typically found in ``~/.kube/config``, and to can be transformed into base64 by running: ``cat ~/.kube/config | base64``.
- ``CONTROLLERS_B64``: The Juju ``controllers.yaml`` file in base64 format. The file is typically found in ``~/.local/share/juju/controllers.yaml``. The file will contain information about all the locally known Juju Controllers, but only the one containing the Legend model is needed. To transform it into base64, run: ``cat ~/.local/share/juju/controllers.yaml | base64``.
- ``CONTROLLER_PASSWORD``: The Juju Controller's admin password. It can be typically found found in ``~/.local/share/juju/accounts.yaml``. This password will be used to authenticate into the Juju Controller.

The action will also send an email in case of failure, so it needs email related secrets to be set up as well: `EMAIL_SERVER`, `EMAIL_PORT`, `EMAIL_USERNAME`, `EMAIL_PASSWORD`, `EMAIL_TO`, `EMAIL_FROM`.

Once the above repository secrets have been set, the job should succeed. A manual job can also be triggered by going to ``Actions > Refresh EKS FINOS Legend deployment > Run workflow``.
