# Building CI/CD pipelines for lambda canary deployments using AWS CDK with Python

(https://catalog.us-east-1.prod.workshops.aws/workshops/5195ab7c-5ded-4ee2-a1c5-775300717f42/en-US)

## App

API Gateway ---> Lambda

## Pipeline

CodeCommit ---> git clone ---> cdk bootstrap ---> S3 (cdk artifacts)
							   cdk deploy    ---> AWS Service Stack
							   
## Create cdk project

```
$ mkdir cdk_workshop && cd cdk_workshop							   
$ cdk init sample-app --language python						# automatically creates a venv
$ source .venv/bin/activate
$ python -m pip3 install --upgrade pip
$ pip3 install -r requirements.txt
```

The sample app just creates an SQS queue and an SNS topic
see cdk-workshop/cdk_workshop/cdk_workshop_stack.py

Modify cdk_workshop_stack.py (this is where most of the code will go)...

## Define lamba function

```

        my_lambda = Function(
            scope = self,
            id = "MyFunction",
            function_name= context["lambda"]["name"],
            handler = "handler.lambda_handler",
            runtime = Runtime.PYTHON_3_9,
            code = Code.from_asset("lambda"),
            current_version_options = VersionOptions(
                description = f'Version deployed on {current_date}',
                removal_policy = RemovalPolicy.RETAIN
            )
        )
```
		
## Define API gateway

```
        LambdaRestApi(
            scope = self,
            id = "RestAPI",
            handler = alias,
            deploy_options = StageOptions(stage_name=self.stage_name)
        )		
```
		
## And a deployment group

```
        LambdaDeploymentGroup(
            scope = self,
            id = "CanaryDeployment",
            alias = alias,
            deployment_config = LambdaDeploymentConfig.CANARY_10_PERCENT_5_MINUTES,
            alarms = [failure_alarm]
        )		
```
		
## Then create the lambda source file (linked from your function definition above)

```
lambda/handler.py

import json

def lambda_handler(event, context):
    return {
        'statusCode': 200,
        'body': json.dumps('Hello from CDK!')
    }
```

## Add config to /cdk.json

```
# use this file to set up variables for different environments (qa and prod)

  "context": {
    "prefix": "cdk-workshop-stack",
    "qa": {
      "region": "us-east-1",
      "lambda": {
        "name": "cdk-workshop-function-qa",
        "alias": "live",
        "stage": "qa"
      },
      "tags": {
        "App":"cdk-workshop",
        "Environment": "QA",
        "IaC": "CDK"
      }
    },
    "prod": {

    }
  }	
```
  
can access this information in the code...

```
        environment_type = self.node.try_get_context("environmentType")
        context = self.node.try_get_context(environment_type)
        self.alias_name = context["lambda"]["alias"]						# live
        self.stage_name = context["lambda"]["stage"]						# qa
		
		function_name= context["lambda"]["name"]							# cdk-workshop-function-qa

```
		
note that environmentType gets passed to the bootstrap command as an argument later...

cdk.json also defines how to run the app:

```
	"app": "python3 app.py",
```

## Bootstrap the app

Only need to do this once		

```		
$ export AWS_DEFAULT_PROFILE=default		
$ export REGION=us-east-1
$ export PROFILE=default
$ export ACCOUNT=$(aws sts get-caller-identity --profile $PROFILE | jq -r .Account)
$ cdk bootstrap aws://$ACCOUNT/$REGION -c account=$ACCOUNT -c environmentType=qa --profile $PROFILE	
```

(had to create new creds for admin)	

## Synth the app

```
$ cdk synth -c account=$ACCOUNT -c environmentType=qa --profile $PROFILE
```

this executes the code

--> CloudFormation artifacts written to JSON files in ...
	/cdk.out/...
	
can also redirect to yaml:

```
$ cdk synth -c account=$ACCOUNT -c environmentType=qa --profile $PROFILE >> template.yaml
```

then use cfn_nag to check for any bad practices in the generated template (see further down)

(https://github.com/stelligent/cfn_nag)

## Deploy the app

```
$ cdk deploy -c account=$ACCOUNT -c environmentType=qa --profile $PROFILE
```

## Clean up

```
$ cdk destroy -c account=$ACCOUNT -c environmentType=qa --profile $PROFILE
```

## Unit Testing & Linting

use cdk assertions (https://docs.aws.amazon.com/cdk/api/v2/docs/aws-cdk-lib.assertions-readme.html)
to check that the generated CloudFormation template is as expected

Fine-grained assertions: test specific aspects of the generated AWS CloudFormation template, such as "this resource has this property with this value."

add libraries to requirements.txt...

```
pytest
coverage
pylint
safety
```

add tests to tests/unit/test_...py

run:
```
$ python -m pytest									# should discover the tests automatically
```

check code coverage:
```
$ python -m coverage run --branch --source=cdk_workshop -m pytest
```

check linting:
```
$ python -m pylint cdk_workshop
```

check for libraries with known vulnerabilities:
```
$ safety check
```

## Using Cfn_Nag to check for bad practices in generated CloudFormation template

```
$ cdk synth -c account=$ACCOUNT -c environmentType=qa --profile $PROFILE >> template.yaml

$ cat <<EOF>> cfn_nag_supress.yaml
RulesToSuppress:
- id: W28
  reason: not applicable to this workshop
- id: W28
  reason: not applicable to this workshop
- id: W68
  reason: not applicable to this workshop
- id: W59
  reason: not applicable to this workshop
- id: W69
  reason: not applicable to this workshop
- id: W64
  reason: not applicable to this workshop
- id: W58
  reason: not applicable to this workshop
- id: W89
  reason: not applicable to this workshop
- id: W92
  reason: not applicable to this workshop
EOF

# install
$ gem install cfn-nag

# run
$ cfn_nag_scan --input-path template.yaml --deny-list-path cfn_nag_supress.yaml
```
## Using CodeBuild to build the stack

Create a buildspec in the project root

```
$ cat <<EOF >> buildspec.yml
version: 0.2
phases:
  install:
    runtime-versions:
      python: 3.8
      nodejs: 12
    commands:
      - npm install -g aws-cdk
      - cdk --version
      - python3 -m venv .env
      - chmod +x .env/bin/activate
      - . .env/bin/activate
      - pip3 install -r requirements.txt
  pre_build:
    commands:
      - ACCOUNT=$(aws sts get-caller-identity | jq -r '.Account')
  build:
    commands:
      - cdk deploy -c account=$ACCOUNT -c environmentType=qa --require-approval never
EOF
```

## Creating a CodePipeline in CDK

Then create a new project for the pipeline:

```
$ cd ..
$ mkdir cdk-workshop-pipeline && cd cdk-workshop-p*
$ cdk init sample-app --language python
$ source .venv/bin/activate
$ pip3 install --upgrade pip
$ pip3 install -r requirements.txt
```

In cdk_workshop_pipeline:

Create a GitHubSourceAction:

```
		# Source stage
        self.source_stage.add_action(
            GitHubSourceAction(
                action_name = "GitHub_source",
                output = source_output,
                owner = self.context["github"]["owner"],
                repo = self.context["github"]["repositoryName"],
                oauthToken = SecretValue.secretsManager(self.context["github"]["oauthSecretName"]),
                branch = "main"
            )
        )
```

Source the repo details from cdk.json:

```
{
  "app": "python3 app.py",
  "context": {
    "qa": {
      "region": "us-east-1",
      "pipeline": {
        "buildSpecLocation": "buildspec.yml",
        "bucketName": "cdk-workshop-assets",
        "targetStack": "cdk-workshop-stack-qa" 
      },
      "github": {
        "repositoryName": "cdk-python-workshop",
        "owner": "freelandr",
        "oauthSecretName": "github-oauth-token"
      }
    }
  }
}
```

Add the other stages:
- code validation:
 - pylint, unit test, etc
- build (to use the buildspec.yml file from main project which includes the deploy command)

Finally, deploy the pipeline:

```
$ cdk deploy -c account=$ACCOUNT -c environmentType=qa --profile $PROFILE
```
