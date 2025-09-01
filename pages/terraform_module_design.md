---
layout: default
---

## Terraform Module Design

Practical guidance for inputs/outputs hygiene, versioning, composition, testing, and multi-env/tenant patterns.

### Principles
- Keep modules single‑purpose, composable, and well‑documented.
- Prefer explicit inputs/outputs over hidden provider or remote state coupling.
- Ship examples and a minimal README for each module.

### Inputs/Outputs hygiene
- Use strong types and defaults only when truly sensible.
- Document required inputs; keep names cloud‑agnostic when possible.
- Validate inputs close to the module boundary; mark secrets as sensitive.

```hcl
// variables.tf
variable "service_name" {
  type        = string
  description = "Logical name for the service"
}

variable "replicas" {
  type        = number
  description = "Desired replica count"
  default     = 2
  validation {
    condition     = var.replicas >= 1 && var.replicas <= 20
    error_message = "replicas must be between 1 and 20."
  }
}

variable "db_password" {
  type        = string
  description = "Database password"
  sensitive   = true
}

// outputs.tf
output "service_url" {
  description = "Public URL for the service"
  value       = module.ingress.url
}
```

### Versioning & constraints
- Pin Terraform and providers in `versions.tf`. Release modules via SemVer tags; write CHANGELOGs.

```hcl
// versions.tf
terraform {
  required_version = ">= 1.5, < 2.0"
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}
```

### Composition & reuse
- Treat `modules/*` as building blocks and `stacks/*` (or `environments/*`) as compositions.
- Keep cross‑module communication via outputs/inputs, not shared state.

```hcl
// stacks/prod/main.tf
module "network" {
  source = "../../modules/network"
  cidr_block = var.vpc_cidr
}

module "app" {
  source       = "../../modules/app"
  service_name = var.service_name
  subnet_ids   = module.network.private_subnet_ids
}
```

### Testing & quality gates
- Run `terraform fmt -check` and `terraform validate` in CI.
- Lint with `tflint`; scan with `tfsec` (or Checkov).
- For critical modules, add integration tests (e.g., Terratest) and at least one `examples/` plan.

### Multi‑env / multi‑tenant strategies
- Separate state per env/region/tenant. Avoid giant workspaces; prefer directory‑based stacks.
- Keep env deltas in `*.tfvars` and inject via CI.

```
infra/
  modules/
    network/
      main.tf
      variables.tf
      outputs.tf
      versions.tf
      README.md
      examples/
        simple/main.tf
    app/
      main.tf
      variables.tf
      outputs.tf
      versions.tf
      README.md
  stacks/
    prod/us-east-1/
      main.tf       # composes modules
      providers.tf
      backend.tf
      prod.tfvars
    staging/us-east-1/
      main.tf
      providers.tf
      backend.tf
      staging.tfvars
  tenants/
    acme/
      prod.tfvars
      staging.tfvars
    globex/
      prod.tfvars
```

### Single‑module repository layout (publishable)
```
terraform-aws-myservice/
  modules/
    core/
      main.tf
      variables.tf
      outputs.tf
      versions.tf
      README.md
      examples/
        minimal/main.tf
        with_db/main.tf
  .tflint.hcl
  .github/workflows/ci.yml
  CHANGELOG.md
```

### CI hints
- Plan on PR, apply on main with manual approval.

[back](../)


