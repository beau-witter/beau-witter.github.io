---
layout: post
title:  "Manage Service Principal Credentials Automatically"
date:   2024-01-24 18:00:00 -0700
categories: [Automation, Azure, Azure DevOps, Credentials, PowerShell]
---

## Goal

Automatically refresh credentials for Service Principals in Azure.

## Restrictions

We do not have the ability to grant Admin Consent for API Permissions (Microsoft Graph)
We can’t run the automated rotation from user credentials as any given individual could leave and break the process

## 10,000 Foot View

A Service Principal that is the facilitator or manager of all other Service Principals
An Azure DevOps Pipeline to execute the credential refresh on a schedule
A PowerShell script that performs the credential refresh

## Service Principal

Due to the restriction of not being allowed to grant Admin Consent for API Permissions, which are needed to allow the Service Principal the ability to alter other Service Principals’ credentials, we need to request permission to our Azure Administration Group for a single Service Principal to be granted the Application.ReadWrite.OwnedBy API Permission as that is the least amount of privileges required to perform our goal without having access to every Service Principal.

![Application.ReadWrite.OwnedBy is an Application type permission that requires Admin consent to assign and is needed to manage the Service Principals this one owns.](/images/posts/2024-01-24-manage-service-principal/img1.PNG)

Once we have that Service Principal with permissions, we needed to set it as an Owner for each Service Principal it was meant to manage. This cannot be done through the Azure Portal as it only supports adding users, so it must be done through the CLI. The following cmdlet requires the AzureAD PowerShell module to be installed to work as well as connecting to Azure using Connect-AzAccount.

```
Add-AzureAdServicePrincipalOwner `
  -ObjectId 00000000-0000-0000-0000-00000000000 `
  -RefObjectId XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXX
```

Where the -ObjectId, shown as 0s here, is the Azure ObjectId of the Service Principal that we are going to set an owner on and the -RefObjectId, shown as Xs here, is the Azure ObjectId of the Service Principal that is going to be set as the Owner (the managing Service Principal).

To make this whole process transparent to users, we persist the secrets to their respective Key Vaults and allow them access to the secrets there when required to utilize the Service Principals. Since they do not reference the secrets directly, and instead only reference the Key Vault Secrets, we can rotate the credentials at any point and it can be handled by simply requesting the Secret value from the Key Vault again. Make sure that the managing Service Principal has appropriate access on the relevant Key Vaults, namely List, Get, Set, and Delete Secrets.

## Azure DevOps

There are a few things that need to be setup in Azure DevOps to ensure that the way we are running the pipeline for rotating credentials has sufficient access to run properly. First, create the Service Connection in your project that corresponds to the Service Principal that will have the ability to alter other Service Principals’ credentials. Then, in the option stack button, click Security for that Service Connection.

![Where the Security Option is located for Service Connections.](/images/posts/2024-01-24-manage-service-principal/img2.PNG)

This service connection needs to be added to the project’s “Endpoint Administrators” group (which looks like “[PROJECT_NAME]\Endpoint Administrators”) so that the service connection the script is running under has permissions to change Azure DevOps endpoints like Service Connections. Also, the Organization’s “Project Collection Build Service Accounts” group needs to be added as a member of the same “[PROJECT_NAME]\Endpoint Administrators” group.

## PowerShell Script

See [this gist](https://gist.github.com/beau-witter/7f99fcc947680e7e0fe0b47b5a8c4a32) for the PowerShell function code and Azure DevOps pipeline yaml directly but continue reading on to understand the goals of the PowerShell script.

The script’s goal is to reset, or effectively rotate, the credentials for a given Service Principal. It can also, optionally, handle updating a corresponding Service Connection in Azure DevOps (if provided) and can also handle if the name of the Service Credential needs to change. It first checks to see if you are changing the name, and if so it removes the old secrets in the Azure Key Vault. Then it regenerates the credential and persists it to the specified Azure Key Vault. Lastly, it checks if there was an associated Service Connection for Azure DevOps specified. If there was one specified, it will send a request that updates just the Service Connection’s Service Principal Password.

## Conclusion

Now that everything is setup, we can create a yaml pipeline in Azure DevOps that uses the “AzureCLI” Task to run the script specified above, making sure to include the “addSpnToEnvironment” if the Service Connection update feature is being used. We can schedule this pipeline to run at whatever cadence desired, 11 months to let the credentials stay the same as long as possible, or much shorter, like a week, if required or desired. We can also take advantage of the “parameters” section to input any or all Service Principals or Service Principal and Service Connections that we want to rotate credentials for.

Note: There is currently an issue with rotating the credentials of the Service Principal in control of rotating the credentials of the other Service Principals. Since that is the context that the script is running, executing the command to reset the credentials immediately invalidates the current session and no further commands can be executed.

# Info Dump

Getting to the solution to this took a long time largely due to the restrictions causing us to have to wait for approval for our permission request but also knowing exactly what permissions would be sufficient for our needs. Part of that was discovering the ability to set a Service Principal as another Service Principal’s owner. Once we had that in place, the rest was just implementation, until another snag occurred with the Azure DevOps pipeline permission, which required more research to discover the “Endpoint Administrators” group that we needed because the solution used a Service Connection with the AzureCli task to run the PowerShell script. I plan on updating this with a V2 that can allow the managing Service Principal to rotate its own credentials.

### Links:
- [Blog post that has a large overlap with our goals but didn’t exactly cover our needs](https://koosg.medium.com/implement-automatic-key-rotation-on-azure-devops-service-connections-13804b92157c)
- [StackOverflow post that brought up the possibility to use the CLI to add a Service Principal as owner to another Service Principal](https://stackoverflow.com/questions/49498953/how-to-add-an-owner-to-registered-application-in-azure-ad)
- [Microsoft Documentation on the Powershell cmdlet to add an owner to a Service Principal](https://learn.microsoft.com/en-us/powershell/module/azuread/add-azureadserviceprincipalowner?view=azureadps-2.0)
- [Microsoft Documentation on Azure DevOps Service Endpoint API](https://learn.microsoft.com/en-us/rest/api/azure/devops/serviceendpoint/endpoints/get-service-endpoints-by-names?view=azure-devops-rest-7.1&amp;tabs=HTTP)
- [StackOverflow post that brought up the need for granting the Build Service Accounts in Azure DevOps the endpoint access](https://stackoverflow.com/questions/53207636/how-to-grant-a-tfs-build-agent-access-rights-to-tfs-rest-api)
