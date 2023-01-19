---
layout: default
---

## Running Terraform through Ansible

From my experience so far, I have seen a lot of organizations using multiple automation tools in their DevOps toolchain. There is no single tool that can do everything required.

Ansible is quite useful for many organization for creating automated playbooks for different tasks e.g. configuration of Nginx, or Management for SQL users etc. But its not very widely used for provisioning for Infrastructure on Cloud providers. For this the most common used tool is Terraform.

But there is a way where you can combine the two tools together to do provisioning with Ansible using Terraform module of ansible.

Here is the example:

### Official Documentation for the module

[https://docs.ansible.com/ansible/latest/collections/community/general/terraform_module.html]

### Example

Here I am running terraform through Ansible with some roles using the terraform module to help me provision resources on AWS.

##### Following is my Directory Structure:
```yaml
├── collections
│   └── requirements.yml
├── inventories
│   ├── dev
│   │   ├── client_a
│   │   │   ├── group_vars
│   │   │   │   └── client.yml
│   │   │   └── hosts
│   │   ├── client_b
│   │   │   ├── group_vars
│   │   │   │   └── client.yml
│   │   │   └── hosts
│   ├── prod
│   │   ├── client_a
│   │   │   ├── group_vars
│   │   │   │   ├── client.yml
│   │   │   └── hosts
│   │   ├── client_b
│   │   │   ├── group_vars
│   │   │   │   └── client.yml
│   │   │   └── hosts
│   └── staging
│       ├── client_a
│       │   ├── group_vars
│       │   │   └── client.yml
│       │   └── hosts
│       ├── client_b
│       │   ├── group_vars
│       │   │   └── client.yml
│       │   └── hosts
├── aws-provisioning-playbook.yml
└── roles
    ├── aws_vpc_setup
    │   ├── meta
    │   │   └── main.yml
    │   ├── tasks
    │   │   ├── main.yml
    │   │   └── terraform.yml
    │   └── templates
    │       └── terraform
    │           ├── internet_gateway.tf
    │           ├── main.tf
    │           ├── nat_gateway.tf
    │           ├── outputs.tf
    │           ├── subnets.tf
    │           └── variables.tf
    ├── aws_rds_setup
    │   ├── meta
    │   │   └── main.yml
    │   ├── tasks
    │   │   ├── main.yml
    │   │   └── terraform.yml
    │   └── templates
    │       └── terraform
    │           ├── credentials.tf
    │           ├── main.tf
    │           ├── outputs.tf
    │           ├── security_groups.tf
    │           └── variables.tf
    ├── common
    │   └── defaults
    │       └── main.yml
```


##### Inventory

I have organized the ansible inventory in main 3 folders which are my environments e.g. dev, staging and prod and then inside each folder I have multiple folders for my clients for which the specific variables can be different. 

##### Roles

Roles are divided for a specific resource type e.g. provisioning of VPC can be in one role and provisioning of RDS can be in another. Every role has its own terraform state file so this allows to have state files separate rather than in one long file also allows infrastructure to be divided and independent.

The common role can contain stuff that is common for all roles and then can be included in other roles.

The role itself contains meta, tasks and templates folder.
e.g.

meta/main.yml
```yaml
---
dependencies:
  - role: common
```
tasks/main.yml
```yaml
---
- name: Run Terraform
  ansible.builtin.import_tasks: terraform.yml
```

templates/terraform/main.tf
```tf
terraform {
  backend "s3" {
  }
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 4.37.0"
    }
  }
}

provider "aws" {
  region = var.aws_region
}

resource "aws_vpc" "base_vpc" {
  cidr_block           = var.aws_vpc_cidr
  enable_dns_hostnames = true
  tags = {
    Name = "${var.vpc_prefix}-vpc"
  }
}
```

##### Playbook

My main playbook on the main directory is the one which running the required roles for provisioning resources on AWS e.g

```yaml
---
- name: Provision resources in AWS
  hosts: client
  roles:
    - aws_vpc_setup
    - aws_rds_setup

```

### Conclusion

This is an interesting way of running Terraform, I cannot say its the best way because this does have some limitations e.g. if the team is big it gets a bit complex to properly plan, review and apply terraform. 

Some positives are that the developers can only interact with ansible to run automation playbooks and provisioning as well. And you can also use Ansible Tower which has a nice user interface to run playbooks and manage inventories.  

Reach out if you have any questions!

Ciao.


[back](../)
