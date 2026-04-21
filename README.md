# chart-neuvector

Helm Chart for NeuVector

This chart wraps the upstream NeuVector `core` chart and adds HMCTS-specific resources around it.
The upstream dependency is included as the aliased subchart `neuvector`, while this repository adds the Azure Key Vault integration, Helm hook automation, and local policy resources that are not maintained upstream.

As the chart pulls secrets from Azure Key Vault using a secret volume, the changes are not generic enough to be merged upstream and that is the reason for creating this derived chart.

## How This Chart Customizes Upstream

On top of the upstream NeuVector release, this wrapper adds the following resources and behavior:

1. A post-install/post-upgrade Job runs `templates/configmap.yaml` via `templates/post-install-job.yaml`.
2. Azure Key Vault secrets are mounted into that Job so the hook can read the admin password, replacement admin password, enterprise license, and Slack webhook.
3. Admission control defaults are rendered as a `NvAdmissionControlSecurityRule` custom resource in `templates/admission-control.yaml`.
4. Response rules are rendered as `NvResponseRuleSecurityRule` custom resources in `templates/response-rules.yaml`.

## Post-Install Hook Responsibilities

The hook script customizes the running NeuVector deployment after the upstream chart has created the core controllers and services. Its responsibilities are:

1. Wait for NeuVector to finish restoring persisted state before applying API changes.
2. Authenticate using the bootstrap secret on first install, or the Azure Key Vault admin credentials on later runs.
3. Handle NeuVector's forced first-login password reset flow automatically when a token cannot yet be issued.
4. Accept the EULA if it has not already been accepted.
5. Reconcile the enterprise license from Key Vault by exporting the current NeuVector config, replacing the `object/config/license` entry, and re-importing it.
6. Rotate the admin password to the desired Key Vault value if the cluster is still using the bootstrap or previous password.
7. Configure the Slack webhook URL through the NeuVector system config API.

The hook is intentionally limited to runtime settings that the upstream chart does not manage declaratively. Admission control and response rules are no longer created by REST calls from the hook; they are rendered by Helm as CRDs instead.

You can set `config.forceLicenseUpdate` to `true` to force the license reconciliation step even when the imported config already contains the same license key.

## Troubleshooting / Debug Mode

By default the hook script suppresses verbose curl output and response bodies to avoid leaking authentication tokens and JWTs into pod logs. When troubleshooting a failed hook run you can re-enable that output without changing the chart code by setting `config.debug` to `true` via a Flux values patch:

```yaml
spec:
  values:
    config:
      debug: true
```

With debug enabled the hook will:

- Pass the `-v` flag to every `curl` call, printing request and response headers (including `X-Auth-Token`).
- Print the full response body after each JSON API call and file upload.

**Remove the patch and trigger a re-reconcile once troubleshooting is complete.** Leaving debug enabled on a long-running cluster means authentication tokens will appear in the hook job logs, which are accessible to anyone with `kubectl logs` access to the `neuvector` namespace.

For the secrets (for example admin password and license key) to be read from Azure Key Vault, an Azure managed identity needs to be available.
For more information refer to the documentation related to Pod Identity and Azure provider for CSI driver:

- [Azure Key Vault Provider for Secrets Store CSI Driver](https://github.com/Azure/secrets-store-csi-driver-provider-azure)
- [AAD Pod Identity](https://github.com/Azure/aad-pod-identity)

The automated configuration script uses the NeuVector REST API therefore the controller service should be exposed at least internally
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
