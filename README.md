# DeployingAppService
A writeup of how to overcome some of the challenges of deploying and provisioning Microsoft App Services
by Michael Braun

## The Challenge
After seeing many of the very interesting vulnerable by design projects for AWS, such as [CloudGOAT](https://github.com/RhinoSecurityLabs/cloudgoat), I noticed there were no real equivalents in Azure. I saw this as an opportunity to challenge myself to learn a bit more about Terraform and Azure. The first release of the VulnerableAzure project can be found [here](https://github.com/metalstormbass/VulnerableAzure). All of the code in this document comes from that repository.

After doing some initial research on some scenarios, I decided that the first version should have the following components:
<br>
1. Publically Exposed Blob Storage <br>
2. Azure Kubernetes Service with a Vulnerable container image <br>
3. Azure App Service with a terrible Django app. [Located Here](https://github.com/metalstormbass/VulnerableWebApp) <br>

The reason I chose these three components, is I though they would be an interesting way to explore attacking and defending Azure Services. 

## The Problem
While all of the components were not to difficult to build (with some research), I came accross an interesting limitation in Terraform. From what I could tell, you can build the app service with Terraform, but you cannot provision the app. 

## The Solution
I wanted a solution that did not involve running anything on my computer. So after some brainstorming, I decided this would be a good opportunity to use Github Actions to orchestrate the commands. 

### Step 1 - Configure Github Actions to run Terraform
Initially, I was using [Terraform Cloud](https://terraform.io) to run the playbooks. I connected Terraform Cloud to Github, so that every commit to Github would trigger a Terraform plan. The first challenge was to figure out how to transfer this task to Github Actions. To resolve, I found the [workflow YAML for Github Actions](https://www.terraform.io/docs/github-actions/setup-terraform.html). I modified it to look like this: 

```bash
#Github Actions to trigger Terraform Cloud and provision using AZ
name: 'Deploy Action'

on:
  push:
    branches:
    - master
  

jobs:
  terraform:
    name: 'Terraform'
    runs-on: ubuntu-latest

    # Use the Bash shell regardless whether the GitHub Actions runner is ubuntu-latest, macos-latest, or windows-latest
    defaults:
      run:
        shell: bash

    # Checkout the repository to the GitHub Actions runner
    steps:
    - name: Checkout
      uses: actions/checkout@v2

    # Install the latest version of Terraform CLI and configure the Terraform CLI configuration file with a Terraform Cloud user API token
    - name: Setup Terraform
      uses: hashicorp/setup-terraform@v1
      with:
        cli_config_credentials_token: ${{ secrets.TERRAFORM }}

    # Initialize a new or existing Terraform working directory by creating initial files, loading any remote state, downloading modules, etc.
    - name: Terraform Init
      run: terraform init

    # Generates an execution plan for Terraform
    - name: Terraform Plan
      run: terraform plan

      # On push to master, build or change infrastructure according to Terraform configuration files
      # Note: It is recommended to set up a required "strict" status check in your repository for "Terraform Cloud". See the documentation on "strict" required status checks for more information: https://help.github.com/en/github/administering-a-repository/types-of-required-status-checks
    - name: Terraform Apply
      if: github.ref == 'refs/heads/master' && github.event_name == 'push'
      run: terraform apply -auto-approve
```
Please note, the ${{ secrets.TERRAFORM }} is a Terraform API token that is stored in as a Secret in the repository.

Once I had this set as my Github Action, I had to edit my Terraform playbook to point it to the Terraform Cloud workspace. I added this [main.tf](https://github.com/metalstormbass/VulnerableAzure/blob/master/main.tf).

```bash
#This info is required for Github Actions to trigger the Terraform Cloud Deployment

terraform {
      backend "remote" {
         # The name of your Terraform Cloud organization.
         organization = "MikeNet"

         # The name of the Terraform Cloud workspace to store Terraform state files in.
         workspaces {
           name = "VulnerableAzure"
         }
       }
     }
```
The final peice of this is to sever the connection between Terraform Cloud and Github. This will allow GitHub Actions to plan and apply the changes.

After creating this configuration, Terraform runs on Github Action, but the variables and state file are stored in Terraform Cloud. This way, I can configure the variables and secrets in Terraform Cloud and don't have to worry about hard coding them into the Github actions workflow. Also, because the state file is stored in Terraform cloud, I can trigger the destroy from there.

### Step 2 - Gather information from Terraform for provisioning
The next challenge
