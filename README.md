# Official Terraform Provider

<div align="center">

![CI](https://github.com/upbound/provider-terraform/workflows/CI/badge.svg) [![GitHub release](https://img.shields.io/github/release/upbound/provider-terraform/all.svg?style=flat-square)](https://github.com/upbound/provider-terraform/releases) [![Go Report Card](https://goreportcard.com/badge/github.com/upbound/provider-terraform)](https://goreportcard.com/report/github.com/upbound/provider-terraform) [![Slack](https://slack.crossplane.io/badge.svg)](https://crossplane.slack.com/archives/C01TRKD4623) [![Twitter Follow](https://img.shields.io/twitter/follow/upbound_io.svg?style=social&label=Follow)](https://twitter.com/intent/follow?screen_name=upbound_io&user_id=788180534543339520)

</div>

Provider Terraform is a [Crossplane](https://crossplane.io/) provider that 
can run Terraform code and enables defining new Crossplane Composite Resources (XRs)
that are composed of a mix of 'native' Crossplane managed resources and your
existing Terraform modules.

The Terraform provider adds support for a `Workspace` managed resource that
represents a Terraform workspace. The configuration of each workspace may be
either fetched from a remote source (e.g. git), or simply specified inline.

```yaml
apiVersion: tf.upbound.io/v1beta1
kind: Workspace
metadata:
  name: example-inline
  annotations:
    # The terraform workspace will be named 'coolbucket'. If you omitted this
    # annotation it would be derived from metadata.name - i.e. 'example-inline'.
    crossplane.io/external-name: coolbucket
spec:
  forProvider:
    # For simple cases you can use an inline source to specify the content of
    # main.tf as opaque, inline HCL.
    source: Inline
    module: |
      // All outputs are written to the connection secret.  Non-sensitive outputs
      // are stored as string values in the status.atProvider.outputs object.
      output "url" {
        value       = google_storage_bucket.example.self_link
      }

      resource "random_id" "example" {
        byte_length = 4
      }

      // The google provider and remote state are configured by the provider
      // config - see examples/providerconfig.yaml.
      resource "google_storage_bucket" "example" {
        name = "crossplane-example-${terraform.workspace}-${random_id.example.hex}"
      }
  writeConnectionSecretToRef:
    namespace: default
    name: terraform-workspace-example-inline
```

```yaml
apiVersion: tf.upbound.io/v1beta1
kind: Workspace
metadata:
  name: example-remote
  annotations:
    crossplane.io/external-name: myworkspace
spec:
  forProvider:
    # Use any module source supported by terraform init -from-module.
    source: Remote
    module: https://github.com/crossplane/tf
    # Variables can be specified inline, or loaded from a ConfigMap or Secret.
    vars:
    - key: region
      value: us-west-1
    varFiles:
    - source: SecretKey
      secretKeyRef:
        namespace: default
        name: terraform
        key: example.tfvar.json
  # All Terraform outputs are written to the connection secret.
  writeConnectionSecretToRef:
    namespace: default
    name: terraform-workspace-example-inline
```

## Getting Started

Follow the quick start guide [here](https://marketplace.upbound.io/providers/upbound/provider-terraform/latest/docs/quickstart).

You can find a detailed API reference with all CRDs and examples [here](https://marketplace.upbound.io/providers/upbound/provider-terraform/latest/crds).

## Further Configuration

You can find more information about configuring the provider further [here](https://marketplace.upbound.io/providers/upbound/provider-terraform/latest/docs/configuration).

## Known limitations:

* You must either use remote state or ensure the provider container's `/tf`
  directory is not lost. `provider-terraform` __does not persist state__;
  consider using the [Kubernetes] remote state backend.
* If the module takes longer than the value of `--timeout` (default is 20m) to apply the
  underlying `terraform` process will be killed. You will potentially lose state
  and leak resources.  The workspace lock will also likely be left in place and need to be manually removed
  before the Workspace can be reconciled again.
* The provider won't emit an event until _after_ it has successfully applied the
  Terraform module, which can take a long time.
* Setting --max-reconcile-rate to a value greater than 1 will potentially cause the provider
  to use up to the same number of CPUs.  Add a resources section to the ControllerConfig to restrict
  CPU usage as needed.
  [Kubernetes]: https://www.terraform.io/docs/language/settings/backends/kubernetes.html
  [git credentials store]: https://git-scm.com/docs/git-credential-store

## Report a Bug

For filing bugs, suggesting improvements, or requesting new features, please
open an [issue](https://github.com/upbound/provider-terraform/issues).

## Contact

Please open a Github issue for all requests. If you need to reach out to Upbound,
you can do so via the following channels:
* Slack: [#upbound](https://crossplane.slack.com/archives/C01TRKD4623) channel in [Crossplane Slack](https://slack.crossplane.io)
* Twitter: [@upbound_io](https://twitter.com/upbound_io)
* Email: [support@upbound.io](mailto:support@upbound.io)

## Licensing

Provider Terraform is under [the Apache 2.0 license](LICENSE) with [notice](NOTICE).
