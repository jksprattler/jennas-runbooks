# Azure AD & RBAC with Terraform

_Last updated: January 29, 2023_

## Overview

The purpose of this runbook is to demonstrate a potential approach to managing Azure AD users, groups and Role-Based Access Control (RBAC) by following Terraform's declarative model with automated checkouts using GitHub Actions CI/CD Workflows. Both the [Azure AD](https://registry.terraform.io/providers/hashicorp/azuread/latest/docs) and [Azure RM](https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs) Terraform providers will be used to implement Identity & Access Management as code which will allow for automated provisioning of users, groups, custom role definitions, role assignments and conditional access policies. The principal of least privilege will be followed in delegation of both Azure AD and Azure RBAC assignments. The procedure will use a combination of AZ CLI, Terraform, python scripts and Azure Portal UI. Users will be assigned to AD Groups dynamically based on their department (ie Art, Engineering). A conditional access policy will be created to demonstrate Multi-factor authentication (MFA) enforcement based on dynamic group assignment. An Azure Service Principal (SPN) will be used to perform the automated GitHub Actions workflow jobs based on code commits generating terraform checks and plan output. Finally, Azure AD self-service password reset (SSPR) will be enabled for all users.

### NOTE
Please see the newest [Part 2 runbook](https://jksprattler.github.io/jennas-runbooks/Azure/bazure-tf-ad-rbac-pt2.html) further exploring this topic where I implemented the [HashiCorp](https://developer.hashicorp.com/terraform/tutorials/azure/azure-ad) `for_each` meta-argument for managing the Azure AD Users.

### Azure AD & RBAC Topics Covered:

- Azure AD identity governance of users and groups (dynamic)
- Azure RBAC group assignment using both built-in and custom roles 
- Azure RBAC assignments scoped to dynamic Azure AD groups
- Azure AD Conditional Access Policy enforcing Multi-factor authentication (MFA) for Users based on dynamic Group membership
- Azure Service Principal (SPN) to provision terraform configuration via GitHub Actions automation
- Azure AD self-service password reset (SSPR)

### Pre-requisites

- GitHub Account
- Microsoft Account
- Azure Subscription
- Azure AD Premium P1/P2 license required for configuring: Conditional Access policy, MFA with conditional access, Dynamic groups
  - A free trial can be activated from within your Azure AD tenant for a limited time
- Azure [CLI](https://docs.microsoft.com/en-us/cli/azure/) installed and credentials for [authentication](https://docs.microsoft.com/en-us/cli/azure/authenticate-azure-cli)
- Terraform [installed](https://learn.hashicorp.com/tutorials/terraform/install-cli)
- Azure RG, Storage account and blob container setup if you choose to maintain terraform state files remotely.

### Design Artifacts

- YouTube [demo recording](https://youtu.be/nVf5pYGeNTc)
- Clone/Fork the repo containing the Terraform artifacts for the Azure AD config here: [jksprattler/azure-security](https://github.com/jksprattler/azure-security)
  - Relevant files are under `/azuread-users-groups-roles`
  - Config for the storage account and blob container hosting the terraform backend state files is under `/azure-dev-infra`

### Diagram

![azuread-rbac-diagram.png](/images/azuread-rbac-diagram.png)

### Procedure

1. Authenticate to Azure and configure local environment variables in order to run terraform commands from your local terminal:
```script
az login
export ARM_SUBSCRIPTION_ID=$(az account show --query id | xargs)
export ARM_ACCESS_KEY="<insert storage account access key used for backend state configs>"
```
2. Create an Azure SPN assigning the RBAC role of owner at the subscription level so it has privileges to both manage all resources and assign RBAC roles to other users. This identity will be used for provisioning the terraform configuration using the GH Actions workflows:
```script
az ad sp create-for-rbac --name "gh-actions-runbooks-ad" --role owner \
                         --scopes /subscriptions/{subscription-id} \
                         --sdk-auth                        
# Replace {subscription-id} with the subscription details
# The command should output a JSON object similar to this:
{
  "clientId": "<GUID>",
  "clientSecret": "<GUID>",
  "subscriptionId": "<GUID>",
  "tenantId": "<GUID>",
  (...)
}  
```
3. Assign the output of the JSON objects as GitHub Secrets in your repo using the following variables:
```scss
ARM_SUBSCRIPTION_ID="<azure_subscription_id>"
ARM_TENANT_ID="<azure_subscription_tenant_id>"
ARM_CLIENT_ID="<service_principal_appid>"
ARM_CLIENT_SECRET="<service_principal_password>"
```
4. Assign the User administrator Azure AD role to the SPN. From Portal UI navigate to Azure AD > Roles and Administrators blade > "User administrator" Role > Add Assignments > Select members > Filter by service principal display name
```tip
As of this writing I couldn’t find an efficient CLI method for applying Azure AD roles to SPN's as Azure CLI is unsupported and Powershell cmdlets, which are still in preview mode, gave errors leaving the portal as the best option.
```
5. Assign the API permissions from the below screenshot to the SPN to allow read/write access to the conditional access policy. From Portal UI navigate to App registrations > locate and select your SPN > API permissions: Add a permission and be sure to select "Grant admin consent for Default Directory" once all the Application type API permissions have been added.
![gh-actions-runbooks-ad-api-permissions.png](/images/gh-actions-runbooks-ad-api-permissions.png)
6. Disable Security defaults in order to enable the creation of a Conditional Access policy. From the Portal UI navigate to Azure AD > properties > manage security defaults > enable security defaults toggle to no
7. Create a new Azure AD user by making a new local branch from the cloned [jksprattler/azure-security](https://github.com/jksprattler/azure-security) repo. Run the `azuread-create-users.py` script to generate the terraform syntax for your new user: `python scripts/azuread-create-users.py`
- The script will run an az cli command invoking a custom request through Microsoft Graph in order to capture your Azure AD default domain using your current az login. 
8. Navigate to the `azuread-users-groups-roles` directory and paste the terraform code for your new user into the `main.tf` file using either of the existing Engineering or Art AD Groups or create a new group. For example:
```script
resource "azuread_user" "raybrown" {
  user_principal_name   = "raybrown@jennasrunbooks.com"
  display_name          = "Ray Brown"
  department            = "Art"
  password              = "Super$ecret01@!"
  force_password_change = true
}
```
9. Save and commit the changes. Review the `github-actions` bot output from your PR, specifically the terraform plan results, which will perform the following functions on your behalf using the Azure SPN (ie `gh-actions-runbooks-ad`):
![azuread-gh-actions.png](/images/azuread-gh-actions.png)
10. From the `azuread-users-groups-roles` directory, perform your `terraform apply` to create the new Azure AD user.
11. Enable Self Service Password Reset (SSPR) for All users. From Portal UI navigate to:  Azure AD > Password reset > Auth methods > SSPR enabled: All

#### Import Existing Azure AD Users into Terraform
If you have a number of users that already exist in your Azure AD and are looking to start managing this part of your cloud estate using Terraform, you can run the `scripts/azuread-import-users.py` script which will extract a list of your current Azure AD user's Display Names, Principal Names and Departments associated with the current Azure tenant you are logged into (`az login`). The script runs an az ad query capturing the user details and copies them to a tsv file which is then read by python and converted into Terraform syntax. Once you have your list of users, follow the Procedure above starting at step 8.
```script
python scripts/azuread-import-users.py
# For much larger lists of users, save the python output to either a txt or tf file
python scripts/azuread-import-users.py > users.tf
```

### Summary

Upon completion of the above procedure, you should now have a basic architecture started for implementing Azure IAM as code using Azure AD and Azure RM Terraform providers. A CI/CD pipeline is implemented using a Github Actions workflow generating Terraform format/init/validate/plan results based on PR commits providing a solution for code review by your team prior to invoking `terraform apply` locally. Python scripts are available for generating terraform syntax for new Azure AD users (`azuread-create-users.py `) and for extracting a list of existing Azure AD users which then generates the terraform code for the list of existing users to be imported into your terraform state (`azuread-import-users.py`). While this approach might be a good initial start for managing your Azure IAM as code, alternatively there's a link in the references section below to another more elegant solution to try provided by HashiCorp which uses a for_each loop against a CSV file of users.

### References

- [Enable Azure AD Multi-Factor Authentication - Microsoft Entra Microsoft Docs](https://docs.microsoft.com/en-us/azure/active-directory/authentication/tutorial-enable-azure-mfa)
- [Enable Azure Active Directory self-service password reset - Microsoft Entra Microsoft Docs](https://docs.microsoft.com/en-us/azure/active-directory/authentication/tutorial-enable-sspr)
- [Automate Terraform with GitHub Actions Terraform - HashiCorp Learn](https://learn.hashicorp.com/tutorials/terraform/github-actions)
- Future improvements: Terraform for_each loop through a list of users defined in a CSV as outlined by [HashiCorp](https://learn.hashicorp.com/tutorials/terraform/azure-ad?in=terraform/azure) to improve your identity governance strategy as your user base grows increasing in complexity of management on a per user basis.
