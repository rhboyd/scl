CI/CD Workshop
# Getting started
## Create a Cloud9 environment
## Update AWS SAM
```bash
sh -c "$(curl -fsSL https://raw.githubusercontent.com/Linuxbrew/install/master/install.sh)"
test -d ~/.linuxbrew && eval $(~/.linuxbrew/bin/brew shellenv)
test -d /home/linuxbrew/.linuxbrew && eval $(/home/linuxbrew/.linuxbrew/bin/brew shellenv)
test -r ~/.bash_profile && echo "eval \$($(brew --prefix)/bin/brew shellenv)" >>~/.bash_profile
echo "eval \$($(brew --prefix)/bin/brew shellenv)" >>~/.profile

npm uninstall -g aws-sam-local
sudo pip uninstall aws-sam-cli
rm -rf $(which sam)

brew tap aws/tap
brew install aws-sam-cli
/home/linuxbrew/.linuxbrew/bin/sam
ln -sf $(which sam) ~/.c9/bin/sam
ls -la ~/.c9/bin/sam

sudo ln -s /usr/bin/pip-3.6 /usr/bin/pip3
pip3 install cfn-lint

```

## Create Repository
```bash
aws codecommit create-repository --repository-name scl-2019
git config --global credential.helper '!aws codecommit credential-helper $@'
git config --global credential.UseHttpPath true
git config --global user.name "Richard Boyd"
git config --global user.email richard@rboyd.dev
git clone https://git-codecommit.us-east-1.amazonaws.com/v1/repos/scl-2019
cd scl-2019
```

## Create our workspaces
```bash
mkdir pipeline
mkdir application
cd application
```

## Create our sample application
```bash
sam init --runtime python3.7 -n MyServerlessApp
# Choose '1' for AWS Quick Start Templates
```

## Create the buildspec.yml file
```bash
version: 0.2

phases:
  install:
    runtime-versions:
      python: 3.7
    commands:
      - pip install --user aws-sam-cli
      - USER_BASE_PATH=$(python -m site --user-base)
      - export PATH=$PATH:$USER_BASE_PATH/bin
  #pre_build:
    #commands:
      # - command
      # - command
  build:
    commands:
      - cd application/MyServerlessApp/
      - sam build
      - aws cloudformation package --template-file .aws-sam/build/template.yaml --s3-bucket $S3Bucket --s3-prefix sample-lambda/codebuild --output-template-file samtemplate.yaml
  #post_build:
    #commands:
      # - command
      # - command
artifacts:
  files:
    - MyServerlessApp/samtemplate.yaml
  name: $(date +%Y-%m-%d)
  discard-paths: yes
  base-directory: './application'
#cache:
  #paths:
    # - paths
```

## Create Pipeline Template
```yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: CodePipeline for the Sample Lambda Function
Parameters:
  ProjectName:
    Description: Name of the Project
    Type: String
    Default: scl-2019
  CustomName:
      Description: Name of the Project
      Type: String
      Default: richard
Resources:
  ArtifactBucket:
    Type: AWS::S3::Bucket
    DeletionPolicy: Retain
  BuildProjectRole:
    Type: AWS::IAM::Role
    Properties:
      RoleName: !Sub ${ProjectName}-CodeBuildRole-${CustomName}
      AssumeRolePolicyDocument:
        Version: 2012-10-17
        Statement:
          -
            Effect: Allow
            Principal:
              Service:
                - codebuild.amazonaws.com
            Action:
              - sts:AssumeRole
      Path: /
  BuildProjectPolicy:
    Type: AWS::IAM::Policy
    DependsOn: S3BucketPolicy
    Properties:
      PolicyName: !Sub ${ProjectName}-CodeBuildPolicy-${CustomName}
      PolicyDocument:
        Version: 2012-10-17
        Statement:
          -
            Effect: Allow
            Action:
              - s3:PutObject
              - s3:GetObject
              - s3:GetObjectVersion
            Resource: *
          -
            Effect: Allow
            Action:
              - logs:CreateLogGroup
              - logs:CreateLogStream
              - logs:PutLogEvents
            Resource: arn:aws:logs:*:*:*
      Roles:
        - !Ref BuildProjectRole
  BuildProject:
    Type: AWS::CodeBuild::Project
    Properties:
      Name: scl-build-project
      Description: my description
      ServiceRole: !GetAtt BuildProjectRole.Arn
      Artifacts:
        Type: CODEPIPELINE
      Environment:
        Type: LINUX_CONTAINER
        ComputeType: BUILD_GENERAL1_SMALL
        Image: aws/codebuild/standard:2.0
        EnvironmentVariables:
          - Name: S3Bucket
            Value: !Ref ArtifactBucket
      Source:
        Type: CODEPIPELINE
      TimeoutInMinutes: 10

  PipeLineRole:
    Type: AWS::IAM::Role
    Properties:
      RoleName: !Sub ${ProjectName}-codepipeline-role-${CustomName}
      AssumeRolePolicyDocument:
        Version: 2012-10-17
        Statement:
          -
            Effect: Allow
            Principal:
              Service:
                - codepipeline.amazonaws.com
            Action:
              - sts:AssumeRole
      Path: /
  PipelinePolicy:
    Type: AWS::IAM::Policy
    DependsOn: S3BucketPolicy
    Properties:
      PolicyName: !Sub ${ProjectName}-codepipeline-policy-${CustomName}
      PolicyDocument:
        Version: 2012-10-17
        Statement:
          -
            Effect: Allow
            Action:
              - codepipeline:*
              - iam:ListRoles
              - cloudformation:Describe*
              - cloudFormation:List*
              - codecommit:List*
              - codecommit:Get*
              - codecommit:*
              - codecommit:GitPull
              - codecommit:UploadArchive
              - codecommit:CancelUploadArchive
              - codebuild:BatchGetBuilds
              - codebuild:StartBuild
              - cloudformation:CreateStack
              - cloudformation:DeleteStack
              - cloudformation:DescribeStacks
              - cloudformation:UpdateStack
              - cloudformation:CreateChangeSet
              - cloudformation:DeleteChangeSet
              - cloudformation:DescribeChangeSet
              - cloudformation:ExecuteChangeSet
              - cloudformation:SetStackPolicy
              - cloudformation:ValidateTemplate
              - iam:PassRole
              - s3:ListAllMyBuckets
              - s3:GetBucketLocation
            Resource:
              - "*"
          -
            Effect: Allow
            Action:
              - s3:PutObject
              - s3:GetBucketPolicy
              - s3:GetObject
              - s3:ListBucket
            Resource:
             - !Sub 'arn:aws:s3:::${ArtifactBucket}/*'
             - !Sub 'arn:aws:s3:::${ArtifactBucket}'
      Roles:
        -
          !Ref PipeLineRole


  CfnDeployRole:
    Type: AWS::IAM::Role
    Properties:
      RoleName: !Sub ${ProjectName}-cfn-role-${CustomName}
      AssumeRolePolicyDocument:
        Version: 2012-10-17
        Statement:
          -
            Effect: Allow
            Principal:
              Service:
                - cloudformation.amazonaws.com
            Action:
              - sts:AssumeRole
      Path: /
  CfnDeployPolicy:
    Type: AWS::IAM::Policy
    Properties:
      PolicyName: !Sub ${ProjectName}-cfn-policy-${CustomName}
      PolicyDocument:
        Version: 2012-10-17
        Statement:
          -
            Effect: Allow
            Action: "*"
            Resource: "*"
      Roles:
        - !Ref CfnDeployRole


  Pipeline:
    Type: AWS::CodePipeline::Pipeline
    Properties:
      RoleArn: !GetAtt PipeLineRole.Arn
      Name: !Ref AWS::StackName
      Stages:
        - Name: Source
          Actions:
            - Name: App
              ActionTypeId:
                Category: Source
                Owner: AWS
                Version: "1"
                Provider: CodeCommit
              Configuration:
                RepositoryName: !Ref ProjectName
                BranchName: master
              OutputArtifacts:
                - Name: SCCheckoutArtifact
              RunOrder: 1
        -
          Name: Build
          Actions:
          -
            Name: Build
            ActionTypeId:
              Category: Build
              Owner: AWS
              Version: "1"
              Provider: CodeBuild
            Configuration:
              ProjectName: !Ref BuildProject
            RunOrder: 1
            InputArtifacts:
              - Name: SCCheckoutArtifact
            OutputArtifacts:
              - Name: BuildOutput
        - Name: DeployToTest
          Actions:
            - Name: CreateChangeSetTest
              ActionTypeId:
                Category: Deploy
                Owner: AWS
                Version: "1"
                Provider: CloudFormation
              Configuration:
                ChangeSetName: !Sub sample-lambda-dev-${CustomName}
                ActionMode: CHANGE_SET_REPLACE
                StackName: !Sub sample-lambda-dev-${CustomName}
                Capabilities: CAPABILITY_NAMED_IAM
                RoleArn: !GetAtt CfnDeployRole.Arn
                TemplatePath: BuildOutput::samtemplate.yaml
              InputArtifacts:
                - Name: BuildOutput
              RunOrder: 1
            - Name: DeployChangeSetTest
              ActionTypeId:
                Category: Deploy
                Owner: AWS
                Version: "1"
                Provider: CloudFormation
              Configuration:
                ChangeSetName: !Sub sample-lambda-dev-${CustomName}
                ActionMode: CHANGE_SET_EXECUTE
                StackName: !Sub sample-lambda-dev-${CustomName}
              InputArtifacts:
                - Name: BuildOutput
              RunOrder: 2
      ArtifactStore:
        Type: S3
        Location: !Ref ArtifactBucket

  S3BucketPolicy:
    Type: AWS::S3::BucketPolicy
    Properties:
      Bucket: !Ref ArtifactBucket
      PolicyDocument:
        Statement:
          -
            Action:
              - s3:*
            Effect: Allow
            Resource:
              - !Sub arn:aws:s3:::${ArtifactBucket}
              - !Sub arn:aws:s3:::${ArtifactBucket}/*
            Principal:
              AWS:
                - !GetAtt [BuildProjectRole,Arn]
```

## Create a Template to handle off-master builds

We'll call this auto-build.yaml
```yaml
Transform: AWS::Serverless-2016-10-31
Parameters:
  ProjectName:
    Description: Name of the Project
    Type: String
    Default: scl-2019
Resources:
  MyFunction:
    Type: AWS::Serverless::Function
    Properties:
      InlineCode: |
        import boto3
        import json
        import os
        cb = boto3.client( 'codebuild' )
        def handler(event, context):
          if event['detail']['referenceName'] == 'master':
            return
          build = {
              'projectName': os.environ['BUILD_PROJECT_NAME'],
              'sourceVersion': event['detail']['commitId'],
              'artifactsOverride': {
                'type': 'S3',
                'location': os.environ['ARTIFACT_BUCKET_NAME'],
                'path': 'builds/',
                'name': event['detail']['commitId'],
                'packaging': 'ZIP',
                'overrideArtifactName': True
              },
              'sourceTypeOverride': 'CODECOMMIT',
              'sourceLocationOverride': os.environ['SOURCE_LOCATION'],
          }
          print( 'Starting build for project {0} from commit ID {1}...'.format( build['projectName'], build['sourceVersion'] ) )
          response = cb.start_build( **build )
          print( 'Successfully launched build.' )
          return response
      Handler: index.handler
      Runtime: python3.7
      Policies:
       - AWSLambdaExecute
       - Version: '2012-10-17'
         Statement:
           - Effect: Allow
             Action:
               - codecommit:*
               - codebuild:*
             Resource: '*'
      Events:
        CodeBuildEvent:
          Type: CloudWatchEvent
          Properties:
            Pattern:
              source:
                - "aws.codecommit"
              detail-type:
                - "CodeCommit Repository State Change"
              resources:
                - !Sub "arn:aws:codecommit:${AWS::Region}:${AWS::AccountId}:${ProjectName}"
      Environment:
        Variables:
          BUILD_PROJECT_NAME: "scl-build-project"
          ARTIFACT_BUCKET_NAME: "pipeline-artifactbucket-1s3qh0v7cf63i"
          SOURCE_LOCATION: !Sub https://git-codecommit.${AWS::Region}.amazonaws.com/v1/repos/${ProjectName}
```
This can then be deployed with the following command:
```bash
aws cloudformation deploy --template-file autobuild.yaml --stack-name autobuild --capabilities CAPABILITY_IAM
```

## Add Cloudwatch Event to trigger Lambda Function
```yaml
EventRule:
  Type: AWS::Events::Rule
  Properties:
    Description: "New Commit added"
    EventPattern:
      source:
        - "aws.codecommit"
      detail-type:
        - "CodeCommit Repository State Change"
      detail:
        state:
          - "stopping"
    State: "ENABLED"
    Targets:
      - Arn: !GetAtt LambdaFunction.Arn
        Id: "TargetFunctionV1"
PermissionForEventsToInvokeLambda:
  Type: AWS::Lambda::Permission
  Properties:
    FunctionName:
      Ref: "LambdaFunction"
    Action: "lambda:InvokeFunction"
    Principal: "events.amazonaws.com"
    SourceArn:
      Fn::GetAtt:
        - "EventRule"
        - "Arn"
```
