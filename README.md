# eks-spot-runners

[![zizmor](https://github.com/chorrell/eks-spot-runners/actions/workflows/zizmor.yml/badge.svg)](https://github.com/chorrell/eks-spot-runners/actions/workflows/zizmor.yml)

Ephemeral, self-hosted GitHub Actions runners on Amazon EKS, running on
**spot instances** for both **amd64 and arm64 (Graviton)**. Each platform of a
multi-platform Docker image builds natively on its own architecture, on
machines that exist only while a job runs.

This is an **example you bring up, use and tear down**, not a service that
runs 24/7: `terraform apply`, run some builds, `terraform destroy`. The example
workload is [docker-node-minimal](https://github.com/chorrell/docker-node-minimal),
which compiles Node.js from source for `linux/amd64` and `linux/arm64`, but any
image that needs two native platforms works the same way.

> **Status:** planning. Nothing is deployed yet; see [Roadmap](#roadmap).

## Why

docker-node-minimal's cold builds take 43–93 minutes per platform on GitHub's
4-vCPU hosted runners (amd64 is the slow one). Larger CPUs help, but GitHub's
larger runners bill per minute while they're up. Spot capacity on EKS gives
32 vCPUs per build for a fraction of on-demand price, and Karpenter removes the
node as soon as the job finishes.

## Design

| Piece | Choice | Why |
| --- | --- | --- |
| Cluster | EKS, standard support version | Managed control plane; stay off extended support (6× the hourly fee) |
| Network | VPC with **public subnets only, no NAT gateway** | A NAT gateway would cost more per month than the builds; nodes get public IPs with locked-down security groups |
| System nodes | One small Graviton node (`t4g`) for Karpenter and the runner controller | Smallest thing that can host the controllers |
| Build nodes | **Karpenter** `NodePool`: spot only, `amd64` and `arm64`, compute-optimized families (`c7g`, `c7i`, `c7a`, `c6g` ...), consolidate when empty | Spot plus a wide instance list keeps prices low and interruptions rare; nodes scale to zero between jobs |
| Runners | **Actions Runner Controller (ARC)** runner scale sets, one per architecture (`eks-spot-amd64`, `eks-spot-arm64`) | Jobs pick a platform with `runs-on`; each runner pod is ephemeral (one job, then gone) |
| Image builds | BuildKit in the runner pod (dind or rootless buildkitd, to be decided) | Native per-platform builds, no QEMU emulation |
| Auth | GitHub App for ARC, EKS Pod Identity for AWS access | No long-lived personal tokens or AWS keys |
| Infrastructure | Terraform (`terraform/`), Helm releases for Karpenter and ARC | One `apply` / `destroy` for the whole stack |

**Not used, because they cost more:** NAT gateways, EKS Auto Mode (adds a
per-instance management fee), on-demand build nodes, extended-support
Kubernetes versions.

## Cost model

Approximate us-east-1 on-demand prices as of November 2026; verify against the
[EKS](https://aws.amazon.com/eks/pricing/) and
[EC2](https://aws.amazon.com/ec2/pricing/on-demand/) pricing pages. ca-central-1
is roughly 10% higher.

| Item | Rate | Notes |
| --- | --- | --- |
| EKS control plane | $0.10/hr | Billed while the cluster exists, which is why it's torn down after use |
| System node (`t4g.small`/`medium`) | ~$0.02–0.03/hr | Spot is cheaper still |
| Build node `c7g.8xlarge` (32 vCPU Graviton) | ~$1.16/hr on-demand | Spot is typically 50–70% less |
| Build node `c7i.8xlarge` (32 vCPU) | ~$1.43/hr on-demand | Spot is typically 50–70% less |
| Public IPv4 addresses, EBS | Cents per hour | Small, but not zero |

**Example session:** two hours with the cluster up, building both platforms of
one Node.js release cold (about an hour of 32-vCPU spot capacity in total):
roughly **$1–3**. Compare with ~$73/month for the control plane alone if it
stayed up, or GitHub's 32-core larger runners at ~$0.082/minute.

Each build reports its runner instance type, spot price and duration, so the
README can show measured cost per build next to these estimates.

## Security

- Runners only take jobs from repositories and workflows you list. **Never let
  pull requests from forks run here**: a fork's PR could run arbitrary code on
  the cluster.
- Runner pods are ephemeral and run without host access beyond what the build
  needs.
- Workflows in this repo pin third-party actions by commit hash (zizmor
  enforces it).

## Roadmap

1. Terraform: VPC (public subnets), EKS, system node group, Karpenter.
2. Karpenter `NodePool` and `EC2NodeClass` for spot amd64 + arm64.
3. ARC controller and two runner scale sets; GitHub App auth.
4. Example workflow: build docker-node-minimal per platform with
   `runs-on: eks-spot-amd64` / `eks-spot-arm64`, then merge the manifest.
5. Measure: cold and warm build times and cost per build, against GitHub-hosted
   runners.
6. Write up results here.

Possible later steps: ArgoCD for the in-cluster pieces, spot interruption
handling demo, cosign signing with a Kyverno admission policy.

See also [doks-runners](https://github.com/chorrell/doks-runners), the same
idea on DigitalOcean Kubernetes.

## Local development

Install [prek](https://github.com/j178/prek) (`brew install prek`) and run the
hooks before pushing:

```sh
prek install
prek run --all-files
```

The markdownlint hook needs Docker running; the Terraform hooks need
`terraform` and `trivy`.

## License

[MPL-2.0](LICENSE)
