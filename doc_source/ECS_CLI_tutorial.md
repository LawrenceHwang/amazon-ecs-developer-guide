# Amazon ECS CLI Tutorial<a name="ECS_CLI_tutorial"></a>

These simple tutorials show a few of the different commands and capabilities of the Amazon ECS CLI\. We provide tutorials using both the Fargate and EC2 launch types\. Before you can start this tutorial, you must install and configure the Amazon ECS CLI\. For more information, see [Installing the Amazon ECS CLI](ECS_CLI_installation.md)\.


+ [Amazon ECS CLI Tutorial using Fargate Launch Type](#ECS_CLI_tutorial_fargate)
+ [Amazon ECS CLI Tutorial using EC2 Launch Type](#ECS_CLI_tutorial_EC2)

## Amazon ECS CLI Tutorial using Fargate Launch Type<a name="ECS_CLI_tutorial_fargate"></a>

This tutorial shows you how to set up a cluster and deploy a task using the Fargate launch type\. It is expected that you have set up an AWS account, and have installed and configured the AWS CLI\. For more information on how to configure your environment, see [AWS Command Line Interface](http://docs.aws.amazon.com/cli/latest/userguide/cli-environment.html)\.

### Step 1: Prerequisites<a name="ECS_CLI_tutorial_fargate_prereqs"></a>

Amazon ECS needs permissions so that your Fargate task will be able to store logs in CloudWatch\. Create a role policy ahead of time so it can be referenced later\.

**Create the Task Execution Role**

1. Create a file named `execution-assume-role.json` with the following contents:

   ```
   {
     "Version": "2012-10-17",
     "Statement": [
       {
         "Sid": "",
         "Effect": "Allow",
         "Principal": {
           "Service": "ecs-tasks.amazonaws.com"
         },
         "Action": "sts:AssumeRole"
       }
     ]
   }
   ```

1. Using the AWS CLI, create the task execution role:

   ```
   aws iam --region us-east-1 create-role --role-name ecsExecutionRole --assume-role-policy-document file://execution-assume-role.json
   ```

1. Using the AWS CLI attach the task execution role policy:

   ```
   aws iam --region us-east-1 attach-role-policy --role-name ecsExecutionRole --policy-arn arn:aws:iam::aws:policy/service-role/AmazonECSTaskExecutionRolePolicy
   ```

### Step 2: Configure the ECS CLI<a name="ECS_CLI_tutorial_fargate_configure"></a>

The ECS CLI requires credentials in order to make API requests on your behalf\. It can pull credentials from environment variables, an AWS profile, or an Amazon ECS profile\. For more information see [Configuring the Amazon ECS CLI](ECS_CLI_Configuration.md)\.

**Create an ECS CLI Configuration**

1. Create a cluster configuration:

   ```
   ecs-cli configure --cluster tutorial --region us-east-1 --default-launch-type FARGATE --config-name tutorial
   ```

1. Create a profile using your access key and secret key:

   ```
   ecs-cli configure profile --access-key AWS_ACCESS_KEY_ID --secret-key AWS_SECRET_ACCESS_KEY --profile-name  tutorial
   ```
**Note**  
If this is the first time you are configuring the ECS CLI these configurations will be marked as default\. If this is not your first time configuring the ECS CLI, see [ecs\-cli configure default](cmd-ecs-cli-configure-default.md) and [ecs\-cli configure profile default](cmd-ecs-cli-configure-profile-default.md) to set this as the default configuration and profile\.

### Step 3: Create a Cluster and Security Group<a name="ECS_CLI_tutorial_fargate_cluster"></a>

Create an Amazon ECS cluster with the ecs\-cli up command\. Since you specified Fargate as your default launch type in the cluster configuration, this command will create an empty cluster and a VPC configured with two public subnets\.

```
ecs-cli up
```

**Note**  
This command may take a few minutes to complete as your resources are created\. Take note of the VPC and subnet IDs that are created as they will be used later\.

Using the AWS CLI create a security group using the VPC ID from the previous output:

```
aws ec2 create-security-group --group-name "my-sg" --description "My security group" --vpc-id "VPC_ID"
```

Using AWS CLI, add a security group rule to allow inbound access on port 80:

```
aws ec2 authorize-security-group-ingress --group-id "security_group_id" --protocol tcp --port 80 --cidr 0.0.0.0/0
```

### Step 4: Create a Compose File<a name="ECS_CLI_tutorial_fargate_compose_create"></a>

For this step, create a simple Docker compose file that creates a WordPress application\. At this time, the Amazon ECS CLI supports [Docker compose file syntax](https://docs.docker.com/compose/compose-file/#versioning) versions 1 and 2\.

Here is the compose file, which you can call `docker-compose.yml`\. The `wordpress` container exposes port 80 for inbound traffic to the web server\. It also configures container logs to go to the CloudWatch log group created earlier\. This is the recommended best practice for Fargate tasks\.

```
version: '2'
services:
  wordpress:
    image: wordpress
    ports:
      - "80:80"
    logging:
      driver: awslogs
      options: 
        awslogs-group: tutorial
        awslogs-region: us-east-1
        awslogs-stream-prefix: wordpress
```

In addition to the Docker compose information, there are some Amazon ECS specific parameters you need to specify for the service\. Using the VPC, subnet, and security group IDs from the previous step, create a file named `ecs-params.yml` with the following content:

```
version: 1
task_definition:
  task_execution_role: ecsExecutionRole
  ecs_network_mode: awsvpc
  task_size:
    mem_limit: 0.5GB
    cpu_limit: 256
run_params:
  network_configuration:
    awsvpc_configuration:
      subnets:
        - "subnet ID 1"
        - "subnet ID 2"
      security_groups:
        - "security group ID"
      assign_public_ip: ENABLED
```

**Note**  
The `assign_public_ip` and `task_size` parameters are only valid for a Fargate task\. This task definition will fail if the launch type is changed to EC2\.

### Step 5: Deploy the Compose File to a Cluster<a name="ECS_CLI_tutorial_fargate_compose_deploy"></a>

After you create the compose file, you can deploy it to your cluster with ecs\-cli compose service up\. By default, the command looks for files called `docker-compose.yml` and `ecs-params.yml` in the current directory; you can specify a different docker compose file with the `--file` option, and a different ECS Params file with the `--ecs-params` option\. By default, the resources created by this command have the current directory in their titles, but you can override that with the `--project-name` option\. The `--create-log-groups` option will create the CloudWatch log groups for the container logs\.

```
ecs-cli compose --project-name tutorial service up --create-log-groups
```

### Step 6: View the Running Containers on a Cluster<a name="ECS_CLI_tutorial_fargate_view"></a>

After you deploy the compose file, you can view the containers that are running in the service with ecs\-cli compose service ps\.

```
ecs-cli compose --project-name tutorial service ps
```

Output:

```
WARN[0000] Skipping unsupported YAML option...           option name=networks
WARN[0000] Skipping unsupported YAML option for service...  option name=networks service name=wordpress
Name                                            State    Ports TaskDefinition
a06a6642-12c5-4006-b1d1-033994580605/wordpress  RUNNING  54.146.193.73:80->80/tcp  tutorial:9
```

In the above example, you can see the `wordpress` container from your compose file, and also the IP address and port of the web server\. If you point your web browser at that address, you should see the WordPress installation wizard\. Also in the output is the `task-id` of the container\. Copy the task ID; you will use it in the next step\.

### Step 7: View the Container Logs<a name="ECS_CLI_tutorial_fargate_view_running"></a>

View the logs for the task:

```
ecs-cli logs --task-id a06a6642-12c5-4006-b1d1-033994580605 --follow
```

**Note**  
The `--follow` option tells the ECS CLI to continously poll for logs\.

### Step 8: Scale the Tasks on the Cluster<a name="ECS_CLI_tutorial_fargate_scale"></a>

You can scale up your task count to increase the number of instances of your application with ecs\-cli compose service scale\. In this example, the running count of the application is increased to two\.

```
ecs-cli compose --project-name tutorial service scale 2
```

Now you should see two more containers in your cluster:

```
ecs-cli compose --project-name tutorial service ps
```

Output:

```
WARN[0000] Skipping unsupported YAML option...           option name=networks
WARN[0000] Skipping unsupported YAML option for service...  option name=networks service name=wordpress
Name                                            State    Ports        TaskDefinition
880f09ed-613d-44bf-99bb-42ca44f82904/wordpress  RUNNING  34.224.60.24:80->80/tcp   tutorial:9
a06a6642-12c5-4006-b1d1-033994580/wordpress  RUNNING  54.146.193.73:80->80/tcp  tutorial:9
```

### Step 9: Clean Up<a name="ECS_CLI_tutorial_fargate_cleanup"></a>

When you are done with this tutorial, you should clean up your resources so they do not incur any more charges\. First, delete the service so that it stops the existing containers and does not try to run any more tasks\.

```
ecs-cli compose --project-name tutorial service down
```

Now, take down your cluster, which cleans up the resources that you created earlier with ecs\-cli up\.

```
ecs-cli down --force
```

## Amazon ECS CLI Tutorial using EC2 Launch Type<a name="ECS_CLI_tutorial_EC2"></a>

### Step 1: Configure the ECS CLI<a name="ECS_CLI_tutorial_configure"></a>

The ECS CLI requires credentials in order to make API requests on your behalf\. It can pull credentials from environment variables, an AWS profile, or an Amazon ECS profile\. For more information see [Configuring the Amazon ECS CLI](ECS_CLI_Configuration.md)\.

**Create an ECS CLI Configuration**

1. Create a cluster configuration:

   ```
   ecs-cli configure --cluster ec2-tutorial --region us-east-1 --default-launch-type EC2 --config-name ec2-tutorial
   ```

1. Create a profile using your access key and secret key:

   ```
   ecs-cli configure profile --access-key AWS_ACCESS_KEY_ID --secret-key AWS_SECRET_ACCESS_KEY --profile-name  ec2-tutorial
   ```
**Note**  
If this is the first time you are configuring the ECS CLI these configurations will be marked as default\. If this is not your first time configuring the ECS CLI, see [ecs\-cli configure default](cmd-ecs-cli-configure-default.md) and [ecs\-cli configure profile default](cmd-ecs-cli-configure-profile-default.md) to set this as the default configuration and profile\.

### Step 2: Create Your Cluster<a name="ECS_CLI_tutorial_cluster_create"></a>

The first action you should take is to create a cluster of Amazon ECS container instances that you can launch your containers on with the ecs\-cli up command\. There are many options that you can choose to configure your cluster with this command, but most of them are optional\. In this example, you create a simple cluster of two `t2.medium` container instances that use the *id\_rsa* key pair for SSH access \(substitute your own key pair here\)\.

By default, the security group created for your container instances opens port 80 for inbound traffic\. You can use the `--port` option to specify a different port to open, or if you have more complicated security group requirements, you can specify an existing security group to use with the `--security-group` option\.

```
ecs-cli up --keypair id_rsa --capability-iam --size 2 --instance-type t2.medium
```

This command may take a few minutes to complete as your resources are created\. Now that you have a cluster, you can create a Docker compose file and deploy it\.

### Step 3: Create a Compose File<a name="ECS_CLI_tutorial_compose_create"></a>

For this step, create a simple Docker compose file that creates a WordPress application consisting of a web server and a MySQL database\. At this time, the Amazon ECS CLI supports [Docker compose file syntax](https://docs.docker.com/compose/compose-file/#versioning) versions 1 and 2\.

The following parameters are supported in compose files for the Amazon ECS CLI: 

+ `cap_add` \(Not valid for tasks using the Fargate launch type\)

+ `cap_drop` \(Not valid for tasks using the Fargate launch type\)

+ `command`

+ `cpu_shares`

+ `dns`

+ `dns_search`

+ `entrypoint`

+ `environment`: If an environment variable value is not specified in the compose file, but it exists in the shell environment, the shell environment variable value is passed to the task definition that is created for any associated tasks or services\.
**Important**  
We do not recommend using plaintext environment variables for sensitive information, such as credential data\.

+ `env_file`
**Important**  
We do not recommend using plaintext environment variables for sensitive information, such as credential data\.

+ `extra_hosts`

+ `hostname`

+ `image`

+ `labels`

+ `links` \(Not valid for tasks using the Fargate launch type\)

+ `log_driver` \(Compose file version 1 only\)

+ `log_opt` \(Compose file version 1 only\)

+ `logging` \(Compose file version 2 only\)

  + `driver`

  + `options`

+ `mem_limit` \(in bytes\)

+ `mem_reservation` \(in bytes\)

+ `ports`

+ `privileged` \(Not valid for tasks using the Fargate launch type\)

+ `read_only`

+ `security_opt`

+ `ulimits`

+ `user`

+ `volumes`

+ `volumes_from`

+ `working_dir`

**Important**  
The `build` directive is not supported at this time\.

For more information about Docker compose file syntax, see the [Compose file reference](https://docs.docker.com/compose/compose-file/#/compose-file-reference) in the Docker documentation\. 

Here is the compose file, which you can call `hello-world.yml`\. Each container has 100 CPU units and 500 MiB of memory\. The `wordpress` container exposes port 80 to the container instance for inbound traffic to the web server\. A logging configuration for the containers is also defined\.

```
version: '2'
services:
  wordpress:
    image: wordpress
    cpu_shares: 100
    mem_limit: 524288000
    ports:
      - "80:80"
    links:
      - mysql
    logging:
      driver: awslogs
      options: 
        awslogs-group: tutorial-wordpress
        awslogs-region: us-east-1
        awslogs-stream-prefix: wordpress
  mysql:
    image: mysql
    cpu_shares: 100
    mem_limit: 524288000
    environment:
      MYSQL_ROOT_PASSWORD: password
    logging:
      driver: awslogs
      options: 
        awslogs-group: tutorial-mysql
        awslogs-region: us-east-1
        awslogs-stream-prefix: mysql
```

### Step 4: Deploy the Compose File to a Cluster<a name="ECS_CLI_tutorial_compose_deploy"></a>

After you create the compose file, you can deploy it to your cluster with the ecs\-cli compose up command\. By default, the command looks for a file called `docker-compose.yml` in the current directory, but you can specify a different file with the `--file` option\. By default, the resources created by this command have the current directory in the title, but you can override that with the `--project-name project_name` option\. The `--create-log-groups` option will create the CloudWatch log groups for the container logs\.

```
ecs-cli compose --file hello-world.yml up --create-log-groups
```

### Step 5: View the Running Containers on a Cluster<a name="ECS_CLI_tutorial_ps"></a>

After you deploy the compose file, you can view the containers that are running on your cluster with the ecs\-cli ps command\.

```
ecs-cli ps
```

```
Name                                            State    Ports                     TaskDefinition
 340488e0-a307-4322-b41c-99f1b70e97f9/wordpress  RUNNING  52.89.204.137:80->80/tcp  ecscompose-docker-compose:2
 340488e0-a307-4322-b41c-99f1b70e97f9/mysql      RUNNING                            ecscompose-docker-compose:2
```

In the above example, you can see the `wordpress` and `mysql` containers from your compose file, and also the IP address and port of the web server\. If you point a web browser to that address, you should see the WordPress installation wizard\.

### Step 6: Scale the Tasks on a Cluster<a name="ECS_CLI_tutorial_compose_scale"></a>

You can scale your task count up so you could have more instances of your application with the ecs\-cli compose scale command\. In this example, you can increase the count of your application to two\.

```
ecs-cli compose --file hello-world.yml scale 2
```

Now you should see two more containers in your cluster\.

```
ecs-cli ps
```

Output:

```
Name                                            State    Ports                     TaskDefinition
 340488e0-a307-4322-b41c-99f1b70e97f9/wordpress  RUNNING  52.89.204.137:80->80/tcp  ecscompose-docker-compose:2
 340488e0-a307-4322-b41c-99f1b70e97f9/mysql      RUNNING                            ecscompose-docker-compose:2
 f80d82d5-3724-4f2f-86b1-5ee5891ce995/mysql      RUNNING                            ecscompose-docker-compose:2
 f80d82d5-3724-4f2f-86b1-5ee5891ce995/wordpress  RUNNING  52.89.205.89:80->80/tcp   ecscompose-docker-compose:2
```

### Step 7: Create an ECS Service from a Compose File<a name="ECS_CLI_tutorial_compose_service"></a>

Now that you know that your containers work properly, you can make sure that they are replaced if they fail or stop\. You can do this by creating a service from your compose file with the ecs\-cli compose service up command\. This command creates a task definition from the latest compose file \(if it does not already exist\) and creates an ECS service with it, with a desired count of 1\.

Before starting your service, stop the containers from your compose file with the ecs\-cli compose down command so that you have an empty cluster to work with\.

```
ecs-cli compose --file hello-world.yml down
```

Now you can create your service\.

```
ecs-cli compose --file hello-world.yml service up
```

Output:

```
INFO[0000] Using ECS task definition                     TaskDefinition=ecscompose-docker-compose:2
 INFO[0000] Created an ECS Service                        serviceName=ecscompose-service-docker-compose taskDefinition=ecscompose-docker-compose:2
 INFO[0000] Updated ECS service successfully              desiredCount=1 serviceName=ecscompose-service-docker-compose
 INFO[0000] Describe ECS Service status                   desiredCount=1 runningCount=0 serviceName=ecscompose-service-docker-compose
 INFO[0015] ECS Service has reached a stable state        desiredCount=1 runningCount=1 serviceName=ecscompose-service-docker-compose
```

### Step 8: Clean Up<a name="ECS_CLI_tutorial_cleaning_up"></a>

When you are done with this tutorial, you should clean up your resources so they do not incur any more charges\. First, delete the service so that it stops the existing containers and does not try to run any more tasks\.

```
ecs-cli compose --file hello-world.yml service rm
```

Output:

```
INFO[0000] Updated ECS service successfully              desiredCount=0 serviceName=ecscompose-service-docker-compose
 INFO[0000] Describe ECS Service status                   desiredCount=0 runningCount=1 serviceName=ecscompose-service-docker-compose
 INFO[0015] ECS Service has reached a stable state        desiredCount=0 runningCount=0 serviceName=ecscompose-service-docker-compose
 INFO[0015] Deleted ECS service                           service=ecscompose-service-docker-compose
 INFO[0015] ECS Service has reached a stable state        desiredCount=0 runningCount=0 serviceName=ecscompose-service-docker-compose
```

Now, take down your cluster, which cleans up the resources that you created earlier with ecs\-cli up\.

```
ecs-cli down --force
```

Output:

```
INFO[0000] Waiting for your cluster resources to be deleted
 INFO[0000] Cloudformation stack status                   stackStatus=DELETE_IN_PROGRESS
 INFO[0061] Cloudformation stack status                   stackStatus=DELETE_IN_PROGRESS
 INFO[0121] Deleted cluster                               cluster=ecs-cli-demo
```