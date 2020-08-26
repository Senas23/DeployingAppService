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
I wanted a solution that did not involve running anything on my computer. Furthermore, I wanted it to be "templatetized" so that nothing needed to be hardcoded. So after some brainstorming, I decided this would be a good opportunity to use Github Actions to orchestrate the commands. 

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
Please note, the TERRAFORM is a Terraform API token that is stored in as a Secret in the repository.

Once I had this set as my Github Action, I had to edit my Terraform playbook to point it to the Terraform Cloud workspace. I added this [main.tf](https://github.com/metalstormbass/VulnerableAzure/blob/master/main.tf). The one annoyance was that I could not use variables for this peice. For whatever reason, Terraform will not allow this.

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
The next challenge was how to gather the contents of a Terraform variable and pass it to the AZ command to provision the application. I decided that the easiest way to do this would be a REST API call to Terraform Cloud. Documentation for that is located [her](https://www.terraform.io/docs/cloud/api/variables.html. <br>

Here is the code that I came up with:

```bash
   #API Call to Terraform.io to get app-services name
    - name: Get App Variable Name
      run: |
           out=$(curl --header "Authorization: Bearer ${{ secrets.TERRAFORM }}" --header "Content-Type: application/vnd.api+json" ${{ secrets.TF_ENV }} | jq -c --arg key "victim-company" '.data[].attributes | select (.key=="victim_company") | .value')
           out=$(echo $out | tr -d \")
           out1="$out-app-service"
           out2="$out-rg"
           echo ::set-env name=APP_NAME::$(echo $out1)
           echo ::set-env name=RG::$(echo $out2)
```

The first thing to note, is that I am passing two variables to my cURL command:<br>
TERRAFORM: This is the Terraform API key.  <br>
TF_ENV: This is the workspace URL. Example: https://app.terraform.io/api/v2/workspaces/INSERT_WORKSPACE_ID_HERE <br>

This API call lists all of the variables in the workspace. I then used JQ to find the correct value. The output value had to then be manipulated to match the naming convention I defined withing my terraform files. FInally, I set the two final values as environment variables that I could then pass on to the next step. 

### Step 3 - Use AZ to provision AppServices
The final step was to provision the application through AZ. This was the most challenging step for me. Here is the code:

```bash
#Use AZ to provision webapp     
    - name: Deploy app using AZ
      run: |
           #Login to Azure
           az login  --service-principal -u ${{ secrets.AZ_ID }} -p ${{ secrets.AZ_SECRET }} -t ${{ secrets.AZ_TENANT }}
           az webapp deployment source config --name ${{ env.APP_NAME}} --resource-group ${{ env.RG }} --repo-url https://github.com/metalstormbass/VulnerableWebApp.git --branch master --manual-integration                     
           az webapp config set -g ${{ env.RG }} -n ${{ env.APP_NAME}} --startup-file /home/site/wwwroot/VulnerableWebApp/startup.sh
```
First, I needed to login with AZ. The variables are:<br>
AZ_ID = Azure Client ID<br>
AZ_SECRET = Azure Client Secret<br>
AZ_TENANT = Directory(Tenant) ID<br>

Then, I used the environment variables defined in Step 2 to confiure the source of my AppService. I pointed it to the repository for my terrible web app. This could easily be modified to be a variable if required.

The last line of code is configurign the app to reference a startup script. After a lot of testing a learned that app services is just a docker instance that exposes the internal port 8000 and acts as a NGINX equivalent to pass incomming external traffic on port 80 to port 8000. It is important to note, the startup.sh file lives in the web application repository.

Here is the code:
```bash
python -m  pip install -r /home/site/wwwroot/VulnerableWebApp/requirements.txt

gunicorn --bind=0.0.0.0:8000 --chdir=/home/site/wwwroot/VulnerableWebApp/ VulnerableWebApp.wsgi
```
While this seems simple, it took me quite a while to figure out it out. You cannot run pip on it's own and you have to be very specific with the file paths.

### Testing
To do any testing you have to commit a change to the repository that contains the Terraform files. I did this by just editing comments in the [workflow file](https://github.com/metalstormbass/VulnerableAzure/blob/master/.github/workflows/terraform.yml). <br>
<b> Be Aware that every commit will trigger a Terraform Apply</b><br>

I did learn (afterwards, unfortunately) that you can do empty commits by using this command:
```bash
git commit --allow-empty
```
## Conclusion
While this may not be the most elegant solution, I was happy to have solved the problem. It was a very good learning excerise and a tangential benefit was that I now had a CI/CD component to the VulnerableAzure project. This is an additional area where protections can be applied. Hopefully Terraform will add the provisioning functionality to make this process simpler. 
