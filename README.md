# Welcome to the Ansible part of the workshop

## The basics
- Where is the Ansible Automation Server? -> [here](https://caap.fvz.ansible-labs.de)
- What is my username? -> It is the email address you gave us when you signed up for this workshop
- What is my password? -> It's on the whiteboard in the workshop room (after you log in you can change your password if you want)

Ansible Automation Platform (AAP) has the notion of "Organizations". For this workshop an Organization named TechXchangeNL has been made and your account is part of that. Everything you do you will do within that organisation.

This workshop has been designed such that **you** will need to do most of the work, signifying the word "work" in workshop ;-) This means we only have the absolute basics set up and you need to build all the components to make everything work.

The division of work between Hashicorp Terraform Cloud and Ansible Automation Platform (as likely already explained to you) is:
- Terraform: Building up and changing infrastructure in the cloud, among which are RHEL10 servers
- AAP: Configure the RHEL10 servers to become a webserver serving a website

This means Terraform has all the credentials needed to do it's thing in the cloud of choice (AWS).
AAP will use the integrations with Terraform to be able to do it's thing on the provisioned infrastructure. AAP has no credentials for the cloud!

So what are the basics we have set up for you:
1. A machine credential named RHEL. You use this machine credential to be able to run your playbooks on the providioned servers.
2. A Custom Credential _Type_ called `Hashicorp Terraform Cloud`. You use this later to create your credential. See [here](https://docs.redhat.com/en/documentation/red_hat_ansible_automation_platform/2.6/html/getting_started_with_hashicorp_and_ansible_automation_platform/terraform-product#creating-custom-credential-type) for details.
3. An Inventory called `local` with the host `localhost` for api based automations.
4. A token to be able to do stuff in Hashicorp Terraform Cloud. This token is available as a var in HCP.
5. An Execution Environment called `ee-tech-x-change-nl` in AAP that provides all the collections and dependencies you need in this workshop.


## Collections
For Terraform there are currently 2 certified ansible collections that are made available in this workshop through the Execution Environment:
1. [cloud.terraform](https://caap.fvz.ansible-labs.de/content/collections/published/cloud/terraform/documentation/) - Maintained by Red Hat. It uses the terraform cli to talk to terraform.
2. [hashicorp.terraform](https://caap.fvz.ansible-labs.de/content/collections/published/hashicorp/terraform/documentation/) - Maintained by HashiCorp
All future development is on the hashicorp.terraform collection. It is the collection for integration with HashiCorp Terraform Enterprise and Cloud and it is based on the provided API. This workshop uses this collection where possible and falls back to the older cloud.terraform collection where needed. 


## Building Blocks
You need to create some building blocks in AAP for this workshop. This document explains what you need to make. When you are done, go back to the README.md in the workshop repo.

note: wherever you can and/or need to specify an Organization, choose `TechXchangeNL`, unless stated otherwise.


### Project
You need to create a project in AAP. The project is your repository with playbooks.
- Fork the the Ansible repository that you can find [here](https://github.com/TechXchangeNL/ansible.git)
- Create a project in `Automation Execution > Projects` and use this fork. Enable `Update REvisions on Job Lauch`


### Controller Credentials
Apart from the already available machine credential, you need a few more..

- A Controller Credential to be able to communicate with Hashicorp Terraform Cloud. Use Credential Type `Hashicorp Terraform Cloud` and the provided token in HCP.
- A Controller Credential to be able to sync the Terraform State File that will be used for the inventory source. Choose the credential type `Terraform backend configuration`. In the backend configuration field enter the following:

  ```text
  hostname = "app.terraform.io"  
  organization = "TechXchangeNL"  
  token = "YOURTOKENHERE"  
  workspaces { name = "YOURWORKSPACE" }  
  ```
  For token, enter the token provided in HCP
  For workspace enter the workspace you made in Terraform (you did...right?)


### EDA Credentials
For the new HCP Terraform _Actions_ feature we need to configure EDA and thus EDA Credentials. These are made under `Automation Decisions > Infrastructure > Credentials` We need two:
- A credential to enable EDA rulebook actions to lauch job templates and workflows in the Controller. It needs to be of type `Red Hat Ansible Automation Platform` Use your provided username and password and the url `https://caap.fvz.ansible-labs.de/api/controller`
- A credential that HCP Terraform Actions will use to be able to send events to EDA. It needs to be of type `Basic Event Stream`. You can camoe up with any username and password as long as you remember them for when they are needed to configure the HCP Terraform Action.


### Inventories
Create an inventory called "TechXchangeNL" and add a dynamic inventory source to it named "Terraform". This source is of type `Terraform State` and needs some configuration to do the magic of syncing the statefile. Use the provided execution environment and the config that you need to give in the `Source Variables` are:
```text
plugin: cloud.terraform.terraform_state
backend_type: remote
compose:
  ansible_host: public_ip
hostnames:
  - tag:Name
keyed_groups:
  - prefix: role
    key: tags.Role
```
  What does this do:
  - `compose` will use the value of the key `public_ip` received from HCP to create a key ansible_host with the same value. ansible_host is used by jobs to connect to the host.
  - `hostnames` is used to use the value of the tag `Name` as received from HCP to give the host its name.
  - `keyed_groups` is used to make inventory groups from tag values as received from HCP. So here a group will be created with prefix role and the value from tag Role as received from HPC. So: `role_<tagvalue>`. The host will be placed in this group.

Also, you need the `Terraform Backend Configuration` Credential you made before as the credential for this source. You can test it by syncing the source manually.
Do **NOT** enable _update on launch_.

### Job Templates
Now that you have the basics set up (project, credentials, inventory), you can define job templates in AAP. As you can see in the repository where this README lives, there are 3 playbooks:
- apply_plan.yml This playbook will run and apply a plan in HCP.
- deploy_webserver.yml. This playbook will deploy a webserver (apache)
- deploy_website.yml. This playbook will deploy a website

Have a look at the playbooks in this repo to get a sense of what they do. You might notice that the playbooks `deploy_webserver` and `deploy_website` are already made for you, but `apply_plan` is not. You need to develop this playbook yourself (because you are here to learn about the integration by doing, remember ;-) ). Use the embedded editor in github. The documentation that you need can be found under `Automation Content > Collections > hashicorp.terraform`. You need the `run` module. As the name of the playbook suggests you need to apply the plan.

> [!TIP]
> If you have timing issues we found that you need to enable polling with an interval of 5 and a timeout of 1200. Also make tf_timeout something like 6000.

Create a Job Template for each of these playbooks.
For the deploy_servers playbook:
- Use the provided "local" inventory. For all others the "TechXchangeNL" inventory.
- Use the "Hashicorp Terraform Cloud" credential you made.
- Add an extra var "workspace" with as the value the name of your workspace

For the other playbooks:
- Use the TechXchangeNL inventory
- Use the TechXchangeNL machine credential

### Workflows
Having job templates (automation building blocks) we create two workflows:
1. A workflow (name suggestion: "Deploy Web App") that runs the following playbooks in that order:
   1. sync inventory source Terraform
   2. update_server
   3. deploy_webserver
   4. deploy_website
2. A workflow (name suggestion: "Deploy Full Web App") that run the following in that specific oprder:
   1. deploy_servers (which applies the terraform plan from HashiCorp Terraform Cloud)
   3. the previously created workflow

### API Token
Part of the workshop is showing how you can run stuff _in_ AAP _from_ HashiCorp Terraform Cloud. For this, you need to provide a token from AAP to your HashiCorp Terraform Cloud workspace. You can create a token yourself using _API token_ under _Access Management_ in the menu. Choose write access. Copy/Paste the token somewhere, because it will only be shown once! HashiCorp Terraform Cloud will need both the username and the token.
