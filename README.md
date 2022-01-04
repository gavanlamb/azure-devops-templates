# Azure DevOps Templates
This repository contains templates for Azure DevOps.

To use these templates, add a resource section to the top of the pipeline file.  
[Read more about resources](https://docs.microsoft.com/en-us/azure/devops/pipelines/process/resources?view=azure-devops&tabs=schema)  
Example:  
```yaml
resources:
  repositories:
    - repository: templates
      type: github
      name: gavanlamb/azure-devops-templates
      endpoint: gavanlamb
```

## AWS
### CLI
#### CodeDeploy deploy
To create a deployment and apply it

The [codedeploy.deploy.yml](./aws/cli/codedeploy.deploy.yml) is a [step template](https://docs.microsoft.com/en-us/azure/devops/pipelines/process/templates?view=azure-devops#step-reuse) meaning it needs to be nested under a `steps:` block.

##### Parameters
| Name                  | Description                                       | Type   | Default               |
|:----------------------|:--------------------------------------------------|:-------|:----------------------|
| sourcePath            | Path of the app spec file                         | string |                       |
| destinationPath       | Path(key) in the bucket to upload file to         | string |                       |
| codedeployBucketName  | Name of the bucket to upload the app spec file to | string |                       |
| workingDirectory      | Directory where the files are located             | string | $(Pipeline.Workspace) |
| serviceConnectionName | Name of the service connection for AWS            | string |                       |
| region                | Name of the AWS region                            | string |                       |

##### Example
```yaml
- template: ./aws/cli/codedeploy.deploy.yml@templates
  parameters:
    serviceConnectionName: 'AWS ap-southeast-2 production'
    region: ap-southeast-2
    sourcePath: time-1.0.23.0.yaml
    destinationPath: time/production/time-1.0.23.0.yaml
    codeDeployBucketName: 'codedeploy-sydney-production'
```

#### ECR push
To build push an image to ECR

The [ecr.push.yml](./aws/cli/ecr.push.yml) is a [step template](https://docs.microsoft.com/en-us/azure/devops/pipelines/process/templates?view=azure-devops#step-reuse) meaning it needs to be nested under a `steps:` block.

##### Parameters
| Name                  | Description                             | Type   | Default              |
|:----------------------|:----------------------------------------|:-------|:---------------------|
| serviceConnectionName | Name of the AWS credentials to use      | string |                      |
| region                | Name of the region to push to           | string |                      |
| imageName             | Name of the image to push               | string |                      |
| repositoryName        | Name of the repository to push image to | string |                      |
| tag                   | Tag of the image to push                | string | $(Build.BuildNumber) |

##### Example
```yaml
- template: ./aws/cli/ecr.push.yml@templates
  parameters:
    serviceConnectionName: 'AWS ap-southeast-2 production'
    region: ap-southeast-2
    imageName: migration
    repositoryName: migration
```


### Terraform
#### Setup
1. Install the [Terraform plugin](https://marketplace.visualstudio.com/items?itemName=ms-devlabs.custom-terraform-tasks)
2. Follow the `Create a new service connection for connecting to an AWS account` step and create an AWS service connection

#### Plan
To generate a Terraform plan for the specified infrastructure.

The [plan template](./aws/terraform/plan.yml) is a [step template](https://docs.microsoft.com/en-us/azure/devops/pipelines/process/templates?view=azure-devops#step-reuse) meaning it needs to be nested under a `steps` block.

##### Steps
###### Install version
Install the version specified in the `version` parameter. Terraform will be available through command line.

###### Init
Initialise Terraform in the `workingDirectory` using the `serviceConnectionName`, `stateBucketName`, `stateLockTableName`, `initAdditionalCommandOptions`.  

The default backend key is `terraform.tfstate`

###### Select Workspace
Will select the workspace or create and select it if it doesn't exist. The `workspaceName` will be transformed to a lower case value.

###### Plan
Plan the changes for the config in the `workingDirectory` and save it to a `plan.tfplan`. If `planAdditionalCommandOptions` is specified this will be appended to the command options.

###### Archive
Will compress the `workingDirectory` into a tar.gz file named `terraform.tar.gz`, replacing any existing archive.

Using tar prevents permission issues when the archive is used in subsequent steps.

###### Publish
Will publish the archive produced from the archive step to the pipeline with the name of the value from the `artifactName`.

##### Parameters
| Name                         | Description                                                                                                                                                                        | Type    | Default                                    |
|:-----------------------------|:-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|:--------|:-------------------------------------------|
| artifactName                 | Name of the artifact produced and published                                                                                                                                        | string  |                                            |
| initAdditionalCommandOptions | Additional command options for Terraform init                                                                                                                                      | string? |                                            |
| planAdditionalCommandOptions | Additional command options for the Terraform plan step                                                                                                                             | string? |                                            |
| serviceConnectionName        | Service connection name, used for init & plan. Name of the service connection created above in the setup step.                                                                     | string  |                                            |
| stateBucketName              | Name of the S3 bucket that Terraform uses to store state                                                                                                                           | string  |                                            |
| stateLockTableName           | Name of the DynamoDB table used by Terraform to lock state. This value is mapped to a backend config variable named `dynamodb_table` in command `commandOptions` of the init step. | string  |                                            |
| version                      | Terraform version                                                                                                                                                                  | string  | `1.0.2`                                    |
| workingDirectory             | Directory where the terraform files are stored                                                                                                                                     | string  | `$(Build.SourcesDirectory)/infrastructure` |
| workspaceName                | Terraform workspace name                                                                                                                                                           | string  |                                            |
| stateFilename                | Name and path of the state file location                                                                                                                                           | string  | terraform.tfstate                          |

##### Example
```yaml
steps:
  - template: aws/terraform/plan.yml@templates
    parameters:
      artifactName: production_ap-southeast-2
      initAdditionalCommandOptions: '-no-color'
      planAdditionalCommandOptions: '-var-file=production.ap-southeast-2.tfvars'
      serviceConnectionName: aws_ap-southeast-2
      stateBucketName: terraform-state-bucket
      stateLockTableName: terraform-state-lock-table
      version: 0.14.9
      workingDirectory: $(Build.SourcesDirectory)/infrastructure
      workspaceName: terraform-workspace
      stateFilename: terraform.tfstate
```


#### Apply
To apply changes specified in the `.tfplan` file

The [apply template](./aws/terraform/apply.yml) is a [step template](https://docs.microsoft.com/en-us/azure/devops/pipelines/process/templates?view=azure-devops#step-reuse) meaning it needs to be nested under a `steps` block.

##### Steps
###### Download Artifact
Download the pipeline artifact specified in `artifactName` to the `workingDirectory`.

###### Extract Archive
Extract the `archiveName` to `workingDirectory`

###### Install version
Install the version specified in the `version` parameter. Terraform will be available through command line.

###### Apply
Apply the changes specified in the `planFile`. If `applyAdditionalCommandOptions` is specified this will be appended to the command options.

##### Parameters
| Name                          | Description                                   | Type    | Default                 |
|:------------------------------|:----------------------------------------------|:--------|:------------------------|
| applyAdditionalCommandOptions | Additional command options for apply step     | string? |                         |
| archiveName                   | Name of the archive in the artifact           | string  | `terraform.tar.gz`      |
| artifactName                  | Name of the artifact to download              | string  |                         |
| planFile                      | Name of the plan file to apply                | string  | `plan.tfplan`           |
| serviceConnectionName         | Service connection name, used for init & plan | string  |                         |
| version                       | Terraform version                             | string  | `1.0.2`                 |
| workingDirectory              | Working directory                             | string  | `$(Pipeline.Workspace)` |

##### Example
```yaml
steps:
- template: aws/terraform/apply.yml@templates
  parameters:
    applyAdditionalCommandOptions: '-no-color'
    archiveName: terraform.tar.gz
    artifactName: production_ap-southeast-2
    planFile: plan.tfplan
    serviceConnectionName: aws_ap-southeast-2
    version: 0.14.9
    workingDirectory: $(Pipeline.Workspace)
```

#### Destroy
To destroy resources

The [destroy template](./aws/terraform/destroy.yml) is a [step template](https://docs.microsoft.com/en-us/azure/devops/pipelines/process/templates?view=azure-devops#step-reuse) meaning it needs to be nested under a `steps` block.

##### Steps
###### Install version
Install the version specified in the `version` parameter. Terraform will be available through command line.

###### Init
Initialise Terraform in the `workingDirectory` using the `serviceConnectionName`, `stateBucketName`, `stateLockTableName`, `initAdditionalCommandOptions`.  

The default backend key is `terraform.tfstate`

###### Select Workspace
Will select the workspace or create and select it if it doesn't exist. The `workspaceName` will be transformed to a lower case value.

###### Destroy
Destroy all resources specified in the in the configuration. If `destroyAdditionalArguments` is specified this will be appended to the command options.

###### Delete workspace
Will delete the workspace from the backend

##### Parameters
| Name                         | Description                                                | Type    | Default                                    |
|:-----------------------------|:-----------------------------------------------------------|:--------|:-------------------------------------------|
| version                      | Terraform version                                          | string? | `1.0.2`                                    |
| stateLockTableName           | Name of the DynamoDB table used by Terraform to lock state | string  |                                            |
| initAdditionalCommandOptions | Additional command options for Terraform init              | string? |                                            |
| destroyAdditionalArguments   | Additional command options for Terraform destroy command   | string? |                                            |
| serviceConnectionName        | Service connection name, used for init & plan              | string  |                                            |
| stateBucketName              | Name of the S3 bucket that Terraform uses to store state   | string  |                                            |
| workingDirectory             | Working directory                                          | string  | `$(Build.SourcesDirectory)/infrastructure` |
| workspaceName                | Terraform workspace name                                   | string  |                                            |
| stateFilename                | Name and path of the state file location                   | string  | terraform.tfstate                          |

##### Example
```yaml
steps:
- template: aws/terraform/apply.yml@templates
  parameters:
    version: 1.0.4
    serviceConnectionName: connection
    initAdditionalCommandOptions: 'args'
    destroyAdditionalArguments: '-var-file=variables/$(environment).$(region).tfvars'
    stateBucketName: bucket
    stateLockTableName: lock-table
    workspaceName: service-production
    stateFilename: terraform.tfstate
```

## Job
### approve.yml
```yaml
jobs:
  - template: job/approve.yml@templates
    parameters:
      dependsOn: plan
      timeoutInMinutes: 60
      notifyUsers: '[Expensely]\Expensely Team'
```

## Docker
#### Build
To build the specified target in a `Dockerfile`.

The [build template](./docker/build.yml) is a [step template](https://docs.microsoft.com/en-us/azure/devops/pipelines/process/templates?view=azure-devops#step-reuse) meaning it needs to be nested under a `steps:` block.

The `target` parameter will be used as the name.

##### Parameters
| Name           | Description             | Type   | Default              |
|:---------------|:------------------------|:-------|:---------------------|
| dockerfilePath | Path of the dockerfile  | string | Dockerfile           |
| target         | Name of target to build | string |                      |
| tag            | Tag of image            | string | $(Build.BuildNumber) |

##### Example
```yaml
- template: ./docker/build.yml@templates
  parameters: 
    dockerfilePath: src/Dockerfile
    target: build
    tag: 1.0.0-preview
```

#### Test
To run a container that will execute tests

The [test template](./docker/test.yml) is a [step template](https://docs.microsoft.com/en-us/azure/devops/pipelines/process/templates?view=azure-devops#step-reuse) meaning it needs to be nested under a `steps:` block.

The `target` parameter will be used as the name.

##### Parameters
| Name                          | Description                                                | Type   | Default                           |
|:------------------------------|:-----------------------------------------------------------|:-------|:----------------------------------|
| containerTestResultsDirectory | Path in the container to where test results are written to | string | /artifacts/tests                  |
| containerName                 | Name of the container to run                               | string | tests                             |
| coverageThreshold             | Threshold for code coverage                                | string | 60                                |
| pathToSources                 | Directory containing source code                           | string | $(System.DefaultWorkingDirectory) |
| tag                           | Tag of image                                               | string | $(Build.BuildNumber)              |
| testResultsFormat             | Format of test result/s file                               | string | VSTest                            |

##### Example
```yaml
- template: ./docker/test.yml@templates
  parameters:
    containerTestResultsDirectory: /artifacts/tests
    containerName: tests
    tag: $(Build.BuildNumber)
    pathToSources: $(System.DefaultWorkingDirectory)
    testResultsFormat: 'VSTest'
    coverageThreshold: '60'
```
