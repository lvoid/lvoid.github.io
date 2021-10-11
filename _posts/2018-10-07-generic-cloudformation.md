---
layout: post
toc: true
title: "Creating Generic CloudFormation"
categories: programming
tags: [AWS, DevOps, CloudFormation]
---

## Engineering Exquisite Clouds

So once upon a time when I was working on the AWS Cloud Engineering team at my last workplace, I stumbled across an issue I wanted to solve for our automated deployment process.  We needed to rapidly deploy and take down multiple different environment stacks in AWS, each of which might've had different ECS Task Definitions, but utilized the same core infrastructure components, such as the same VPC and ECS Cluster.  Furthermore, we sometimes wanted to redeploy a certain stack with one or more additional containers.  

Previously, we went about this by taking the entire stack down, customizing the CloudFormation deployment template to include the additions or modifications, and then putting the entire stack back up, all components recreated.

I wanted to find another way to do this using CloudFormation.  A more... *generic* way to customize an AWS stack to include **N** different containers at any time after the primary infrastructure for the stack was created.

So like any good little employee, I hopped on JIRA and created a 32 hour ticket halfway through the current two-week sprint.

## Creating Logic in a Template

Coming from my background as a programmer, I was wondering if there was some sort of way to implement a **for** loop inside of a .yaml template.  If you're familiar with .yaml templates, go ahead and laugh.  In code, this task would've been simple: create a loop for **N** number of containers to add to an ECS Task Definition, the details of each container being stores in some sort of data structure.  But this wasn't programming and I couldn't do that.

I did some research and stumbled across [CloudFormation Conditionals](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/conditions-section-structure.html).  I realized that I could utilize this feature to add conditionals at the beginning of my template that would allow an engineer to specify up to **N** different containers, the details of which could be stored in something like a properties file.

Let me explain.

## Specifying Properties

The first step in making this work was creating a highly reusable and modifiable **.properties** file, which an engineer could modify to suit their needs for a given deployment of containers to an ECS Cluster.  This properties file could then be referenced in a bash script, which would obtain the specific values it needed, passing them to the .yaml CloudFormation template, creating the stack via AWS CLI commands.

The properties file contains two types of information:

- General AWS Stack information, including the Region, Domain Name, Availability Zones, Subnets, and project specific information such as the customer, environment, and management stack name.

```bash
# Primary stack parameters
application=testapp 
customer=testcustomer 
environment=dev
management_stack_name=testapp-testcustomer-dev
region=us-east-1
ec2_key_pair=testapp-testcustomer-east-1
environment_vpc_cidr=172.32.0.0/16
cluster_size=1
domain_name=dev.testdomain.com
hosted_zone_id=ZTRY1HTFLP0Q7

app_public_subnet_acidr=172.32.1.0/24
app_public_subnet_bcidr=172.32.0.0/24
app_private_subnet_acidr=172.32.3.0/24
app_private_subnet_bcidr=172.32.2.0/24

availability_zone_a=us-east-1a
availability_zone_b=us-east-1b
availability_zone_c=us-east-1d
```

- Container specific information, including the name of the service you want to create and all pertinent specifications for the containers you want to deploy in the service.

```bash
# Customizable container definition parameters
# These container definitions all belong to same service
# If a value is not needed for a property string/integer use 'null'/-1
service_name=test-service
#service_container_name = load balancer container
service_container_name=nginx\\,null\\,null\\,null\\,null 
service_container_port=9080\\,-1\\,-1\\,-1\\,-1
container_def_number=3
container_def_names=auth-service\\,ui-service\\,server-service\\,null\\,null
container_def_vers=latest\\,57\\,48\\,-1\\,-1
container_def_mem=1024\\,1024\\,1024\\,-1\\,-1
container_def_links=null\\,null\\,null\\,null\\,null
container_def_container_ports=8080\\,80\\,8000\\,-1\\,-1
container_def_host_ports=9080\\,80\\,8000\\,-1\\,-1
container_def_protocols=tcp\\,tcp\\,tcp\\,null\\,null
```

As you can see above, we are specifying an ECS Service, `test-service`.  `service_container_name` and `service_container_port` specify the container name and the port that the ALB is forwarding traffic to, which in this case is an NGINX container that will serve the web application files.  We are then providing information for each of the containers that will go in the ECS Task Definition.

We specify 3 different containers, called `auth-service`, `ui-service`, and `server-service`, each of which correspond to a container stored in AWS ECR.  Additional information is provided for each of thr three containers, including the version tag, memory of the container, links to other containers, port mappings, and protocols.

If we do not need a value for what would be a string, we put **null**, and if we do not need a value for what would be a numerical, we need **-1**.

## Transferring Properties to the Template

In this case, a shell script serves the purpose of acquiring custom properties from our `deployment.properties` file and using the AWS CLI to either update or create a stack using the CloudFormation template.

For each property, we set a variable in the shell script which references the property in the `deployment.properties` file.

```bash
ENVIRONMENT="$environment"
CUSTOMER="$customer"
APPLICATION="$application"
MANAGEMENT_STACK_NAME="$management_stack_name"
REGION="$region"

SERVICE_NAME=$service_name
SERVICE_CONTAINER_NAME=$service_container_name
SERVICE_CONTAINER_PORT=$service_container_port
CONTAINER_DEF_NUMBER=$container_def_number
CONTAINER_DEF_NAMES=$container_def_names
CONTAINER_DEF_VERS=$container_def_vers
CONTAINER_DEF_IMAGES=$container_def_images
CONTAINER_DEF_MEM=$container_def_mem
CONTAINER_DEF_LINKS=$container_def_links
CONTAINER_DEF_CONTAINER_PORTS=$container_def_container_ports
CONTAINER_DEF_HOST_PORTS=$container_def_host_ports
CONTAINER_DEF_PROTOCOLS=$container_def_protocols
```

Using this information, we can now utilize the [AWS CLI CloudFormation commands](https://docs.aws.amazon.com/cli/latest/reference/cloudformation/) to create or update our stack in AWS.  The `--parameters` argument will take in a list of parameter key/value pairs, the key corresponding to the property in the CloudFormation template, and the value corresponding to the relevant variable we just set above.

```bash
if [ $1 == "-c" ] || [ $1 == "--create" ]; then
    aws cloudformation create-stack \
      --stack-name "$SERVICE_STACK_NAME" \
      --template-url https://s3.amazonaws.com/$APPLICATION-$CUSTOMER-$ENVIRONMENT/deployment/cloudformation-templates/template-environment-service.yml \
      --on-failure DO_NOTHING \
      --enable-termination-protection \
      --profile $CLI_PROFILE \
      --region $REGION \
      --timeout-in-minutes 180 \
      --parameters \
      ParameterKey=pEnvironment,ParameterValue="${ENVIRONMENT}" \
      ParameterKey=pCustomer,ParameterValue="${CUSTOMER}" \
      ParameterKey=pApplicationName,ParameterValue="${APPLICATION}" \
      ParameterKey=pECSServiceContainerName,ParameterValue="${SERVICE_CONTAINER_NAME}" \
      ParameterKey=pECSServiceContainerPort,ParameterValue="${SERVICE_CONTAINER_PORT}" \
      ParameterKey=pServiceName,ParameterValue="${SERVICE_NAME}" \
      ParameterKey=pContainerDefNum,ParameterValue="${CONTAINER_DEF_NUMBER}" \
      ParameterKey=pContainerDefNames,ParameterValue="${CONTAINER_DEF_NAMES}" \
      ParameterKey=pContainerDefVersions,ParameterValue="${CONTAINER_DEF_VERS}" \
      ParameterKey=pContainerDefMemory,ParameterValue="${CONTAINER_DEF_MEM}" \
      ParameterKey=pContainerDefLinks,ParameterValue="${CONTAINER_DEF_LINKS}" \
      ParameterKey=pContainerDefContainerPorts,ParameterValue="${CONTAINER_DEF_CONTAINER_PORTS}" \
      ParameterKey=pContainerDefHostPorts,ParameterValue="${CONTAINER_DEF_HOST_PORTS}" \
      ParameterKey=pContainerDefProtocol,ParameterValue="${CONTAINER_DEF_PROTOCOLS}" \
      ParameterKey=pECSCluster,ParameterValue="${ECS_CLUSTER_ID}" \
      ParameterKey=pECSServiceRole,ParameterValue="${ECS_SERVICE_ROLE}" \
      ParameterKey=pALBTargetGroup,ParameterValue="${ALB_TARGET_GROUP}" 
fi

if [ $1 == "-u" ] || [ $1 == "--update" ]; then
    aws cloudformation update-stack \
      --stack-name "$SERVICE_STACK_NAME" \
      --template-url https://s3.amazonaws.com/$APPLICATION-$CUSTOMER-$ENVIRONMENT/deployment/cloudformation-templates/template-environment-service.yml \
      --profile $CLI_PROFILE \
      --region $REGION \
      --parameters \
      ParameterKey=pEnvironment,ParameterValue="${ENVIRONMENT}" \
      ParameterKey=pCustomer,ParameterValue="${CUSTOMER}" \
      ParameterKey=pApplicationName,ParameterValue="${APPLICATION}" \
      ParameterKey=pECSServiceContainerName,ParameterValue="${SERVICE_CONTAINER_NAME}" \
      ParameterKey=pECSServiceContainerPort,ParameterValue="${SERVICE_CONTAINER_PORT}" \
      ParameterKey=pServiceName,ParameterValue="${SERVICE_NAME}" \
      ParameterKey=pContainerDefNum,ParameterValue="${CONTAINER_DEF_NUMBER}" \
      ParameterKey=pContainerDefNames,ParameterValue="${CONTAINER_DEF_NAMES}" \
      ParameterKey=pContainerDefVersions,ParameterValue="${CONTAINER_DEF_VERS}" \
      ParameterKey=pContainerDefMemory,ParameterValue="${CONTAINER_DEF_MEM}" \
      ParameterKey=pContainerDefLinks,ParameterValue="${CONTAINER_DEF_LINKS}" \
      ParameterKey=pContainerDefContainerPorts,ParameterValue="${CONTAINER_DEF_CONTAINER_PORTS}" \
      ParameterKey=pContainerDefHostPorts,ParameterValue="${CONTAINER_DEF_HOST_PORTS}" \
      ParameterKey=pContainerDefProtocol,ParameterValue="${CONTAINER_DEF_PROTOCOLS}" \
      ParameterKey=pECSCluster,ParameterValue="${ECS_CLUSTER_ID}" \
      ParameterKey=pECSServiceRole,ParameterValue="${ECS_SERVICE_ROLE}" \
      ParameterKey=pALBTargetGroup,ParameterValue="${ALB_TARGET_GROUP}" 
fi
```

## Template Hacking

Now we have our properties file, and a shell script to update or create a stack using the AWS CLI.  So now comes the main attraction.  The CloudFormation template itself.

As I mentioned before, I made significant use of the CloudFormation conditionals to create this template.  Whether or not we add a certain container to an ECS Task Definition is based on whether or not the container exists in the custom properties file, and the template has to know this, so we can do something like this:

```yaml
# container existence conditionals
TwoContainerDefs: !Not [!Equals [1, !Ref pContainerDefNum]]
ThreeContainerDefs: !And
            - !Not [!Equals [1, !Ref pContainerDefNum]]
            - !Not [!Equals [2, !Ref pContainerDefNum]]
FourContainerDefs: !And
            - !Not [!Equals [1, !Ref pContainerDefNum]]
            - !Not [!Equals [2, !Ref pContainerDefNum]]
            - !Not [!Equals [3, !Ref pContainerDefNum]]
FiveContainerDefs: !Equals [5, !Ref pContainerDefNum]
```

In case you are not familiar with CloudFormation conditionals, this block of YAML is doing the following:
1. We assume that there will always be at least 1 container when the stack is created or updated.
2. `TwoContainerDefs` evaluates to true if the number of user specified containers is not equal to 1.
3. `ThreeContainerDefs` evaluate to true if the number of user specified containers is not equal to 1 and is not equal to 2.
4. `FourContainerDefs` evaluate to true if the number of user specified containers is not equal to 1 and is not equal to 2 and is not equal to 3.
5. Finally, `FiveContainerDefs` evaluate to true if the number of user specified containers equals 5.

Makes sense, right?  This same pattern can be extended to account for **N** number of containers where **N > 1**.

These conditionals are used later in the template to evaluate whether or not we should add a given container definition to the ECS Task Definition.  We will get to that in a moment, but there are some more conditionals we have to account for.

It is in the nature of containers to differ from one another.  For example, an NGINX container may be linked to a Jetty container, while an SFTP container may be linked to neither but nonetheless run in the same ECS Service.  Similarly, it may be the case that a host or container port are not specified.  Remember in our `deployment.properties` file when we put **null** or **-1** if a certain property was not needed?  We can use this to create conditionals for the individual containers links, container ports, host ports, and protocols.

```yaml
  # container link conditionals
  NoFirstContainerDefLink: !Equals [!Select [0, !Ref pContainerDefLinks], "null"]
  NoSecondContainerDefLink: !Equals [!Select [1, !Ref pContainerDefLinks], "null"]
  NoThirdContainerDefLink: !Equals [!Select [2, !Ref pContainerDefLinks], "null"]
  NoFourthContainerDefLink: !Equals [!Select [3, !Ref pContainerDefLinks], "null"]
  NoFifthContainerDefLink: !Equals [!Select [4, !Ref pContainerDefLinks], "null"]

  # container port conditionals
  NoFirstContainerPort: !Equals [!Select [0, !Ref pContainerDefContainerPorts], -1]
  NoSecondContainerPort: !Equals [!Select [1, !Ref pContainerDefContainerPorts], -1]
  NoThirdContainerPort: !Equals [!Select [2, !Ref pContainerDefContainerPorts], -1]
  NoFourthContainerPort: !Equals [!Select [3, !Ref pContainerDefContainerPorts], -1]
  NoFifthContainerPort: !Equals [!Select [4, !Ref pContainerDefContainerPorts], -1]

  # container host port conditionals
  NoFirstContainerHostPort: !Equals [!Select [0, !Ref pContainerDefHostPorts], -1]
  NoSecondContainerHostPort: !Equals [!Select [1, !Ref pContainerDefHostPorts], -1]
  NoThirdContainerHostPort: !Equals [!Select [2, !Ref pContainerDefHostPorts], -1]
  NoFourthContainerHostPort: !Equals [!Select [3, !Ref pContainerDefHostPorts], -1]
  NoFifthContainerHostPort: !Equals [!Select [4, !Ref pContainerDefHostPorts], -1]

  #container protocol conditionals
  NoFirstContainerProtocol: !Equals [!Select [0, !Ref pContainerDefProtocol], "null"]
  NoSecondContainerProtocol: !Equals [!Select [1, !Ref pContainerDefProtocol], "null"]
  NoThirdContainerProtocol: !Equals [!Select [2, !Ref pContainerDefProtocol], "null"]
  NoFourthContainerProtocol: !Equals [!Select [3, !Ref pContainerDefProtocol], "null"]
  NoFifthContainerProtocol: !Equals [!Select [4, !Ref pContainerDefProtocol], "null"]
```

## Container Definitions

As I've mentioned before, this template assumes that we have at least one container to deploy to the Task Definition.  For the first container definition, in other words, we do not need the container existence conditional.

```yaml
- Name: !Select [0, !Ref pContainerDefNames]
  Essential: true
  Image: !Join
        - ''
        - - !Sub '${AWS::AccountId}.dkr.ecr.${AWS::Region}.amazonaws.com/testapp-'
          - !Join
            - ':'
            - - !Select [0, !Ref pContainerDefNames]
              - !Select [0, !Ref pContainerDefVersions]
  Memory: !Select [0, !Ref pContainerDefMemory]
  Links:
    !If
      - NoFirstContainerDefLink
      - !Ref AWS::NoValue
      - !Select [0, !Ref pContainerDefLinks]
  PortMappings:
    - ContainerPort:
        !If
          - NoFirstContainerPort
          - !Ref AWS::NoValue
          - !Select [0, !Ref pContainerDefContainerPorts]
      HostPort:
        !If
          - NoFirstContainerHostPort
          - !Ref AWS::NoValue
          - !Select [0, !Ref pContainerDefHostPorts]
      Protocol:
        !If
          - NoFirstContainerProtocol
          - !Ref AWS::NoValue
          - !Select [0, !Ref pContainerDefProtocol]
  LogConfiguration:
    LogDriver: awslogs
    Options:
      awslogs-group: !Join
                    - '-'
                    - - !Sub '${pApplicationName}-${pCustomer}-${pEnvironment}-${pServiceName}'
                      - !Select [0, !Ref pContainerDefNames]
      awslogs-region: !Ref AWS::Region
      awslogs-stream-prefix: !Sub awslogs-${AWS::StackName}
```

As you can see, we still utilized the conditionals for the container link, port mappings, and protocol.  For the subsequent container definitions, we will need an `!If` statement to know whether or not the container should be included.  As an example, here is the YAML for the second container:

```yaml
- !If
- TwoContainerDefs
-
  Name: !Select [1, !Ref pContainerDefNames]
  Essential: true
  Image: !Join
        - ''
        - - !Sub '${AWS::AccountId}.dkr.ecr.${AWS::Region}.amazonaws.com/testapp-'
          - !Join
            - ':'
            - - !Select [1, !Ref pContainerDefNames]
              - !Select [1, !Ref pContainerDefVersions]
  Memory: !Select [1, !Ref pContainerDefMemory]
  Links:
    !If
      - NoSecondContainerDefLink
      - !Ref AWS::NoValue
      - !Select [1, !Ref pContainerDefLinks]
  PortMappings:
    - ContainerPort:
        !If
          - NoSecondContainerPort
          - !Ref AWS::NoValue
          - !Select [1, !Ref pContainerDefContainerPorts]
      HostPort:
        !If
          - NoSecondContainerHostPort
          - !Ref AWS::NoValue
          - !Select [1, !Ref pContainerDefHostPorts]
      Protocol:
        !If
          - NoSecondContainerProtocol
          - !Ref AWS::NoValue
          - !Select [1, !Ref pContainerDefProtocol]
  LogConfiguration:
    LogDriver: awslogs
    Options:
      awslogs-group: !Join
                    - '-'
                    - - !Sub '${pApplicationName}-${pCustomer}-${pEnvironment}-${pServiceName}'
                      - !Select [1, !Ref pContainerDefNames]
      awslogs-region: !Ref AWS::Region
      awslogs-stream-prefix: !Sub awslogs-${AWS::StackName}
- !Ref AWS::NoValue
```

The container definition is added only if our conditional we previously defined evaluates to true.  Otherwise, we just pass in an **AWS::NoValue**.

This general pattern can be extended to all subsequent container definitions.

After the container definitions, we can also add custom CloudWatch log groups which are added only if the container existence condition holds true:

```yaml
CloudWatchLogsGroupOne:
  Type: AWS::Logs::LogGroup
  Properties:
    LogGroupName: !Join
                  - '-'
                  - - !Sub '${pApplicationName}-${pCustomer}-${pEnvironment}-${pServiceName}'
                    - !Select [0, !Ref pContainerDefNames]
    RetentionInDays: 365

CloudWatchLogsGroupTwo:
  Type: AWS::Logs::LogGroup
  Condition: TwoContainerDefs
  Properties:
    LogGroupName: !Join
                  - '-'
                  - - !Sub '${pApplicationName}-${pCustomer}-${pEnvironment}-${pServiceName}'
                    - !Select [1, !Ref pContainerDefNames]
    RetentionInDays: 365
```

## Afterword

When I discovered that you could engineer a CloudFormation template to have this kind of behavior, I was pretty captivated.  I encourage you to implement and improve upon this design when writing your own CloudFormation templates to further speed up and automate your deployments.
