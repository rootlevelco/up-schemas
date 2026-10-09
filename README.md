# up-schemas

Curated KCL schema packages for [up](https://github.com/rootlevelco/up): typed resources for OpenTofu providers and Kubernetes kinds. Each package is published to `ghcr.io/patrycju/up-schemas/<name>:<version>`.

| Package | Versions | Source |
|---|---|---|
| `aws` | 6.66.0 | `hashicorp/aws` |
| `k8s` | 1.37.1 (`Pod`) | Kubernetes OpenAPI spec |

## Using a package

Pin it in your project's `kcl.mod`:

```toml
[dependencies]
aws = { oci = "oci://ghcr.io/patrycju/up-schemas/aws", tag = "6.66.0" }
```

up pulls it into `.up/cache/kcl/oci` on the first run. The packages are private for now, so set `UP_REGISTRY_TOKEN` (or `GITHUB_TOKEN`) to a token with `read:packages`.

Big providers are split by service: the first word of a Terraform type name is its subpackage, so `aws_s3_bucket` is `s3.Bucket`. Import only what you use; each import is parsed on every evaluation.

```python
import aws
import aws.s3

cloud = aws.Provider { config = {region = "eu-west-1"} }
logs = s3.Bucket { bucket = "my-logs" }
arn = Output(logs.arn)
```

`aws.Provider` is pinned to the package's provider version, so the `kcl.mod` tag is the only version to change.

## Adding or updating a package

1. Edit `curated.toml`: add a provider, or a version to an existing one.
2. `up schemas build` generates every listed package into `packages/` (not committed).
3. `up schemas publish` pushes the versions that aren't in the registry yet. It needs a token with `write:packages` in `UP_REGISTRY_TOKEN`.
