
# chart-neuvector


Helm Chart for NeuVector

This chart wraps the upstream NeuVector `core` chart and adds HMCTS-specific resources around it.
The upstream dependency is included as the aliased subchart `neuvector`, while this repository adds the Azure Key Vault integration, Helm hook automation, and local policy resources that are not maintained upstream.

## Functionalities

As the chart pulls secrets from Azure Key Vault using a secret volume, the changes are not generic enough to be merged upstream and that is the reason for creating this derived chart.

## How This Chart Customizes Upstream

On top of the upstream NeuVector release, this wrapper adds the following resources and behavior:

1. A post-install/post-upgrade Job (`templates/post-install-job.yaml`) mounts and executes the `config.sh` script provisioned by the `neuvector-restapi-config` ConfigMap rendered from `templates/configmap.yaml`.
2. Azure Key Vault secrets are mounted into that Job so the hook can read the admin password, replacement admin password, enterprise license, and Slack webhook.
3. Admission control defaults are rendered as a `NvAdmissionControlSecurityRule` custom resource in `templates/admission-control.yaml`.
4. Response rules are rendered as `NvResponseRuleSecurityRule` custom resources in `templates/response-rules.yaml`.

## Rule values

The chart now renders admission and network CRDs from values instead of hardcoded templates.

- `rules.admission.defaultRules` contains the chart baseline admission rules.
- `rules.admission.rules` can be used to add environment-specific admission rules.
- `rules.admission.includeDefaultRules` can be set to `false` if an environment needs to replace the baseline.
- `rules.network.defaultRules` contains the chart baseline network CRDs.
- `rules.network.rules` can be used to add environment-specific network CRDs.
- `rules.network.includeDefaultRules` can be set to `false` if an environment needs to replace the baseline.
- `groups` can be used to add environment-specific NeuVector group CRDs alongside network rules.

Custom rules in `rules.*.rules` are rendered **before** the baseline rules in the output, so they take precedence in NeuVector's top-down, first-match-wins evaluation.

This split is intentional because Helm replaces arrays during values merges rather than appending them.

### Policy Modes

NeuVector uses three policy modes, controlled by the `policymode` field on a rule's `target`:

| Mode | Behaviour |
|------|-----------|
| `Discover` | Learning mode — no alerts, no blocking. NeuVector observes traffic and builds a baseline. |
| `Monitor` | Audit mode — violations are logged and alerted, but traffic is **not** blocked. |
| `Protect` | Enforcement mode — unauthorized traffic is **blocked** and violations are logged. |

`Protect` is the enforcement mode. There is no separate `Enforce` value — **`Protect` equals enforce**.

### `includeDefaultRules` scope

`rules.network.includeDefaultRules` is a **global toggle for the entire HelmRelease**. Setting it to `false` in a Flux overlay removes the chart baseline network rules (`allpods`, `azurepostgresql`, etc.) for **every namespace**, not just the one being targeted. Only disable it when you are replacing the full rule set for the deployment. If you only want to restrict a specific namespace, keep `includeDefaultRules: true` and add a namespace-scoped custom rule alongside the defaults — NeuVector matches rules top-down and the custom rule will take precedence for the target selector.

For a strict namespace-scoped policy such as `<myNamespace>`, the current chart baseline is broader than the goal because `rules.network.defaultRules` includes the cluster-wide `allpods` rule and allows more than PostgreSQL and DNS. In that case, disable the baseline network rules in Flux and provide namespace-specific rules instead.

Example Flux values:

```yaml
spec:
  values:
    rules:
      network:
        includeDefaultRules: false
        rules:
          - apiVersion: neuvector.com/v1
            kind: NvClusterSecurityRule
            metadata:
              name: <myNamespace>-egress
              namespace: ""
            spec:
              egress:
                - action: allow
                  applications:
                    - DNS
                  name: <myNamespace>-dns-egress
                  ports: udp/53,tcp/53
                  priority: 0
                  selector:
                    comment: ""
                    criteria:
                      - key: address
                        op: =
                        value: kube-dns-upstream.kube-system.svc.cluster.local
                    name: KubeDns
                    original_name: ""
                - action: allow
                  applications:
                    - PostgreSQL
                    - SSL
                  name: <myNamespace>-postgres-egress
                  ports: tcp/5432
                  priority: 0
                  selector:
                    comment: ""
                    criteria:
                      - key: address
                        op: =
                        value: '*.postgres.database.azure.com'
                    name: AzurePostgreSQL
                    original_name: ""
              file: []
              ingress: []
              process: []
              target:
                policymode: Protect
                selector:
                  comment: ""
                  criteria:
                    - key: namespace
                      op: =
                      value: <myNamespace>
                  name: <myNamespace>
                  original_name: ""
```

> **Note:** Validate the DNS selector against the cluster's actual DNS service before rollout. In this estate there is evidence of `kube-dns-upstream` under `kube-system`, but some clusters may differ.

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
- [AAD Workload Identity](https://github.com/Azure/azure-workload-identity)

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

`httpredirect` now defaults to `false`, so the chart renders cleanly when the setting is omitted.


## Releases

We use semantic versioning via GitHub releases to handle new releases of this application chart, this is done via automation called Release Drafter. When you merge a PR to master, a new draft release will be created.
More information is available about the [release process and how to create draft releases for testing purposes in more depth](https://hmcts.github.io/ops-runbooks/Testing-Changes/drafting-a-release.html)

---

## Examples

Below are examples of how to use the chart inputs to add each available NeuVector resource type for a generic namespace (e.g., `my-namespace`).

### 1. Network Rule Example

```yaml
rules:
  network:
    includeDefaultRules: true
    rules:
      - apiVersion: neuvector.com/v1
        kind: NvClusterSecurityRule
        metadata:
          name: my-namespace-egress
        spec:
          egress:
            - action: allow
              applications: [SSL]
              name: AllowExternal443
              ports: tcp/443
              selector:
                criteria:
                  - key: address
                    op: =
                    value: example.com
          ingress: []
          file: []
          process: []
          target:
            policymode: Protect
            selector:
              criteria:
                - key: namespace
                  op: =
                  value: my-namespace
              name: MyNamespacePods
          waf:
            settings: []
            status: true
```

### 2. Admission Control Rule Example

```yaml
rules:
  admission:
    enable: true
    includeDefaultRules: true
    rules:
      - action: deny
        criteria:
          - name: namespace
            op: =
            value: my-namespace
        comment: "Deny all admissions in my-namespace"
```

### 3. Response Policy Example

```yaml
rules:
  response:
    policies:
      - name: block-priv-escalation
        event: security-event
        conditions:
          - type: name
            value: Container.Privilege.Escalation
        actions:
          - quarantine
        disable: false
      - name: notify-on-deny
        event: admission-control
        conditions:
          - type: name
            value: Admission.Control.Denied
        actions:
          - webhook
        disable: false
```

### 4. Group Example

```yaml
groups:
  - name: my-namespace-group
    criteria:
      - key: namespace
        op: =
        value: my-namespace
```
