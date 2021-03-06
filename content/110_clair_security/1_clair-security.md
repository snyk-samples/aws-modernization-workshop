+++
title = "Security Modernization: Image scanning"
chapter = false
weight = 1
+++

## Security Modernization: Image scanning

### Expected Outcome:*
* Understanding of a image scanning
* Deploy and envoke Clair scanning

**Lab Requirements:**
* Existing Code9 environment.
* Completed Labs: 
* container-orchestration
* continuous-integration
* continuous-delivery
* CICD
* Artifactory

**Average Lab Time:** 30 minutes

### Introduction
In this module, we will deploy https://github.com/coreos/clair/[Clair] to your AWS account using a CloudFormation template.
For our lab, we will be provisioning Clair as a Docker image running on AWS Fargate.

Clair is an image scanning solution which detects vulnerabilities with container images, image dependencies.
Security teams and developers alike can use Clair to scan resources for security risks. Development teams can do this on an adhoc basis while building to see if there are concerns with the resources they are consuming. 

Security teams can perform scans against resources, work with developement teams to address the concerns and then publish pre-approved or white listed resources to a repository which developers can consume with confidence.

Implementing image scanning as part of a DevSecOps process is the first step in protecting against risks from 3rd party sources. Consuming images not build by you, there is often not a clear understanding of the components, layers and dependencies and the posible risks related to these.

For more detail regarding Clair see https://github.com/coreos/clair[https://github.com/coreos/clair]

#### Getting Started with Clair container image scanner.

Steps to Deploy Clair using a CloudFormation template via CLI:

**Step 1:** Switch to the tab where you have your Cloud9 environment opened and change to this modules directory by running:
```
cd ~/environment/aws-modernization-workshop/modules/security
```

**Step 2:** Deploy the CloudFormation template.
```
aws cloudformation create-stack --stack-name "Clair" \
--template-body file://clair-new-vpc.yaml \ 
--capabilities CAPABILITY_NAMED_IAM
```

**Step 3:** Optional: check if the stack has completed, you do not need to wait for this you can proceed with next steps while this builds.
```
until [[ `aws cloudformation describe-stacks --stack-name "Clair" --query "Stacks[0].[StackStatus]" --output text` == "CREATE_COMPLETE" ]]; do  echo "The stack is NOT in a state of CREATE_COMPLETE at `date`";   sleep 30; done && echo "The Stack is built at `date` - Please proceed"
```
**NOTE:** *It may take between 10 to 15 minutes for the stack to complete.*

**Step 4:** Go to the https://console.aws.amazon.com/ecs[Amazon ECS console], select your cluster (e.g. Clair), select the *Tasks* tab. Now select the *Logs* Tab, you will see the events for the Clair health service, database migrations and update of the vulnerabilities databases.
This step will take some time to complete. Should the scanner be used before this update is complete, many security risks may not be detected.

For more detail regarding Clair see: https://github.com/coreos/clair[https://github.com/coreos/clair]


#### Invoking Clair scanner:
We will use a commandline tool Klar to invoke a Clair scan.
Klar is a simple tool to analyze images stored in a private or public Docker registry for security vulnerabilities using Clair.

For more detail regarding Klar see https://github.com/optiopay/klar[https://github.com/optiopay/klar]

**Step 5:** Switch to the tab where you have your Cloud9 environment opened and change to this modules directory by running:

**Step 6:** Download Klar binary.
```
curl -kLO https://github.com/optiopay/klar/releases/download/v2.3.0/klar-2.3.0-linux-amd64
```

**Step 7:** Set execute permisions.
```
chmod +x ./klar-2.3.0-linux-amd64
```

**Step 8:** Rename binary for simpler command execution.
```
mv ./klar-2.3.0-linux-amd64 ./klar
```

**Step 9:** Collect the IP of the Clair Fargate instance.
```
aws ec2 describe-network-interfaces \ 
--network-interface-ids=$(aws ecs describe-tasks --cluster=Clair \ 
--tasks=`aws ecs list-tasks --cluster=Clair --query taskArns[0] --output=text` \
--query tasks[0].attachments[0].details[1].value --output=text) \
--query NetworkInterfaces[0].Association.PublicIp --output=text
```

This IP will be combined with the tcp port 6060 (e.g. x.x.x.x:6060).

**Step 10:** Manual scan of container image in Dockerhub (this step will require the Clair CloudFormation stack launched above to be in a Create Complete State).
```
CLAIR_ADDR=<YOUR CLAIR FARGATE INSTACE IP >:6060 ./klar debian:9
```

Note this will display the number of vulnerabilities, number of high risks, the spacifics of the risk and mitigations.

**Step 11:** Manual scan of Dockerhub image with json output.
```
JSON_OUTPUT=true CLAIR_ADDR=<YOUR CLAIR FARGATE INSTACE IP >:6060 ./klar debian:9 > outputs/report.json
```

This will allow you to comsume the json into other solutions and audit systems.

**Step 12:** Manual scan of MySQL.
```
CLAIR_ADDR=<YOUR CLAIR FARGATE INSTACE IP>:6060 ./klar mysql/mysql-server
```

In both the above scans you are able to to detect risks within artifacts devleopment teams may be consuming within their projects.
Security teams can make use of this type of scanning in an asynconous fashion to scan resources , address concerns and then publish white listed resources for developement teams to consume with confidence.

These pre-approved resources could be stored within Artifactory as seen in previsous labs.

#### Integration of Klar and Clair with AWS ECR.

**Lab Requirements:**
* Existing VPC.
* Existing Code9 environment.
* Existing ECR repo with images pushed to it, you can complete the containerize-application module.

AWS ECR does not use perminant credentials, these must be retrived using aws ecr get-login and they are valid for 12 hours.

```
DOCKER_LOGIN=`aws ecr get-login --no-include-email`
PASSWORD=`echo $DOCKER_LOGIN | cut -d' ' -f6`
DOCKER_USER=AWS DOCKER_PASSWORD=${PASSWORD} ./klar <Repository URI:TAG>
```

We have put this together into a simple script which accepts the Repository URI and TAG as an input
**Step 12:** First lets modify the permisions on the script.
```
chmod +x ./scan.sh
```

**Step 13:** colect the `Repository URI:TAG`
*Go to the Amazon ECS console, select Repositories, then select petstore_postgres.
You will see the Repository URI listed at the top and the tag at the bottom.
These should be combined Repository URI:TAG*

**Step 11:** execute the script to scan image on ECR repository.
```
./scan.sh <Repository URI:TAG> 
```

#### Integrating image scanning into CICD.
This process can be integrated into the CICD process by adding the Klar instructions into the buildspec.yml used by AWS Code Build to build the images.

The following is the buildspec.yml used in the CICD lab:

```
version: 0.2
phases:
  pre_build:
    commands:
      - echo Logging in to Amazon ECR...
      - aws --version
      - $(aws ecr get-login --region $AWS_DEFAULT_REGION --no-include-email)
      - REPOSITORY_URI=$(aws ecr describe-repositories --repository-name petstore_frontend --query=repositories[0].repositoryUri --output=text)
      - COMMIT_HASH=$(echo $CODEBUILD_RESOLVED_SOURCE_VERSION | cut -c 1-7)
      - IMAGE_TAG=${COMMIT_HASH:=latest}
      - PWD=$(pwd)              
  build:
    commands:
      - echo Build started on `date`
      - echo Building the Docker image...
      - cd modules/containerize-application
      - docker build -t $REPOSITORY_URI:latest .
      - docker tag $REPOSITORY_URI:latest $REPOSITORY_URI:$IMAGE_TAG
  post_build:
    commands:
      - echo Build completed on `date`
      - echo Pushing the Docker images...
      - docker push $REPOSITORY_URI:latest
      - docker push $REPOSITORY_URI:$IMAGE_TAG
      - echo Writing image definitions file...
      - echo Source DIR ${CODEBUILD_SRC_DIR}
      - printf '[{"name":"petstore","imageUri":"%s"}]' $REPOSITORY_URI:$IMAGE_TAG > ${CODEBUILD_SRC_DIR}/imagedefinitions.json        
```

The following would need to be added to the pre_build step:
```
      - echo Setting up Clair client Klar
      - curl -kLO https://github.com/optiopay/klar/releases/download/v2.3.0/klar-2.3.0-linux-amd64
      - chmod +x ./klar-2.3.0-linux-amd64
      - mv ./klar-2.3.0-linux-amd64 ./klar
      - DOCKER_LOGIN=`aws ecr get-login --region $AWS_DEFAULT_REGION`
      - PASSWORD=`echo $DOCKER_LOGIN | cut -d' ' -f6`
      - mkdir outputs
```

The following would need to be run post build:
```
      - bash -c "if [ /"$CODEBUILD_BUILD_SUCCEEDING/" == /"0/" ]; then exit 1; fi"
      - echo Build stage successfully completed on `date`
      - echo Pushing the Docker image...
      - docker push $IMAGE_URI
      - echo Running Clair scan on the image
      - DOCKER_USER=AWS DOCKER_PASSWORD=${PASSWORD} CLAIR_ADDR=$CLAIR_URL ../klar $REPOSITORY_URI:$IMAGE_TAG
```

The final product would look something like:
```
version: 0.2
phases:
  pre_build:
    commands:
      - echo Logging in to Amazon ECR...
      - aws --version
      - $(aws ecr get-login --region $AWS_DEFAULT_REGION --no-include-email)
      - REPOSITORY_URI=$(aws ecr describe-repositories --repository-name petstore_frontend --query=repositories[0].repositoryUri --output=text)
      - COMMIT_HASH=$(echo $CODEBUILD_RESOLVED_SOURCE_VERSION | cut -c 1-7)
      - IMAGE_TAG=${COMMIT_HASH:=latest}
      - PWD=$(pwd) 
      - echo Setting up Clair client Klar
      - wget https://github.com/optiopay/klar/releases/download/v2.3.0/klar-2.3.0-linux-amd64
      - chmod +x ./klar-2.3.0-linux-amd64
      - mv ./klar-2.3.0-linux-amd64 ./klar
      - DOCKER_LOGIN=`aws ecr get-login --region $AWS_DEFAULT_REGION`
      - PASSWORD=`echo $DOCKER_LOGIN | cut -d' ' -f6`
      - mkdir outputs             
  build:
    commands:
      - echo Build started on `date`
      - echo Building the Docker image...
      - cd modules/containerize-application
      - docker build -t $REPOSITORY_URI:latest .
      - docker tag $REPOSITORY_URI:latest $REPOSITORY_URI:$IMAGE_TAG
  post_build:
    commands:
      - bash -c "if [ /"$CODEBUILD_BUILD_SUCCEEDING/" == /"0/" ]; then exit 1; fi"
      - echo Build completed on `date`
      - echo Pushing the Docker images...
      - docker push $REPOSITORY_URI:latest
      - docker push $REPOSITORY_URI:$IMAGE_TAG
      - echo Running Clair scan on the image
      - DOCKER_USER=AWS DOCKER_PASSWORD=${PASSWORD} CLAIR_ADDR=$CLAIR_URL ../klar $REPOSITORY_URI:$IMAGE_TAG
      - echo Writing image definitions file...
      - echo Source DIR ${CODEBUILD_SRC_DIR}
      - printf '[{"name":"petstore","imageUri":"%s"}]' $REPOSITORY_URI:$IMAGE_TAG > ${CODEBUILD_SRC_DIR}/imagedefinitions.json   
```
