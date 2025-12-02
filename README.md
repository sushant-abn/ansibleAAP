# Welcome to the Ansible part of the workshop

## The basics
The division of work between HCP and AAP (as likely already explained to you) is:
- HCP Terraform: Building up and changing infrastructure in the cloud, among which are RHEL10 servers
- AAP: Configure the servers to become a webserver serving a website

This seperation of concerns means HCP Terraform has all the credentials needed to do it's thing in the cloud of choice (AWS) and AAP has no credentials nor visibility in the cloud infrastructure. AAP uses the integrations with HCP Terraform to be able to do it's thing on the provisioned infrastructure.

So what are the basics we have set up for you:
1. AAP has the notion of _Organizations_. For this workshop an Organization named `TechXchangeNL` has been made and you will work within that organization.
2. A _Machine Credential_ named `RHEL`. You use this machine credential to be able to run your playbooks on the provisioned servers. HCP Terraform deploys the keys inside this credential to the cloud.
3. A _Custom Credential Type_ called `Hashicorp Terraform Cloud`. You will use this later to create your own credential of this type. See [here](https://docs.redhat.com/en/documentation/red_hat_ansible_automation_platform/2.6/html/getting_started_with_hashicorp_and_ansible_automation_platform/terraform-product#creating-custom-credential-type) for details.
4. An _Inventory_ called `local` with the host `localhost` for api based automations. Playbooks that use an api for their work typically use localhost.
5. A _token_ to be able to do stuff in Hashicorp Terraform Cloud. This token is available as a var in HCP Terraform.
6. An _Execution Environment_ called `ee-tech-x-change-nl` in AAP that provides all the collections and dependencies you need in this workshop. For Terraform there are currently 2 certified ansible collections:
  - [cloud.terraform](https://caap.fvz.ansible-labs.de/content/collections/published/cloud/terraform/documentation/) - Maintained by Red Hat. It uses the terraform cli to talk to terraform.
  - [hashicorp.terraform](https://caap.fvz.ansible-labs.de/content/collections/published/hashicorp/terraform/documentation/) - Maintained by HashiCorp
  > All future development is on the hashicorp.terraform collection. It is the collection for integration with HashiCorp Terraform Enterprise and Cloud and it is based on the provided API. This workshop uses this collection where possible and falls back to the older cloud.terraform collection where needed. 


This workshop has been designed such that **you** will need to do most of the work, signifying the word "work" in workshop :wink:. This means we only have the absolute basics set up and you need to build all the components to make everything work. Fun, right? :tada:

- Where is the Ansible Automation Server? -> [here](https://caap.fvz.ansible-labs.de)
- What is my username? -> It is the email address you gave us when you signed up for this workshop
- What is my password? -> You can find it as a stored variable in HCP Terraform under `Default Project -> Settings -> Variable Sets` with a key of `AAP_UI_PASSWORD`


## Controller Building Blocks
You need to create some building blocks in AAP for this workshop. This document explains what you need to make. When you are done, go back to the README.md in the workshop repo.

> Wherever you can and/or need to specify an Organization, choose `TechXchangeNL`, unless stated otherwise.


### Project
You need to create a project in AAP. The project is your repository with playbooks.
- Log into github with your own account. If you do not have one, create one.
- Fork the the Ansible repository that you can find [here](https://github.com/TechXchangeNL/ansible.git)
- Create a project in `Automation Execution > Projects` and use this fork. Enable `Update REvisions on Job Lauch`


### Controller Credentials
Apart from the already available machine credential, you need a few more..

- A Credential to be able to communicate with Hashicorp Terraform Cloud. Use Credential Type `Hashicorp Terraform Cloud` and the provided token in a var in HCP Terraform (formely known as Terraform Cloud).
- A Credential to be able to sync the Terraform State File that will be used for the inventory source. Choose the credential type `Terraform backend configuration`. In the backend configuration field enter the following:

  ```text
  hostname = "app.terraform.io"  
  organization = "TechXchangeNL"  
  token = "YOURTOKENHERE"  
  workspaces { name = "YOURWORKSPACE" }  
  ```

> You need to have a Terraform token for both credentials. You find this in HCP Terraform under Projects -> defualt project -> settings -> variable sets -> AAP -> TERRAFORM_TOKEN. For workspace enter the workspace you made in Terraform as part of the prep work you need to do in HCP Terraform

### Inventories
Create an inventory called "TechXchangeNL" and add a dynamic inventory source to it named "Terraform". This source is of type `Terraform State` and needs some configuration to do the magic of syncing the statefile. Use the provided execution environment `ee-tech-x-change-nl` and the credential you made for it. The config that you need to give in the `Source Variables` is:
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
  - `compose` will use the value of the key `public_ip` received from HCP Terraform to create a key ansible_host with the same value. `ansible_host` is used by jobs to connect to the host.
  - `hostnames` is used to use the value of the tag `Name` as received from HCP Terraform to give the host its name.
  - `keyed_groups` is used to make inventory groups from tag values as received from HCP Terraform. So here a group will be created with prefix role and the value from tag Role as received from HPC. So: `role_<tagvalue>`. The host will be placed in this group.

Also, you need the `Terraform Backend Configuration` Credential you made before as the credential for this source. You can test it by syncing the source manually.
Do **NOT** enable _update on launch_.


### Playbooks
As you can see in the repository where this README lives, there are 3 playbooks:
- apply_plan.yml This playbook will run and apply a plan in HCP Terraform.
- deploy_webserver.yml. This playbook will deploy a webserver (apache)
- deploy_website.yml. This playbook will deploy a website

Have a look at the playbooks in this repo to get a sense of what they do. You might notice that the playbooks `deploy_webserver` and `deploy_website` are already made for you, but `apply_plan` is not. You need to develop this playbooks yourself (because you are here to learn the integration, remember 😉). Use the embedded editor in github. The documentation that you need can be found under `Automation Content > Collections > hashicorp.terraform`. You need the `run` module. As the name of the playbook suggests you need to `apply` the plan.

> If you have timing issues we found that you need to enable polling with an interval of 5 and a timeout of 1200. Also make tf_timeout something like 6000.


### Job Templates
Now that you have the basics set up (project, credentials, inventory, playbooks), you can define job templates in AAP. 
Create a Job Template for each of these playbooks.

For the apply_plan playbook:
- Use the provided `local` inventory.
- Use the `Hashicorp Terraform Cloud` credential you made.

For the other playbooks:
- Use the `TechXchangeNL` inventory
- Use the `RHEL` machine credential

### API Token
Part of the workshop is showing how you can run stuff _in_ AAP _from_ HashiCorp Terraform Cloud. For this, you need to provide a token from AAP to your HCP Terraform workspace. You can create a token yourself using _API token_ under _Access Management_ in the menu. Choose write access. Copy/Paste the token somewhere, because it will only be shown once! Then, store the token in HCP Terraform as a env under your workspace variables with key `AAP_TOKEN`.

## EDA building blocks
The following building blocks are needed for the new _Terraform Actions_ feature and are made under `Ansible Decision`.

> Wherever you can and/or need to specify an Organization, choose `TechXchangeNL`, unless stated otherwise.

### rulebook
Event Drive Ansible works with rulebooks. You find a template rulebook in this repo under the directory `rulebooks` named `terrafrom_actions.yml`. It is not finished and you need to define the rule that comes from the `HCP Terraform Action`. The documentation for making rules you find [here](https://docs.ansible.com/projects/rulebook). You need to craft a rule that:
- Checks for the condition `event.payload.template_type == "workflow_job"`
- If the condition is met, perform an action to run a workflow_job with the following parameters:
  - name: "{{ event.payload.workflow_job_template_name }}"
  - organization: "{{ event.payload.organization_name }}"

### Project
Make a project with the same URL as the project you made for the Controller. (We have choosen to put the playbooks for controller and rulebooks for EDA in the same repo for this workshop. As you can see rulebooks are stored in a seperate folder. However, You do need to define a seperate project for EDA).

### Credentials
These are made under `Automation Decisions > Infrastructure > Credentials` We need two:
- A credential to enable EDA rulebook actions to launch _Job Templates_ and _Workflows_ in the Controller. It needs to be of type `Red Hat Ansible Automation Platform`. Use your provided username and password and the url `https://caap.fvz.ansible-labs.de/api/controller`
- A credential that _HCP Terraform Actions_ will use to be able to send events to EDA. It needs to be of type `Basic Event Stream`. You can camoe up with any username and password as long as you remember them for when they are needed to configure the _HCP Terraform Action_.

### Event Stream
Define an Event Stream. This is used by _Terraform Actions_ later on to emit events to. Use the event stream type `Basic Event Stream` and provide the credential of the same type you just made. After creation it will generate a unique URL that _Terraform Actions_ will send its events to and is protected, so you also need the _username_ and _password_ of the credential. Suggestion for a name: "Terraform Actions".

### Rulebook Activation
This defines and runs a listener for the events on the generated URL. Use the rulebook from the project you just created. The credential is the one for the controller. For `Event Stream` you need to make something called a _mapping_ which maps the source definition to this stream. There is only one mapping available so that is easy. Name suggestion: "Terraform Actions Listener"

> Set the log level to debug which would help you when things don't work.

