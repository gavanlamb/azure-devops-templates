# Azure DevOps Templates
This repository contains templates for Azure DevOps.

To use the templates add a resource section to the top of the pipeline file.  
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
#### ECR push
To build push an image to ECR

The [ecr.push](./docker/build.yml) is a [step template](https://docs.microsoft.com/en-us/azure/devops/pipelines/process/templates?view=azure-devops#step-reuse) meaning it needs to be nested under a `steps:` block.

##### Parameters
| Name              | Description                             | Type   | Default              |
|:------------------|:----------------------------------------|:-------|:---------------------|
| awsCredentials    | Name of the AWS credentials to use      | string |                      |
| awsRegion         | Name of the region to push to           | string |                      |
| imageName         | Name of the image to push               | string |                      |
| repositoryName    | Name of the repository to push image to | string |                      |
| tag               | Tag of the image to push                | string | $(Build.BuildNumber) |

##### Example
```yaml
- template: ./aws/cli/ecr.push.yml@templates
  parameters:
    awsCredentials: 'AWS ap-southeast-2 production'
    awsRegion: ap-southeast-2
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
| version                      | Terraform version                                                                                                                                                                  | string  | `0.14.9`                                   |
| workingDirectory             | Directory where the terraform files are stored                                                                                                                                     | string  | `$(Build.SourcesDirectory)/infrastructure` |
| workspaceName                | Terraform workspace name                                                                                                                                                           | string  |                                            |

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
| version                       | Terraform version                             | string  | `0.14.9`                |
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
