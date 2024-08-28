# chart-neuvector
Helm Chart for NeuVector

This chart builds upon the official Neuvector Helm chart adding a job that automates post-install and post-upgrade configuration.
As the chart pulls secrets from Azure keyvault using a secret volume, the changes are not generic enough to be merged to upstream and that is the reason for creating this derived chart.

## Functionalities
After installing Neuvector using the official Neuvector chart, this chart does the following:

1. Accepts the EULA if not accepted already (i.e. because this is an upgrade)
2. Sets the license key if not set already. You can set `config.forceLicenseUpdate` to `true` for force updating license
3. Changes the default admin password if it has not been changed already
4. Sets the Slack webhook URL
5. Sets the admission rules
6. Sets the response rules

For the secrets (e.g. admin password, license key) to be read from Azure keyvault, an Azure managed identity needs to be available.
For more information refer to the documentation related to Pod Identity and Azure provider for CSI driver:
- [Azure Key Vault Provider for Secrets Store CSI Driver](https://github.com/Azure/secrets-store-csi-driver-provider-azure)
- [AAD Pod Identity](https://github.com/Azure/aad-pod-identity)

The automated configuration script uses the Neuvector REST API therefore the controller service should be exposed at least internally 
(i.e. without ingress).

## Optional http->https redirect
An optional http->https redirect can be enabled on the manager ingress by setting the following boolean parameter:

```yaml
spec:
  values:
    neuvector:
      manager:
        ingress:
          httpredirect: true
```


## Releases
We use semantic versioning via GitHub releases to handle new releases of this application chart, this is done via automation called Release Drafter. When you merge a PR to master, a new draft release will be created.
More information is available about the [release process and how to create draft releases for testing purposes in more depth](https://hmcts.github.io/ops-runbooks/Testing-Changes/drafting-a-release.html)
