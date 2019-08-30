# Review app

## Workflow goals
From a workflow the requirements are:

1. Trigger on feature branch update and/or PR start.
2. Reuse the same build steps as regular workflow.
3. Reuse the same deploy steps, with the difference that the target is a specific, possibly temporary, location (k8s = a specific namespace with Dev Spaces enabled; create ns if not exists can be part of the logic).
4. The new resources is added to the Environment
5. The tracebility works for the new resource in the environment
6. The new resource is tagged with all the required data. Url, identifier for dynamic environment resource, PR, commit identifiers etc.
7. Clean-up / retention of dynamic environment based on environemnt.resources tagged as dynamic

One other intent is to lower the concept count. For example avoid adding new complexities to yaml. We explored numerious options from a new typed jon/stage to lifecycle hooks.


## Backend capabilities which are a must have
1. Dynamically register a new K8s namespace to an existing Environment
	 Applies to a web App (slot) as well
 2. Ability to write tags/metadat to a Env.Resource

## Proposed yaml
```yaml
# Deploy the changes on PR and commit to Azure Kubernetes Service
# Build and push image to Azure Container Registry; Deploy to Azure Kubernetes Service
# https://docs.microsoft.com/azure/devops/pipelines/languages/docker

# Trigger only for master branch, Bikes folder
# Mono-repo scenario
trigger:
  branches:
    include:
    - master
  paths:
    include:
    - samples/BikeSharingApp/Bikes/*

resources:
- repo: self

variables:

  # Environemnt resource. Review apps will be deployed in a Dev environment. 
  # Environment will have a K8s namespace resource where latest of all the services will be deployed from master branch
  # The underlying service connection will have permission to cretae a new Namespace in the K8s cluster
  devEnvironment: 'DevEnvironment.MasterNS'
  
  # Kubernetes Namespace where all changes from master branch are deployed
  k8sNamespace: 'dev'
  
  # Dynamically created Kubernetes Namespace where all pull request changes are deployed
  k8sNamespaceforpr: $(system.pullRequest.sourceBranch)

# CI stage for build and pushing image to container registr y
stages:
- stage: Build
  displayName: Build stage
  jobs:  
  - job: Build
    pool:
      vmImage: 'ubuntu-latest'
    steps:
    - task: Docker@2
      displayName: Build and push an image to container registry
      inputs:
        command: buildAndPush
        
# Stage contains a common deploymnent job
# Deployment job is preeceded by a job responsible for conditionally creating a review app Environment.Resource

- stage: DeployToDevEnv
  displayName: Deploy to Dev Environment
  dependsOn: Build
  
  jobs:
  - deployment: Provision and configure Dynamic environment for a PR
    displayName: Dynamic environment for a PR
    pool:
      vmImage: 'ubuntu-latest'
    condition: and(succeeded(), startsWith(variables['Build.SourceBranch'], 'refs/pull/'))     
    environment: $(devEnvironment)
      reviewApp: true
      # The exact name is TBD
      # For K8s and Azure Web App, this flag when set to true will create a clone of Env.Resource
      # For K8s it will
      #   1. Create a new namespace (combinbation of existing namespace and feature branch name 
      #   2. Register the new namespace to the Environment as a resource
      #   3. Set the Env.Resource from the existing namespace to the new namespace created for PR
      #   3. Tag the Env.Resource it with data like: Url, Dynamic Environment identifier, PR-commit details etc
      #   4. The tag metadata will be used for writing back to the git repository. For example app url
      #   5. The tag metadata will be used for cleanup of dynamic environment resource as well
      # For Web app the same will be done for a slot
      # Optionally user can skip the flag or augument its functionality by running additional steps in the job
      # For example: Any extra configuration required by the review app. In case of Azure DevSpaces - add root label
    strategy:
      runOnce:
        deploy:
          steps:
          - task: Kubernetes@1
            displayName: 'Add devspace label'
            inputs:
              command: label
              arguments: '--overwrite ns $(system.pullRequest.sourceBranch) azds.io/space=true'
              checkLatest: true

          - task: Kubernetes@1
            displayName: 'Setup root devspace to dev'
            inputs: 
              command: label
              arguments: '--overwrite ns $(system.pullRequest.sourceBranch) azds.io/parent-space=$(k8sNamespace)'
              checkLatest: true
              
    # The deploy job is common. 
    # It will trigger always and deploy to the environment which enables full tracebility
    # If it is a PR scenario the environemnt will point to the new dynamically created namespace
    # If it is a Commit scenario, it will deploy to the root namespace
  - deployment: Common Deploy job
    displayName: Deploy job
    pool:
      vmImage: 'ubuntu-latest'

    environment: $(devEnvironment)
    strategy:
      runOnce:
        deploy:
          steps:
          - task: KubernetesManifest@0
            displayName: Create imagePullSecret
            inputs:
              action: createSecret
       
          - task: KubernetesManifest@0
            displayName: bake
            name: bake
            inputs:
              action: bake
               
          - task: KubernetesManifest@0
            displayName: deploy
            name: deploy
            inputs:
              action: deploy   
```

## Scratchboard [Ignore]
```yaml
    - powershell: |
              Write-Host "Register the new resource to the environment"
              Write-Host "##vso[task.setvariable variable=RETURI]$devEnvironment"
              displayName: 'Register to '
            
          - powershell: |
              $uriret = (Get-Content $(AGENT.TEMPDIRECTORY)/azdsuri.txt | Out-String | ConvertFrom-Json)[0].uri
              Write-Host "##vso[task.setvariable variable=RETURI]$uriret"
              Write-Host "$uriret"
            displayName: 'Get PR review app url for the current PR and write to Environemt.DynamicResource tags'
```
