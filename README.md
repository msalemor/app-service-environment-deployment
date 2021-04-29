# Deploy a .Net Core App to App Service Environments using ADO


## What is an App Service Environment

- Provides isolated hardware and networking
- Can be deployed inside a VNET
- Provides advanced networking control capabilities
- High scalability
- Ideal choice in highly regulated environment such as finance, PCI, HIPAA and government
- Requires DNS entries and inbound and outbound ports to operate

### Requirements for deployting to an internal ASE from ADO

- Azure DevOps account
- Code deployed to a project
- Ability to create and execute pipelines
- A self-hosted agent, with tools (Git, Dotnet CLI, etc.) to the VNET where the ASE is deployed
- The pipeline needs to use this self-hosted agent to process the pipeline

```yaml
pool:
      name: 'Windows Hosts'
```

## About the yaml pipeline

- Builds and publishs the .Net Core app
- Saves the artifacts
- Deploys the code from the artifact location to a dev/test ASE
- Upon approval, swaos dev with test slots
- Upon approval, deploys the code to the production ASE from the artificate location

### Deploy to Dev

```yaml
- task: AzureRmWebAppDeployment@4
            displayName: 'Deploying Web App to Dev environment'
            inputs:
              ConnectedServiceName: '$(serviceConnection)'
              appType: webApp
              WebAppName: $(appName)
              SlotName: 'dev'
              package: '$(Pipeline.Workspace)/drop/$(Build.BuildId).zip'
```

### Swap Dev with Test

```yaml
- task: AzureAppServiceManage@0
            displayName: 'Swap'
            inputs:
              ConnectedServiceName: '$(serviceConnection)'
              WebAppName: $(appName)
              ResourceGroupName: '$(rgName)'
              Action: 'Swap Slots'
              SourceSlot: 'dev'
```

### Deploy to Prod

```yaml
- task: AzureRmWebAppDeployment@4
            displayName: 'Deploying Web App to Dev environment'
            inputs:
              ConnectedServiceName: '$(serviceConnection)'
              appType: webApp
              WebAppName: $(prodAppName)
              SlotName: 'dev'
              package: '$(Pipeline.Workspace)/drop/$(Build.BuildId).zip'
```

### Complete pipeline

```yaml
# ASP.NET Core (.NET Framework)
# Build and test ASP.NET Core projects targeting the full .NET Framework.
# Add steps that publish symbols, save build artifacts, and more:
# https://docs.microsoft.com/azure/devops/pipelines/languages/dotnet-core

trigger:
- master

variables:
  workingDirectory: 'DemoMVApp'
  serviceConnection: 'asev3svcconnection'
  appName: 'aseapp2'
  prodAppName: 'aseapp3'
  rgName: 'rg-asev3-eus-demo'

stages:
- stage: Build
  displayName: Build stage

  jobs:
  - job: Build
    displayName: Build
    pool:
      name: 'Windows Hosts'

    steps:
    - task: DotNetCoreCLI@2
      displayName: Build
      inputs:
        command: 'build'
        projects: |
          $(workingDirectory)/*.csproj

    - task: DotNetCoreCLI@2
      displayName: Publish
      inputs:
        command: 'publish'
        projects: |
          $(workingDirectory)/*.csproj
        arguments: --output $(System.DefaultWorkingDirectory)/publish_output --configuration Release

    - task: ArchiveFiles@2
      displayName: 'Archive files'
      inputs:
        rootFolderOrFile: '$(System.DefaultWorkingDirectory)/publish_output'
        includeRootFolder: false
        archiveType: zip
        archiveFile: $(Build.ArtifactStagingDirectory)/$(Build.BuildId).zip
        replaceExistingArchive: true

    - publish: $(Build.ArtifactStagingDirectory)/$(Build.BuildId).zip
      artifact: drop

- stage: Deploy
  displayName: Deploy to dev
  dependsOn: Build
  condition: succeeded()      

  jobs:
  - deployment: Deploy
    displayName: Deploy to dev
    environment: 'dev'
    pool:
      name: 'Windows Hosts'

    strategy:
      runOnce:
        deploy:

          steps:
          - task: AzureAppServiceManage@0
            displayName: 'Stop Web App'
            inputs:
              ConnectedServiceName: '$(serviceConnection)'
              WebAppName: $(appName)
              Action: 'Stop Azure App Service'
          
          - task: AzureRmWebAppDeployment@4
            displayName: 'Deploying Web App to Dev environment'
            inputs:
              ConnectedServiceName: '$(serviceConnection)'
              appType: webApp
              WebAppName: $(appName)
              SlotName: 'dev'
              package: '$(Pipeline.Workspace)/drop/$(Build.BuildId).zip'

          - task: AzureAppServiceManage@0
            displayName: 'Start'
            inputs:
              ConnectedServiceName: '$(serviceConnection)'
              WebAppName: $(appName)
              Action: 'Start Azure App Service'

- stage: Swap
  displayName: Swap Dev to Test
  dependsOn: Build
  condition: succeeded()      

  jobs:
  - deployment: Deploy
    displayName: Deploy to test
    environment: 'test'
    pool:
      name: 'Windows Hosts'

    strategy:
      runOnce:
        deploy:

          steps:
          - task: AzureAppServiceManage@0
            displayName: 'Swap'
            inputs:
              ConnectedServiceName: '$(serviceConnection)'
              WebAppName: $(appName)
              ResourceGroupName: '$(rgName)'
              Action: 'Swap Slots'
              SourceSlot: 'dev'

- stage: Deploy_Prod
  displayName: 'Deploy to prod'
  dependsOn: Build
  condition: succeeded()      

  jobs:
  - deployment: Deploy_Prod
    displayName: 'Deploy to prod'
    environment: 'prod'
    pool:
      name: 'Windows Hosts'

    strategy:
      runOnce:
        deploy:

          steps:
          - task: AzureAppServiceManage@0
            displayName: 'Stop Web App'
            inputs:
              ConnectedServiceName: '$(serviceConnection)'
              WebAppName: $(prodAppName)
              Action: 'Stop Azure App Service'
          
          - task: AzureRmWebAppDeployment@4
            displayName: 'Deploying Web App to Dev environment'
            inputs:
              ConnectedServiceName: '$(serviceConnection)'
              appType: webApp
              WebAppName: $(prodAppName)
              SlotName: 'dev'
              package: '$(Pipeline.Workspace)/drop/$(Build.BuildId).zip'

          - task: AzureAppServiceManage@0
            displayName: 'Start'
            inputs:
              ConnectedServiceName: '$(serviceConnection)'
              WebAppName: $(prodAppName)
              Action: 'Start Azure App Service'
```              

## References

- https://docs.microsoft.com/en-us/azure/devops/pipelines/process/stages?view=azure-devops&tabs=yaml
