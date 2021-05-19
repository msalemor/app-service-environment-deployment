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
- Code deployed to a repo
- Ability to create and execute pipelines
- A self-hosted agent in a VNET, with the deployment tools (Git, Dotnet CLI, etc.), having access to the ASE
- The pipeline needs to use this self-hosted agent to process the pipeline
- An ADO service connection to the resource group where the ASE is hosted

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
              deployToSlotOrASE: true
              ResourceGroupName: $(rgName)
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
  # Azure Resource Manager connection created during pipeline creation
  azureSubscription: '{{ azureRmConnection.Id }}'
  # Agent VM image name
  vmImageName: 'vmagent1'
  # Working Directory
  workingDirectory: 'DemoMVApp'
  serviceConnection: 'asev3svcconnection'
  rgName: 'rg-asev3-eus-demo'
  appName: 'alemortest'
  prodAppName: 'alemorprod'

stages:
- stage: Build
  displayName: Build stage
  pool:
      name: 'Windows Hosts'
  
  jobs:
  - job: Build
    displayName: Build
        
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
        publishWebProjects: true
        arguments: '--output $(System.DefaultWorkingDirectory)/publish_output --configuration Release'
        zipAfterPublish: false
        modifyOutputPath: false

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
  pool:
      name: 'Windows Hosts'

  jobs:
  - deployment: Deploy
    displayName: Deploy to dev
    environment: 'dev'

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
              deployToSlotOrASE: true
              ResourceGroupName: $(rgName)
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
  pool:
      name: 'Windows Hosts'

  jobs:
  - deployment: Deploy
    displayName: Deploy to test
    environment: 'test'

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
  pool:
      name: 'Windows Hosts'

  jobs:
  - deployment: Deploy_Prod
    displayName: 'Deploy to prod'
    environment: 'prod'

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
