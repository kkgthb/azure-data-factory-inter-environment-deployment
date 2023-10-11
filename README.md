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