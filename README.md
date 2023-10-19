# CI/CD helper for migrating one Azure Data Factory resource's definition into another ADF resource

## Introduction

Once you've:

1. [Connected a "nonproduction" Azure Data Factory _("ADF")_ resource to Azure DevOps Repos or to GitHub.com via the Git protocol](https://katiekodes.com/git-azure-data-factory/), 
1. "Published" your work into that nonproduction ADF, 
1. Tested it, and 
1. Seen that you like what it does...

You might decide that you'd like to migrate all of your hard work into a "production" ADF resource.

This tutorial and codebase can help you do so by adding "CI/CD pipeline" code into the same Git repository that stores a copy of your "nonproduction" ADF resource's definition.

--- 

## Permissions

If you're working in an enterprise setting, Azure permissions requests might have to go through a ticketing process that takes a few days to fulfill, so we'll start by focusing on permissions.

### Deployment via CI/CD pipeline

You'll need to ensure that the CI/CD pipeline codebase you're about to write is permitted to **alter** the state of your "production" ADF resource.  Otherwise, it wouldn't be very useful for deploying your nonproduction ADF definition into production.

Permissions always consist of an **authentication** aspect _("prove that you are who you say you are")_ and an **authorization** aspect _("okay, I believe that you are who you say you are, but let me check if you're actually allowed to do the thing you're asking me to let you do?")_.

#### Authentication

You'll need to give your CI/CD pipeline codebase a special identity that it can use when asked to **authenticate** itself while trying to deploy an ADF definition into production _("Hi, I'm a CI/CD codebase dedicated to the 'Hello World' ADF project!")_.

To do that, you'll create a Microsoft Entra ID _("Entra," formerly "Azure Active Directory" / "AAD")_ "**[Service Principal](https://learn.microsoft.com/en-us/azure/active-directory/develop/app-objects-and-service-principals?tabs=browser)**" -- specifically one of the "**[App Registration](https://learn.microsoft.com/en-us/azure/active-directory/develop/quickstart-register-app)**" variety.

_(Note:  In an enterprise setting, you might have to open a ticket to have one created.  Alternatively, you might have -- or be able to ask for -- a Privileged Identity Management ("PIM") role that lets you create an App Registration yourself, and into which you can temporarily escalate your own privileges into as needed.)_

##### Naming app registrations

When I [create an App Registration](https://katiekodes.com/provision-azure-ad-service-principal-website-contributor/#provision-an-azure-ad-application-registration), I like to be very descriptive with its name.

For example, if:

1. I worked for Payroll in my company's HR department, 
1. I were building out our "hello world" application, 
1. one of its components was an ADF resource _(and another one was an Azure App Service webapp resource)_, and
1. I had 2 nonproduction environments, the "higher" of which was called "staging" or "`stg`," as well as 1 production environment nicknamed "`prd`"...

Then I might choose one of the following 4 naming patterns for all of the App Registrations I create:

1. One App Registration named:
    * `hr-payroll-helloworld-cicd`
2. Two App Registrations named:
    * `hr-payroll-helloworld-stg-cicd`
    * `hr-payroll-helloworld-prd-cicd`
3. Two App Registrations named:
    * `hr-payroll-helloworld-webapp-cicd`
    * `hr-payroll-helloworld-adf-cicd`
4. Four App Registrations named:
    * `hr-payroll-helloworld-webapp-stg-cicd`
    * `hr-payroll-helloworld-webapp-prd-cicd`
    * `hr-payroll-helloworld-adf-stg-cicd`
    * `hr-payroll-helloworld-adf-prd-cicd`

In an enterprise setting, ask your team, as well as teams of colleagues specializing in identity and access management, infosec, application development, cloud infrastructure, and software architecture if there are any standards you should be following.

While we're talking about "authentication" right now, ultimately, you'll be granting "authorization" privileges to these "identities," so it might help maintain sanity if you **name things according to what they're expected to be able to do**.

* The fewer app registrations you create, the more _**powerful**_ you'll have to make each one's "authorization" -- which could be a security vulnerability and break the "principle of least privilege."
* On the other hand, the more app registrations you create, the more your team will have to _**remember**_ you created -- and human unmanageability to the point of poor maintenance also makes application architecture components security vulnerabilities.

The patterns by which your cloud infrastructure team has chosen to cluster various Azure resources into Azure's "resource group" and "subscription" hierarchy can also influence the damage potential of "authorization" grants.  _(My tutorials will always describe granting privileges scoped to 1 individual Azure resource ID at a time, which is the most conservative, least-privilege approach.  My approach means you don't have to think about how resources are clumped at all.  However, in enterprise settings, it can be common to scope the granting of privileges to entire groups of resources.  Therefore, the way you safely plan for "authorization" decisions **while inventing "authentication"-related App Registration names** may be impacted by the "grouping" of resources at your company and whether you plan to grant "authorization" scoped to individual resource IDs or to entire groups of resources.)_

There's no 100% correct way to scope and name App Registrations, but your enterprise and/or team probably has standardized ways.  Ask around.

##### Federated credentials and your CI/CD codebase

You'll also need to [attach a "federated credential" to the Service Principal](https://learn.microsoft.com/en-us/azure/active-directory/workload-identities/workload-identity-federation-create-trust) associated with the App Registration you just created.

You might be able to do this yourself _(particularly with "PIM" privilege escalation)_, or you might need to open a ticket to have one of your colleagues do it for you.  If you open a ticket, be sure to provide the details that my notes say you'd need if you were doing it yourself.

###### Azure DevOps

If your codebase is stored in an Azure DevOps Repo, you'll need to add a federated credential in Entra using the "**Other issuer**" scenario picklist option.  You'll get the "**Issuer**" and "**Subject identifier**" details you need out of the Azure DevOps portal.  Be sure to give the federated credential a good **name** _(note that you can't rename the federated credential once you save it, though you can delete it and create an identical one with a new name)_ and, preferably, a good 600-character-or-less **description**.  Honestly ... so far, I've just named mine "`log-ado-service-connection-into-this-service-principal`."  The name only has to be unique _within_ the context of the service principal, so it doesn't have to be terribly creative.

Over in the Azure DevOps portal, [create a new **Service Connection**](https://learn.microsoft.com/en-us/azure/devops/pipelines/release/configure-workload-identity) of the "**Azure Resource Manager**" type and pick the "**Workload Identity federation (manual)**" option.

Pick a meaningful name for it like "**`Hello World`**," "**`HR Payroll Hello World ADF Production`**," etc. depending on what you named your Entra App Registration and depending on whether things like "HR" and "Payroll" are already implied by the names of your Azure DevOps "organization" and "project."  Click the "**Next**" button.

Copy the "**Issuer**" and "**Subject identifier**" values onto your clipboard.  If you're setting up your Entra Service Principal's Federated Credential yourself, go ahead and paste these values into the corresponding fields in Entra's "**Add a credential**" wizard.  _(And, after adding a name and possibly a description, click the "**Add**" button.)_  If you're opening a support ticket, be sure to include these 2 details along with the name you'd like to give your federated credential.

Getting your Entra Service Principal's Federated Credential created will make sure that when you finish saving your Azure DevOps Service Connection that points to your Entra Service Principal, you don't get the following error:

> Azure Active Directory rejected the token issued by Azure DevOps with error code AADSTS70021: No matching federated identity record found for presented assertion.

By the way, every Azure DevOps project has a unique ID -- you just don't usually see it in your URL bar while browsing around Azure DevOps.  The unique ID of your Azure DevOps project ends up in the "issuer" value, and the name of the service connection you create ends up in your "subject identifier" value, so you end up with something like:

- `https://vstoken.dev.azure.com/a1234b5c-d6ef-1234-987-abcdefabcdef`
- `sc://your-ado-org-name/your-ado-project-name/HR Payroll Hello World ADF Production`

Click the "**Keep as draft**" button at the bottom of the "New Azure service connection" right-hand flyout pane in Azure DevOps to close it out and come back to it later.  _(There are some "authentication" things we have to take care of before we can finish creating it, but we had to start creating it to get the "issuer" and "subject identifier" details we needed, since it's not otherwise easy to figure out your ADO project's unique ID.)_

##### GitHub.com federated identity and OIDC

If your codebase is stored at GitHub.com, you'll need to add a federated credential using the "**GitHub Actions deploying Azure resources**" scenario picklist option -- you'll get the "**Issuer**" and "**Subject identifier**" details you need out of the Azure DevOps portal.  Be sure to know the organization name or username of the GitHub account hosting your codebase, the repository name of your codebase, and ...TODO... .

...TODO...

#### Authorization _(to deploy data factories)_

Once you've created an App Registration to represent the CI/CD codebase you'd like to deploy your ADF definition into production with, you need to specify that it's actually allowed to do so, using an "Azure RBAC Role Assignment."

Let's say you named your App Registration "`hr-payroll-helloworld-adf-prd-cicd`."

In that case, you'd need to:

1. _assign_ the Microsoft-prebuilt _("built-in")_ RBAC role named "**Data Factory Contributor**"...
2. ...to `hr-payroll-helloworld-adf-prd-cicd`...
3. ...scoped to the Azure **Resource ID** of your production ADF resource for the "hello world" project.

_(Or open a ticket with your colleagues to have them assign it on your behalf.  Again, consider asking around whether Entra PIM might let you do this yourself rather than having to open a ticket.)_

###### Finish creating your ADO Service Connection

Once the "Data Factory Contributor" role has been assigned and you've waited a minute or few, you should be able to finish out creating your "`HR Payroll Hello World ADF Production`" Azure DevOps Service Connection that points at `hr-payroll-helloworld-adf-prd-cicd` _("**Verify and save**" button, bottom right, to finish it out)_, using the subscription ID, subscription name, and tenant ID of the Azure subscription (and its parent tenant) that contain your production ADF resource.

Note:  I believe that if you also have RBAC roles assigned to your `hr-payroll-helloworld-adf-prd-cicd` Entra Service Principal that are scoped to, say, your "**staging**" ADF resource ID -- even if that resource lives in a different Azure subscription than you punched into the Azure DevOps Service Connection creation wizard -- your "`HR Payroll Hello World ADF Production`" ADO Service Connection will also be usable for CI/CD pipeline codebases that want to deploy to your "staging" ADF resource.  _(This is why I said to think ahead about authorization before you name things for authentication!)_  I need to test it to be sure, but if true, I kind of wonder why Microsoft even bothers making you punch an Azure subscription ID into the ADO service connection creation wizard at all.

### Runtime permissions

Is your production ADF resource ready to talk to _other_ production Azure resources, such as your production Key Vault resource, once it's "deployed" and up and running?

Again, since tickets can take time to fulfill, think about this right away, before you even start writing any deployment-oriented code.

#### Authentication

Ask around whether your production ADF resource has been given a [System-Assigned Managed Identity](https://learn.microsoft.com/en-us/azure/active-directory/managed-identities-azure-resources/overview).

#### Authorization _(to, say, read Key Vault values)_

Also ask around whether:

1. an appropriate RBAC role _(e.g. "Key Vault Reader")_ has been assigned...
2. ...to your production ADF resource's System-Assigned Managed Identity...
3. ...scoped to the Azure **Resource ID** of, for example, your production Key Vault resource for the "hello world" project.

If not, find out how to get that done and get the ball rolling.

---

## Parameterization and naming standards

Is your "non-production" ADF resource _really_ designed in a way that makes your life easiest when trying to migrate it to "higher" environments like production?

...TODO...





---

---

## Troubleshooting

Unfortunately, if you try to call this repository from your Azure DevOps pipeline like this:

```yaml
resources:
  repositories: 
    - repository: 'adfBuildAndDeployTemplatesRepo'
      type: 'github'
      name: 'kkgthb/azure-data-factory-inter-environment-deployment'
      ref: 'refs/heads/main'

steps:

- template: '/src/cicd-templates/ado-yaml-pipelines/build-deployable-artifact/level2-tasks/build_an_adf_environment.yaml@adfBuildAndDeployTemplatesRepo'
  parameters:
    ...
```

You'll get an error saying:

```
Repository adfBuildAndDeployTemplatesRepo requires an endpoint Parameter name: endpoint
```

Even though this is a public repository, if you want an Azure DevOps pipeline to run code stored at GitHub.com, you have to hook up your pipeline to a _specific_ GitHub.com account via an Azure DevOps "service connection."

And no, you can't get around this by creating a GitHub-typed service connection with "helloworld" as a personal access token instead of a real one.  The pipeline would then throw this error:

```
Could not get the latest source version for repository kkgthb/azure-data-factory-inter-environment-deployment hosted on https://github.com/ using ref refs/heads/main. GitHub reported the error, "Bad credentials"
```

I think it's silly.

But that's the way it is, for whatever reason, as far as I can tell.

That said, when you log into GitHub.com from the Azure DevOps Service Connection wizard, you don't have to actually let the mechanism with which you log into that GitHub.com account _do_ anything interesting in GitHub.com.  For example, I created a fine-grained Personal Access Token for one of my GitHub accounts and picked a random repository I didn't have much in and gave the PAT _0_ permissions against that repository _(for some reason, you can do 0 permissions, but you still have to pick a repository to have 0 permissions against)_.  Once I connected my ADO Service Connection creation wizard to that GitHub.com account using that PAT _(note:  in an enterprise setting, I'm pretty sure you should use OAuth, not PATs, and control access using "application" scope or something like that)_, I was able to run the above code just fine, even though the repository I attached to the PAT wasn't the one I was referencing in the code above.

Also, you could also just spin up a new Azure DevOps repository using my repository's URL as a basis, and then reference it in your pipeline with a syntax more like this:

```yaml
resources:
  repositories: 
    - repository: 'adfBuildAndDeployTemplatesRepo'
      type: 'git'
      name: 'your-ado-project-name/the-name-of-the-ado-repository-you-based-off-of-this-one'
      ref: 'refs/heads/main'

steps:

- template: '/src/cicd-templates/ado-yaml-pipelines/build-deployable-artifact/level2-tasks/build_an_adf_environment.yaml@adfBuildAndDeployTemplatesRepo'
  parameters:
    ...
```