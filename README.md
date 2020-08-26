# DeployingAppService
A writeup of how to overcome some of the challenges of deploying and provisioning Microsoft App Services
by Michael Braun

## The Challenge
After seeing many of the very interesting vulnerable by design projects for AWS, such as [CloudGOAT](https://github.com/RhinoSecurityLabs/cloudgoat), I noticed there were no real equivalents in Azure. I saw this as an opportunity to challenge myself to learn a bit more about Terraform and Azure. The first release can be found [here](https://github.com/metalstormbass/VulnerableAzure).

After doing some initial research on some scenarios, I decided that the first version should have the following components:
<br>
1. Publically Exposed Blob Storage <br>
2. Azure Kubernetes Service with a Vulnerable container image <br>
3. Azure App Service with a terrible Django app. [Located Here](https://github.com/metalstormbass/VulnerableWebApp) <br>

The reason I chose these three components, is I though they would be an interesting way to explore attacking and defending Azure Services. 

## The Problem
While all of the components were not to difficult to build (with some research), I came accross an interesting limitation in Terraform. From what I could tell, you can build the app service with Terraform, but you cannot provision the app. So after some brainstorming, I decide this would be a good opportunity to use Github Actions to orchestrate the commands.


