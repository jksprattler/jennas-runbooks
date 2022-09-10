# Azure AD & RBAC with Terraform

_Last updated: September 10, 2022_

## Overview

The purpose of this runbook is to demonstrate a potential approach to managing Azure AD users/groups and Role-Based Access Control (RBAC) following a declarative model by using Terraform and GitHub Actions CI/CD Workflows. Both the [Azure AD](https://registry.terraform.io/providers/hashicorp/azuread/latest/docs) and [Azure RM](https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs) Terraform providers will be used to implement IAM as code which will allow for automated provisioning of users, groups and role assignments. The principal of least privilege will be followed in delegation of both Azure AD manually through the Portal UI and Azure RBAC assignments programmatically through Terraform. Users will be assigned to AD Groups dynamically based on department (ie Art, Engineering). A conditional access policy will be created to demonstrate Multi-factor authentication (MFA) based on dynamic group assignment. Finally, Azure AD self-service password reset (SSPR) will be enabled for all users. An Azure Service Principal will be used to perform the automated GitHub Actions workflow jobs generating terraform output based on code commits.

### Azure AD & RBAC Topics Covered:

- Azure Service Principal (SPN) to provision terraform configuration via GitHub Actions automation
- Azure AD identity governance of users and groups (dynamic)
- Azure RBAC group assignment using both built-in and custom roles 
- Azure RBAC assignments scoped to dynamic Azure AD groups
- Azure AD Conditional Access Policy enforcing Multi-factor authentication (MFA) for Users based on dynamic Group membership
- Azure AD self-service password reset (SSPR)

### Pre-requisites

- GitHub Account
- Microsoft Account
- Azure Subscription
- Azure AD Premium P1/P2 license required for configuring: Conditional Access policy, MFA with conditional access, Dynamic groups
  - A free trial can be activated from within our Azure AD tenant for a limited number of days
- Azure [CLI](https://docs.microsoft.com/en-us/cli/azure/) installed and credentials for [authentication](https://docs.microsoft.com/en-us/cli/azure/authenticate-azure-cli)
- Terraform [installed](https://learn.hashicorp.com/tutorials/terraform/install-cli)
- Azure RG, Storage account and blob container setup if you choose to maintain terraform state files remotely.

### Design Artifacts

- YouTube [demo recording FIXME]()
- Clone/Fork the repo containing the Terraform artifacts for the Azure AD config here: [jksprattler/azure-security](https://github.com/jksprattler/azure-security)
  - Relevant files are under `/azuread-users-groups-roles`
  - Config for the storage account and blob container hosting the terraform backend state files is under `/azure-dev-infra`

### Diagram

![azuread-rbac-diagram.png](/images/azuread-rbac-diagram.png)

### Procedure

1. Authenticate to Azure and configure local environment variables in order to run terraform commands from your local terminal:
```scss
az login
export ARM_SUBSCRIPTION_ID=$(az account show --query id | xargs)
export ARM_ACCESS_KEY="<insert storage account access key used for backend state configs>"
```
2. Create an Azure Service Principal (SPN) as your identity for provisioning the terraform confgiuration using the GH Actions automation workflows:
```scss
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
4. Assign the User administrator role to the SPN. From Portal UI navigate to Azure AD > Roles and Administrators blade > "User administrator" Role > Add Assignments > Select members > Filter by service principal display name ***Note as of this writing I couldn’t find an efficient CLI method for applying Azure AD roles to SPN's as Azure CLI is unsupported and Powershell cmdlets, which are still in preview mode, gave me errors so portal seems to be the best option.
5. Disable Security defaults in order to enable the creation of a Conditional Access policy. From Portal UI navigate to Azure AD > properties > manage security defaults > enable security defaults toggle to no
6. Create a new Azure AD user by making a new local branch from the cloned [jksprattler/azure-security](https://github.com/jksprattler/azure-security) repo. Run the `scripts/azuread-create-users.py` script to generate the terraform syntax for your new user: `python scripts/azuread-create-users.py`
- The script will run an az cli command invoking a custom request through Microsoft Graph in order to capture your Azure AD default domain using your current login-credentials. 
7. Navigate to the `azuread-users-groups-roles` directory and paste the terraform code for your new user into the `main.tf` file using either of the existing Engineering or Art AD Groups or create a new group. For example:
```scss
resource "azuread_user" "raybrown" {
  user_principal_name   = "raybrown@jennasrunbooks.com
  display_name          = "Ray Brown"
  department            = "Art"
  password              = "Super$ecret01@!"
  force_password_change = true
}
```
8. Save and commit the changes. Review the `github-actions` bot output from your PR, specifically the terraform plan results, which will perform the following functions on your behalf using the Azure SPN (ie `gh-actions-runbooks-ad`):
![azuread-gh-actions.png](/images/azuread-gh-actions.png)
9. From the `azuread-users-groups-roles` directory, perform your `terraform apply` to create the new Azure AD user.
10. Enable Self Service Password Reset (SSPR) for All users. From Portal UI navigate to:  Azure AD > Password reset > Auth methods > SSPR enabled: All

### Summary

Upon completion of the above procedure, you should now have a basic architecture started for implementing Azure Identity & Access Management as code using Azure AD and Azure RM Terraform providers. A CI/CD pipeline is implemented using a Github Actions workflow automation in order to provide the output of Terraform format/init/validate/plan results based on PR commits providing a solution for code review by your team prior to actualy implementation by running a `terraform apply` locally. Python scripts are available for generating Terraform syntax for new Azure AD users (`azuread-create-users.py `) and also for extracting a list of existing Azure AD users which then generates the terraform code for the list of existing users to be imported into your terraform state (`azuread-import-users.py`). While this approach might be a good initial start for managing your Azure IAM as code, alternatively I've included a link in the references section below to another more elegant solution by HashiCorp by using a for_each loop with a CSV file of users.

### References

- [Enable Azure AD Multi-Factor Authentication - Microsoft Entra Microsoft Docs](https://docs.microsoft.com/en-us/azure/active-directory/authentication/tutorial-enable-azure-mfa)
- [Enable Azure Active Directory self-service password reset - Microsoft Entra Microsoft Docs](https://docs.microsoft.com/en-us/azure/active-directory/authentication/tutorial-enable-sspr)
- [Automate Terraform with GitHub Actions Terraform - HashiCorp Learn](https://learn.hashicorp.com/tutorials/terraform/github-actions)
- Future improvements: Terraform for_each loop through a list of users defined in a CSV as outlined by [HashiCorp](https://learn.hashicorp.com/tutorials/terraform/azure-ad?in=terraform/azure) to improve your identity governance strategy as your user base grows increasing in complexity of management on a per user basis.
