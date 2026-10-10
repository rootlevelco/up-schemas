# up-schemas

Curated KCL schema packages for [up](https://github.com/rootlevelco/up): typed resources for OpenTofu providers and Kubernetes kinds. Each package is published to `ghcr.io/rootlevelco/up-schemas/<name>:<version>`.

| Package | Versions | Source |
|---|---|---|
| `aws` | 6.66.0 | `hashicorp/aws` |
| `azurerm` | 5.9.0 | `hashicorp/azurerm` |
| `google` | 8.6.0 | `hashicorp/google` |
| `hcloud` | 1.69.0, 1.70.0 | `hetznercloud/hcloud` |
| `helm` | 3.3.0 | `hashicorp/helm` |
| `k8s` | 1.37.1 (every kind up serves) | Kubernetes OpenAPI spec |

## Using a package

Pin it in your project's `kcl.mod`:

```toml
[dependencies]
aws = { oci = "oci://ghcr.io/rootlevelco/up-schemas/aws", tag = "6.66.0" }
```

up pulls it into `.up/cache/kcl/oci` on the first run. The packages are private for now, so set `UP_REGISTRY_TOKEN` (or `GITHUB_TOKEN`) to a token with `read:packages`.

Big providers (aws, azurerm, google) are split by service: the first word of a Terraform type name is its subpackage, so `aws_s3_bucket` is `s3.Bucket` and `azurerm_resource_group` is `resource.Group`. Import only what you use; each import is parsed on every evaluation. hcloud and k8s are one package each.

```python
import aws
import aws.s3

cloud = aws.Provider { config = {region = "eu-west-1"} }
logs = s3.Bucket { bucket = "my-logs" }
arn = Output(logs.arn)
```

`aws.Provider` is pinned to the package's provider version, so the `kcl.mod` tag is the only version to change.

```python
import hcloud
import k8s

token = Secret { env = "HCLOUD_TOKEN" }
cloud = hcloud.Provider { config = {token = token.value} }
net = hcloud.Network { tf_name = "apps", ip_range = "10.0.0.0/16" }
web = hcloud.Server {
    tf_name = "web"
    server_type = "cx23"
    image = "ubuntu-24.04"
    network = [{network_id = net.id}]
}

cluster = Kubernetes { context = "dev" }
app = k8s.Deployment {
    metadata.namespace = "apps"
    spec = {
        selector.matchLabels = {app = "web"}
        template = {
            metadata.labels = {app = "web"}
            spec.containers = [{name = "web", image = "nginx:1.27"}]
        }
    }
}
```

`k8s` has every built-in kind with a stable API version. Kinds only in alpha or beta APIs use `KubernetesManifest`. Custom resources get schemas from their CRDs with `up gen crds`.

Helm charts go through `hashicorp/helm`:

```python
import helm
import yaml

charts = helm.Provider { config.kubernetes = {config_context = "dev"} }
pg = helm.Release {
    tf_name = "cnpg"
    repository = "https://cloudnative-pg.github.io/charts"
    chart = "cloudnative-pg"
    version = "0.28.2"
    namespace = "cnpg-system"
    create_namespace = True
    values = [yaml.encode({monitoring.podMonitorEnabled = False})]
}
```

## Adding or updating a package

1. Edit `curated.toml`: add a provider, or a version to an existing one.
2. `up schemas build` generates every listed package into `packages/` (not committed).
3. `up schemas publish` pushes the versions that aren't in the registry yet. It needs a token with `write:packages` in `UP_REGISTRY_TOKEN`.
