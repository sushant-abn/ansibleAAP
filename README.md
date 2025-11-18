# Welkom to the Ansible part of the workshop

## The basics
- Where is the Ansible Automation Server: https://caap.fvz.ansible-labs.de
- What is my username: the part before the @ of the email address you gave us
- What is my password: it's on the whiteboard in the workshop room

Ansible Automation Platform (AAP) has the notion of "Organisations". For this workshop an Organisation named TechXchangeNL has been made and your account is part of that. Everything you do you will do within that organisation.

This workshop has been designed such that you will need to do most of the work, signifying the word "work" in workshop ;-) This means we only have the absolute basics set up and you need to build all the components to make everything work.

The division of work between Hashicorp Terraform Cloud and Ansible Automation Platform (as likely lready explained to you) is:
- Terraform: Building up and changing infrastructure in the cloud, among which are RHEL10 servers
- AAP: Configure the RHEL10 servers to become a webserver serving a website

This means Terraform has all the credentials needed to do it's thing in the cloud of choice (AWS).
AAP will use the integrations with Terraform to be able to do it's thing on the provisioned infrastructure. AAP has no credentials for the cloud!

So what are the basics we have set up for you:
1. A machine credential named RHEL. You use this machine credential to be able to run your playbooks on the providioned servers.
2. A Custom Credential _Type_ called "Hashicorp Terraform Cloud". You use this later to create your credential.
3. A token to be able to do stuff in Hashicorp Terraform Cloud. This token is available at: <TBC>

## Building Blocks
First, you need to create some building blocks.

note: wherever you can and/or need to specify an Organisation, choose "TechXchangeNL'.

### Project
You need to create a project in AAP. The project is your repository with playbooks.
- The URL of the git repo with the platybooks for this workshop: https://github.com/TechXchangeNL/ansible.git

### Credentials
Apart from the already available machine credential, you need a few more..

- A Credential to be able to communicate with Hashicorp Terraform Cloud. Use Credential Type "Hashicorp Terraform Cloud" and the provided token.
- A Credential to be able to sync the Terraform State File that will be used for the inventory source. Choose the credential type "Terraform backend configuration". In the backend configuration field enter the following:

```text
hostname = "app.terraform.io"  
organization = "TechXchangeNL"  
token = "YOURTOKENHERE"  
workspaces { name = "YOURWORKSPACE" }  
```

For token, enter the token provided
For workspace anter the workspace you made in Terraform (you did...right?)



