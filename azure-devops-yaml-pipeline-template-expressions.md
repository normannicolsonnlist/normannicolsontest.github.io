# Azure Dev Ops Yml Template Expressions

Oct 2023

> Yaml Azure DevOps pipleline template expressions

Separate reusable build and release steps into separate files to make maintainable pipelines.

```
trigger:
- develop

name: 0.1.$(Year:yyyy)$(Month)$(DayOfMonth)$(Rev:.r)

pool:
  vmImage: 'windows-latest'

variables:
  buildPlatform: 'Any CPU'
  buildConfiguration: 'Release'
  publishOutput: 'publish-output'

stages:
- stage: build
  jobs:
  - job: build
    steps:
    - template: /pipelines/build.yml
      parameters:
        vstsFeed: '$(vstsFeed)'
        buildConfiguration: '$(buildConfiguration)'
        versionSuffix: '$(Build.BuildNumber)'
        publishOutput: '$(publishOutput)'

- ${{ if ne(variables['Build.Reason'], 'PullRequest') }}: 
  - stage: dev
    dependsOn: build
    condition: succeeded()
    jobs:

      - template: /pipelines/deploy-infrastructure.yml
        parameters:
          environment: dev
          package: '$(Pipeline.Workspace)\drop\Nlist.Infrastructure.zip'

      - template: /pipelines/deploy-database.yml
        parameters:
          environment: dev
          package: '$(Pipeline.Workspace)\drop\Nlist.Data.Bundle.zip'
          db_connection_string: '$(api_api_connectionstring_dev)'

      - template: /pipelines/deploy-api.yml
        parameters:
          environment: dev
          package: '$(Pipeline.Workspace)\drop\Nlist.Api.zip'

          api_b2c_tenantname: '$(api_b2c_tenant_dev)'
          api_b2c_tenantid: '$(api_b2c_tenantid_dev)'
          api_b2c_api_clientid: '$(api_b2c_itemsapi_clientid_dev)'

      - template: /pipelines/deploy-ui.yml
        parameters:
          environment: dev
          package: '$(Pipeline.Workspace)\drop\Nlist.Ui.zip'
```

template: /pipelines/build.yml
```
parameters:
  vstsFeed: ''
  buildConfiguration: ''
  versionSuffix: ''
  publishOutput: ''

steps:

  - task: DotNetCoreCLI@2
    displayName: 'dotnet restore'
    inputs:
      command: 'restore'
      projects: '**/*.csproj'
      feedsToUse: 'select'
      vstsFeed: '$(vstsFeed)'

  - task: DotNetCoreCLI@2
    displayName: 'dotnet build'
    inputs:
      command: 'build'
      projects: '**/*.csproj'
      arguments: '--configuration $(buildConfiguration) --no-restore /p:VersionSuffix=$(Build.BuildNumber)'

  - task: DotNetCoreCLI@2
    displayName: 'dotnet test'
    inputs:
      command: 'test'
      projects: '**/*.Test.csproj'
  
  - task: PowerShell@2
    displayName: "dotnet publish ef migrations Nlist.Items.Infrastructure.Data"
    condition: and(succeeded(), ne(variables['Build.Reason'], 'PullRequest'))
    inputs:
      filePath: 'pipelines/build-migrations.ps1'
      arguments: '-output $(publishOutput)'

  - task: DotNetCoreCLI@2
    displayName: "dotnet publish Nlist.Items.Api"
    condition: and(succeeded(), ne(variables['Build.Reason'], 'PullRequest'))
    inputs:
      command: publish
      arguments: '--configuration $(buildConfiguration) --no-build --no-restore --output $(publishOutput)'
      projects: 'src/Nlist.Items.Api/Nlist.Items.Api.csproj'
      publishWebProjects: false

  - task: DotNetCoreCLI@2
    displayName: "dotnet publish Nlist.Items.UI.Client"
    condition: and(succeeded(), ne(variables['Build.Reason'], 'PullRequest'))
    inputs:
      command: publish
      arguments: '--configuration $(buildConfiguration) --no-build --no-restore --output $(publishOutput)'
      projects: 'src/Nlist.Items.UI.Client/Nlist.Items.UI.Client.csproj'
      publishWebProjects: false

  - task: PublishBuildArtifacts@1
    displayName: "publish artifacts"
    condition: and(succeeded(), ne(variables['Build.Reason'], 'PullRequest'))
    inputs:
      PathtoPublish: '$(publishOutput)'
      publishLocation: 'Container'
```

template: /pipelines/deploy-api.yml

```
parameters:
  environment: 'dev'
  environment_domain: 'dev.'
  sslThumbprint: ''
  package: ''
  website_name: 'ItemsApi'
  ui_domain: 'items.example.com'
  api_domain: 'itemsapi.example.com'

  api_b2c_tenantname: ''
  api_b2c_tenantid: ''
  api_b2c_itemsapi_clientid: ''
  api_applicationinsights_connectionstring: ''
  api_items_connectionstring: ''
  api_appconfiguration_connectionstring: ''

jobs:
- deployment: ${{ parameters.environment }}_api
  environment: 
    name: ${{ parameters.environment }}
    resourceType: virtualMachine
    tags: ${{ parameters.environment }} 
  variables:
    # Api
    ApplicationInsights.ConnectionString: '${{ parameters.Api_applicationinsights_connectionstring }}'
    AzureAdB2cOptions.ClientId: '${{ parameters.Api_b2c_itemsapi_clientid }}'
    AzureAdB2cOptions.Domain: '${{ parameters.Api_b2c_tenantname }}.onmicrosoft.com'
    AzureAdB2cOptions.Instance: 'https://${{ parameters.Api_b2c_tenantname }}.b2clogin.com'
    AzureAdB2cOptions.SignUpSignInPolicyId: 'B2C_1A_SIGNIN'
    AzureAdB2cOptions.TenantId: '${{ parameters.Api_b2c_tenantid }}'
    CorsOptions.AllowedOrigins: 'https://${{ parameters.environment_domain }}${{ parameters.ui_domain }}' 
  strategy:
    runOnce:
      deploy:   
        steps:

        # Output parameter values 
        - task: PowerShell@2
          displayName: 'parameters'
          inputs:
            targetType: 'inline'
            script: |
              Write-Host '${{ convertToJson(parameters) }}'

        # Output variable values 
        - task: PowerShell@2
          displayName: 'variables'
          inputs:
            targetType: 'inline'
            script: |
              Write-Host '${{ convertToJson(variables) }}'

        # IIS management
        - task: IISWebAppManagementOnMachineGroup@0
          displayName: 'items iss management'
          inputs:            
            IISDeploymentType: 'IISWebsite'
            ActionIISWebsite: 'CreateOrUpdateWebsite'
            WebSiteName: '${{ parameters.website_name }}'
            WebsitePhysicalPath: '%SystemDrive%\inetpub\wwwroot\${{ parameters.website_name }}'
            WebsitePhysicalPathAuth: 'WebsiteUserPassThrough'
            CreateOrUpdateAppPoolForWebsite: true
            AppPoolNameForWebsite: '${{ parameters.website_name }}'
            DotNetVersionForWebsite: 'No Managed Code'
            PipeLineModeForWebsite: 'Integrated'
            AppPoolIdentityForWebsite: 'ApplicationPoolIdentity'
            AddBinding: true
            Bindings: |
              {
                bindings:[
                {
                  "protocol":"http",
                  "ipAddress":"*",
                  "hostname":"${{ parameters.environment_domain }}${{ parameters.Api_domain }}",
                  "port":"80",
                  "sslThumbprint":"",
                  "sniFlag":false
                },
                {
                  "protocol":"https",
                  "ipAddress":"*",
                  "hostname":"${{ parameters.environment_domain }}${{ parameters.Api_domain }}",
                  "port":"443",
                  "sslThumbprint":"${{ parameters.sslThumbprint }}",
                  "sniFlag":true
                }]
              }

        - task: IISWebAppManagementOnMachineGroup@0
          displayName: 'items iss stop'
          inputs:
            IISDeploymentType: 'IISWebsite'
            ActionIISWebsite: 'StopWebsite'
            WebSiteName: '${{ parameters.website_name }}'
            StartStopWebsiteName: '${{ parameters.website_name }}'

        # IIS deploy
        - task: IISWebAppDeploymentOnMachineGroup@0
          displayName: 'items iss deploy'
          inputs:
            WebSiteName: '${{ parameters.website_name }}'
            Package: '${{ parameters.package }}'
            RemoveAdditionalFilesFlag: true
            JSONFiles: 'appsettings.json'

        - task: IISWebAppManagementOnMachineGroup@0
          displayName: 'items iss start'
          inputs:
            IISDeploymentType: 'IISWebsite'
            ActionIISWebsite: 'StartWebsite'
            WebSiteName: '${{ parameters.website_name }}'
            StartStopWebsiteName: '${{ parameters.website_name }}'

```