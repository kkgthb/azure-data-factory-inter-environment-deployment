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

I think it's silly.

But that's the way it is, for whatever reason, as far as I can tell.

Of course, you could also just spin up a new Azure DevOps repository using my repository's URL as a basis, and then reference it in your pipeline with a syntax more like this:

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