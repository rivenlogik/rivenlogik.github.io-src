---
title: Terraform Notes
layout: post
categories: tools
date: 2020-07-14 22:42 -0500
---
![Terraform](data:image/png;base64,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)

Below are notes I took a few months ago when doing a Udemy Terraform course.  I may update them over time as I learn more.

## Terraform commands

* Plan
* Apply
* Destroy

### Plan

Plan is used to preview changes to the infrastructure.  
Use:  ``terraform plan`` for output stdin, or in practice ``teraform plan -out <file>``  
The file is used in apply.

### Apply

Makes changes to the infrastructure.

### Destroy

Removes all infrastucture defined in plan files.  Be careful in production!

## Terraform State

Terraform keeps the remote state of the infrastructure.

It stores it in a file called ``terraform.tfstate``.  There's also a backup of the previous state (.backup).

If you run terraform apply after terminating an instance (or any defined node etc.) manually, Terraform will re-create it if it is part of the configuration.

The state file can be put into version control to see history of the infrastructure (JSON file). Allows for collaboration (beware of conflicts).

Large projects require storage of state in a remote repository.

Best not to have giant state files.

### Remote Storage

This is called the *backend*.

* S3
* Consul
* Terraform enterprise

#### Benefits

* Allows for collaboration, always available to the whole team.
* Sensitive information is not stored locally.
* *Enhanced backends* allow for remote operations.
* Avoid having to push the terraform.tfstate to version control.

#### Cautions

* Not all remote stores support *locking*.  S3 and Consul do.

### Example Configuration

```hcl
terraform {
    backend "s3" {
        bucket = "bucket"
        key = "terraform/myproject"
        region = "us-west-1"
    }
}
```

### Other State Information

You can also specify a read-only remote store directly in ``.tf`` files.  These are called a *datasource*.

## Terraform Variables

Terraform variables were completely re-worked in 0.12 release.  

### Simple Types

* String
* Number
* Bool

### Complex Types

* List
* Set
* Map
* Object
* Tuple

#### List

Lists are always ordered.

Referencing: ``myvar[0]`` index number

#### Set

Like a list, but always unique values and does not maintain order, ie. ``[5,1,2,2,5] == [1,2,5]``

#### Map

Like a dictionary (key/value).  
Referencing: ``myvar['key']``

#### Object

Like a map, but each element can have a different type. First is string, second is number.

```hcl
{  
firstname = "Bob"
housenumber = 100  
}
```

#### Tuple

Like a list, but each element can have a different type.  First is string, second is boolean, third is number.  
ie. ``["test", false, 0]``

### Other Variable Notes

* Variables that are declared in ``vars.tf`` but not defined will need to be defined in the ``terraform.tfvars`` file.
* Best practice to NOT commit .tfvars files to repositories.
