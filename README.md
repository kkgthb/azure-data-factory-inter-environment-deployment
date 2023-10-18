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

To do that, you'll create a Microsoft Entra ID _(formerly "Azure Active Directory" / "AAD")_ "**[Service Principal](https://learn.microsoft.com/en-us/azure/active-directory/develop/app-objects-and-service-principals?tabs=browser)**" -- specifically one of the "**[App Registration](https://learn.microsoft.com/en-us/azure/active-directory/develop/quickstart-register-app)**" variety.

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

* The fewer app registrations you create, the more _**powerful**_ you'll have to make each one's "authorization" -- which could be a security vulnerability and break the "principle of least privilege."
* On the other hand, the more app registrations you create, the more your team will have to _**remember**_ you created -- and human unmanageability to the point of poor maintenance also makes application architecture components security vulnerabilities.

The patterns by which your cloud infrastructure team has chosen to cluster various Azure resources into Azure's "resource group" and "subscription" hierarchy can also act as a limiter on how much widely any given "authorization" scoping choice will have to do damage.  _(My tutorials will always describe granting privileges scoped to 1 individual Azure resource ID at a time, which is the most conservative, least-privilege approach and means you don't have to think about how resources are clumped.  However, in enterprise settings, it can be common to grant privileges to entire groups of resources.  Therefore, the way you safely plan for "authorization" decisions **while inventing "authentication"-related App Registration names** may be impacted by the "grouping" of resources at your company and whether you plan to grant "authorization" scoped to individual resource IDs or to entire groups of resources.)_

There's no 100% correct way to scope and name App Registrations, but your enterprise and/or team probably has standardized ways.  Ask around.

##### GitHub.com federated identity and OIDC

If your codebase is stored at GitHub.com, you'll probably also want to attach a "federated identity" to each app registration to help GitHub.com log into the App Registration over OIDC.

#### Authorization

Once you've created an App Registration to represent the CI/CD codebase you'd like to deploy your ADF definition into production with, you need to specify that it's actually allowed to do so, using an "Azure RBAC Role Assignment."

Let's say you named your App Registration "`hr-payroll-helloworld-adf-prd-cicd`."

In that case, you'd need to:

1. _assign_ the Microsoft-prebuilt _("built-in")_ RBAC role named "**Data Factory Contributor**"...
2. ...to `hr-payroll-helloworld-adf-prd-cicd`...
3. ...scoped to the Azure **Resource ID** of your production ADF resource for the "hello world" project.

_(Or open a ticket with your colleagues to have them assign it on your behalf.  Again, consider asking around whether AAD PIM might let you do this yourself rather than having to open a ticket.)_

### Runtime permissions

Is your production ADF resource ready to talk to _other_ production Azure resources, such as your production Key Vault resource, once it's "deployed" and up and running?

Again, since tickets can take time to fulfill, think about this right away, before you even start writing any deployment-oriented code.

Ask around whether your production ADF resource has been given a [System-Assigned Managed Identity](https://learn.microsoft.com/en-us/azure/active-directory/managed-identities-azure-resources/overview), and whether:

1. an appropriate RBAC role _(e.g. "Key Vault Reader")_ has been assigned...
2. ...to your production ADF resource's Managed Identity...
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