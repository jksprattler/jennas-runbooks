# Azure AD & RBAC with Terraform Part 2

_Last updated: January 31, 2023_

![bazure-tf-ad-rbac-pt2.png](/images/bazure-tf-ad-rbac-pt2.png)

## Overview

After publishing my [initial runbook](https://jksprattler.github.io/jennas-runbooks/Azure/azure-tf-ad-rbac.html) exploring this topic, I decided to test out and implement the [HashiCorp](https://developer.hashicorp.com/terraform/tutorials/azure/azure-ad) `for_each` meta-argument method for managing the Azure AD User base of a production environment I'm currently working with. I wanted to share my findings from that experience here. In this 2nd part of my series on Azure AD & RBAC with Terraform, I define the requirements necessary for setting up Azure AD User and Group administration using this alternative method. I've also highlighted some tips from issues I ran into during my implementation of this and a Validation section containing helpful commands for post-checkouts and troubleshooting. While the initial runbook and demo in this series still contains useful information such as a deep dive on the security behind Azure AD and RBAC and some test login scenarios, I've found this CSV file method to be much more efficient at managing users and group membership. Also, I've refined the security privileges around the GitHub Actions SPN and have implemented the creation and management of the SPN, including its API permissions, using Terraform resource blocks.

### Topics Covered:

- GH Actions SPN creation and permission configurations done in Terraform
- GH Actions SPN gets further locked down with reduced privileges from Subscription Owner to Reader and from User Admin to specific API Reader level permissions
- Azure AD Users managed by a Terraform for_each meta-argument using data populated into a CSV file
- Azure AD Group members assigned using Terraform for_each meta-argument against users assigned to a specific Department

### Pre-requisites

- See Part 1 runbook: [Azure AD & RBAC with Terraform](https://jksprattler.github.io/jennas-runbooks/Azure/azure-tf-ad-rbac.html#pre-requisites)
- Ensure Security defaults are disabled in order to enable the creation of a Conditional Access policy. 
  - From the Portal UI navigate to Azure AD > properties > manage security defaults > enable security defaults toggle to no.
- Not required: if you're interested in how I got the highlighted changes in my GH Actions bot PR comments see my runbook: [Silence "Refreshing state…" & Highlight Changes in Github Actions Terraform Plan Output](https://jksprattler.github.io/jennas-runbooks/DevOps/CI-CD/ghactions-silence-refreshing-diff.html)

### Design Artifacts

- YouTube [demo recording](https://youtu.be/cIqvYL9burQ)
- Clone/Fork the repo containing the Terraform artifacts for the Azure AD config here: [jksprattler/azure-security](https://github.com/jksprattler/azure-security)
  - Relevant files are under `/azuread-users-groups-roles-pt2`
- Config for the GH Actions SPN, storage account and blob container hosting the terraform backend state files is under `/azure-dev-infra`

### Procedure

1. Authenticate to Azure and configure local environment variables in order to run terraform commands from your local terminal:
```script
az login
export ARM_SUBSCRIPTION_ID=$(az account show --query id | xargs)
export ARM_ACCESS_KEY="<insert storage account access key used for backend state configs>"
```
For enterprise environments with multiple subscriptions, run the following to get the specific SubID you need: `az account list --query '[].{SubID:id, SubName:name}' -o table`
2. Create the `gh-actions-runbooks-ad` SPN and its required API permissions configured in the `apps.tf` file under the `azure-dev-infra` directory. This will be used to perform the terraform plan output in the GitHub PR comments. This config resides under this directory since the SPN is required for all other infrastructure plan outputs in the other directories of the repo. 
3. Once created, apply the "Grant admin consent for Default Directory" to the API permissions for the SPN by running the az CLI command: `az ad app permission admin-consent --id <app_id value from outputs>`
4. Assign the SPN outputs as GitHub Secrets in your repo using the following variables:
```scss
ARM_SUBSCRIPTION_ID="<sub_id>"
ARM_TENANT_ID="<tenant_id>"
ARM_CLIENT_ID="<app_id>"
ARM_CLIENT_SECRET="<auth_client_secret>"
```
```tip
In the previous runbook, step 4. mentioned assigning the SPN the User Admin AD role however, I found this is not required with the API Permissions configured to allow AD User, Group and Domain Reader access. The Subscription Owner RBAC role is not needed since we can assign the Subscription Reader RBAC role as we just need it to be allowed to read the data during the terraform plan. 
```
5. Populate the `users.csv` file with users. If you're importing an existing Azure AD user base into Terraform, navigate in the Portal UI to Azure AD > Users : Download users to capture a csv file of existing users. Extract the user data from the existing CSV file and populate the `users.csv` Terraform file with the required fields: `first_name,last_name,mail_nickname,preferred_language`
```tip
The `usage_location` value is required for users that are assgined Microsoft licenses such as O365
```
6. Assign the users to a `department` such as Art or Engineering if you'd like to auto-assign them to the Azure AD Groups created in this lab. The Art group uses Dynamic Membership to assign users requiring the Azure AD Premium P1 license. The Art group also includes a Conditional Access policy assignment enforcing MFA into the portal for all users in the group. The Engineering group resource block includes a for_each argument which loops through all users and for each user assigned to the Engineering department it assigns their membership to the Engineering group. The Engineering group does not require any Azure AD Premium licenses and is a nice option for automating group membership when you want to keep costs down and don't require the use of conditional access policies.
7. Save and commit the changes. Review the `github-actions` bot output in your PR comments which uses the `gh-actions-runbooks-ad` SPN created in Step 2. Make adjustments to your code as needed.
8. Perform a manual terraform apply from your local directory.
```tip
- Terraform will assign each Azure AD user resource a unique ID using the `mail_nickname` value set in the `users.csv` file. This identifier can be used for assigning Azure AD group owner/membership. 
- User Principal Names are used for the Azure Portal login username and will require a password reset upon initial login. The new user will be created with an auto-generated password using the following pattern in all lowercase letters in a single string: `lastname + first letter of first name + numerical value for length of first name + !123`
  - I added chars `123` as I found the Microsoft password length requirement was not met on users with last names less than 5 chars long. This should fix that issue.
```
9. To delete users, remove the entire line entry for that user from the `users.csv` file and be sure to remove any user identifiers (i.e., `azuread_user.users["userarose"]`) manually assigned to Azure AD groups.

### Validations

- List resources managed by Terraform: `terraform state list`
- Show AD user info: `terraform state show 'azuread_user.users["userarose"]'`
- List all Azure AD users: `az ad user list --query "[].{name:displayName,userPrincipalName:userPrincipalName, ObjectID:id}" -o tsv`
- List the 2 groups that were created: `az ad group list --query "[?contains(displayName,'Engineering')].{ name: displayName }" -o tsv`
- List the users in the groups: `az ad group member list --group "Engineering" --query "[].{ name: displayName }" -o tsv`

### Conclusion

You now have much more efficient method for managing Azure AD Users using the CSV file and Terraform `for_each` meta-arguments. Azure AD Group membership is dynamically configured using `for_each` meta-arguments against department names assigned to users which excludes the requirement for purchasing Azure AD Premium P1 licenses. The SPN used by the Github Actions workflow is further locked down to adhere to the principal of least privileges and it's configured in the Terraform code. Having your AD users, groups and SPN's configured as code allows for consistency of settings and permissions across your infrastructure. It also increases a level of security awareness since PR's will require review/approval for code changes.
