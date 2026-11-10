# AGENTS.md

An example of ephemeral, self-hosted GitHub Actions runners on Amazon EKS,
using spot instances for amd64 and arm64 (Graviton) builds. It is brought up,
used and torn down, never run 24/7. See [README.md](README.md) for the design,
cost model and roadmap.

## Structure

```text
├── terraform/                  # (planned) VPC, EKS, system node group, Karpenter, ARC Helm releases
├── kubernetes/                 # (planned) Karpenter NodePool/EC2NodeClass, runner scale set values
├── .pre-commit-config.yaml     # gitleaks, markdownlint-cli2 (Docker), hygiene hooks, zizmor, terraform fmt/validate/trivy
├── .markdownlint.yaml          # MD013 (line length) disabled
├── AGENTS.md / CLAUDE.md       # Agent guidance; CLAUDE.md only imports AGENTS.md
├── LICENSE                     # MPL-2.0
└── .github/
    ├── CODEOWNERS              # * @chorrell
    ├── dependabot.yml          # `tooling` group (Actions + pre-commit) plus a separate terraform entry
    ├── zizmor.yml              # chorrell/* may be tag-pinned; everything else hash-pinned
    └── workflows/              # Stubs calling chorrell/workflows (markdownlint, pre-commit, zizmor, automerge)
```

## Rules

- **Cheapest working option wins.** No NAT gateways (public subnets only), no
  EKS Auto Mode, no on-demand build nodes, no extended-support Kubernetes
  versions. If a change adds a cost that runs while the cluster is idle, call
  it out in the PR.
- **Everything must tear down cleanly.** `terraform destroy` has to remove
  every billable resource, including Karpenter-launched instances, load
  balancers and volumes. Don't create AWS resources outside Terraform.
- **Spot and both architectures.** Build capacity is spot only, for `amd64` and
  `arm64`. Keep the instance list wide so spot capacity is available.
- **Never run fork pull requests on these runners.** Runner scale sets are
  scoped to named repositories, and example workflows only run on push or
  `workflow_dispatch`.
- **No secrets in the repo.** ARC uses a GitHub App; AWS access uses EKS Pod
  Identity. Variable files with values (`*.auto.tfvars`) are gitignored.
- Cost figures in the README are estimates with a date; replace them with
  measured numbers as builds run, and say which are which.
- **Reusable workflows live in [chorrell/workflows](https://github.com/chorrell/workflows).**
  Workflows here are stubs that call them by tag (`@v1`). Third-party actions
  are pinned by commit hash with a version comment; zizmor enforces this.
  Reference same-repository workflows with `$/` syntax, not `./`.
- **Repository settings aren't configured here.** Merge settings, rulesets and
  required status checks live in
  [chorrell/github-settings](https://github.com/chorrell/github-settings)
  (`repos.yaml`). Don't change them in the GitHub UI.
- Run `prek run --all-files` before pushing (markdownlint needs Docker; the
  Terraform hooks need `terraform` and `trivy`, and CI skips them).
- Dependabot opens one monthly `tooling` PR for Actions and pre-commit hooks,
  auto-merged when every update is minor or patch. Terraform updates arrive in
  their own PR and are never auto-merged.
- Commits and PR titles use conventional commits (`feat: ...`, `ci: ...`,
  `chore(deps): ...`). Branch from `main` with a `feature/` prefix. PRs are
  squash-merged, so keep PR bodies short and focused on intent.
