# **ECS Exec in action via the AWS CLI workflow with NGINX Container**
<br />

## **Create the environment**   

### Declare variables

``` 
$ export AWS_REGION=<xxxxxxx>
$ export VPC_ID=<vpc-xxxx>
$ export PUBLIC_SUBNET1=<subnet-xxxxx>
$ export PUBLIC_SUBNET2=<subnet-xxxxx>
$ export ECS_EXEC_BUCKET_NAME=khaja-aws-ecs-exec-s3-bucket-output-xxxxxxxxxx
```

### Create an IAM Policy Document

> We will create an IAM Policy with the name as `ecs-tasks-trust-policy.json` and with the below content. Then, we will apply it to the ECS task role and ECS task execution role. 

``` json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": [
          "ecs-tasks.amazonaws.com"
        ]
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
```

### Create required AWS Resources

1. KMS Keys, this is to encryot the ECS Exec data channel
    - Create KMS key in the region of your choice
        ``` shell
        $ KMS_KEY=$(aws kms create-key --region $AWS_REGION)
        ```
    - Fetch the KMS Key ARN
        ``` shell
        $ KMS_KEY_ARN=$(echo $KMS_KEY | jq --raw-output .KeyMetadata.Arn)
        ```
    - Create alias to your KMS Keys
        ``` shell
        $ aws kms create-alias --alias-name alias/ecs-exec-demo-kms-key --target-key-id $KMS_KEY_ARN --region $AWS_REGION
        $ echo "The KMS Key ARN is: "$KMS_KEY_ARN
        ```
    <br />
1. Create ECS Cluster with `executeCommandConfiguration` details
    ``` shell
    $ aws ecs create-cluster \
        --cluster-name ecs-exec-demo-cluster \
        --region $AWS_REGION \
        --configuration executeCommandConfiguration="{logging=OVERRIDE,\
        kmsKeyId=$KMS_KEY_ARN,\
        logConfiguration={cloudWatchLogGroupName="/aws/ecs/ecs-exec-demo",\
        s3BucketName=$ECS_EXEC_BUCKET_NAME,\
        s3KeyPrefix=exec-output}}"
    ```
    <br />
1. CloudWatch log group with two streams 
    - `(a)` For the container stdout and 
    - `(b)` For logging output of the new ECS Exec feature
    ``` shell
    $ aws logs create-log-group --log-group-name /aws/ecs/ecs-exec-demo --region $AWS_REGION
    ```
    <br />
1. 	*Optional*: S3 bucket for the logging output of the new ECS Exec feature
    ``` shell
    $ aws s3api create-bucket --bucket $ECS_EXEC_BUCKET_NAME --region $AWS_REGION --create-bucket-configuration LocationConstraint=$AWS_REGION 
    ```
    <br />
1. Create Security group to allow traffic on port 80 to hit the nginx container
    ``` shell
    $ ECS_EXEC_DEMO_SG=$(aws ec2 create-security-group --group-name ecs-exec-demo-SG --description "ECS exec demo SG" --vpc-id $VPC_ID --region $AWS_REGION) 
    $ ECS_EXEC_DEMO_SG_ID=$(echo $ECS_EXEC_DEMO_SG | jq --raw-output .GroupId)
    $ aws ec2 authorize-security-group-ingress --group-id $ECS_EXEC_DEMO_SG_ID --protocol tcp --port 80 --cidr 0.0.0.0/0 --region $AWS_REGION
    ```
    <br />
1. Create two IAM roles that we will use to define the ECS task role and the ECS task execution role
    ``` shell
    $ aws iam create-role --role-name ecs-exec-demo-task-execution-role --assume-role-policy-document file://ecs-tasks-trust-policy.json --region $AWS_REGION
    $ aws iam create-role --role-name ecs-exec-demo-task-role --assume-role-policy-document file://ecs-tasks-trust-policy.json --region $AWS_REGION
    ```
    <br />
1. Create a policy with the name `ecs-exec-demo-task-role-policy.json`. Some of the variables should be updated in the below policy. Please update them accordingly. Attach the default policy `AmazonECSTaskExecutionRolePolicy` to `ecs-exec-demo-task-execution-role` and the the above policy json to the `ecs-exec-demo-task-role` with the json policy name as `ecs-exec-demo-task-role-policy.json` 
    - For the ECS task role, we have added a custom policy so that it allows the container to open the secure channel session via SSM and log the ECS Exec output to both CloudWatch and S3 (to the LogStream and to the bucket created above). Now, update the following items in the
      - **<AWS_REGION>**
      - **<ACCOUNT_ID>**
      - **<ECS_EXEC_BUCKET_NAME>** (whose value is in the ECS_EXEC_BUCKET_NAME variable)
      - **<KMS_KEY_ARN>** created above (whose value is in the KMS_KEY_ARN variable)
        ``` json
        {
        "Version": "2012-10-17",
        "Statement": [
            {
                "Effect": "Allow",
                "Action": [
                    "ssmmessages:CreateControlChannel",
                    "ssmmessages:CreateDataChannel",
                    "ssmmessages:OpenControlChannel",
                    "ssmmessages:OpenDataChannel"
                ],
                "Resource": "*"
            },
            {
                "Effect": "Allow",
                "Action": [
                    "logs:DescribeLogGroups"
                ],
                "Resource": "*"
            },
            {
                "Effect": "Allow",
                "Action": [
                    "logs:CreateLogStream",
                    "logs:DescribeLogStreams",
                    "logs:PutLogEvents"
                ],
                "Resource": "arn:aws:logs:<AWS_REGION>:<ACCOUNT_ID>:log-group:/aws/ecs/ecs-exec-demo:*"
            },
            {
                "Effect": "Allow",
                "Action": [
                    "s3:PutObject"
                ],
                "Resource": "arn:aws:s3:::<ECS_EXEC_BUCKET_NAME>/*"
            },
            {
                "Effect": "Allow",
                "Action": [
                    "s3:GetEncryptionConfiguration"
                ],
                "Resource": "arn:aws:s3:::<ECS_EXEC_BUCKET_NAME>"
            },
            {
                "Effect": "Allow",
                "Action": [
                    "kms:Decrypt"
                ],
                "Resource": "<KMS_KEY_ARN>"
            }
        ]
        }
        ```
    <br />
1. Execute the following AWS CLI commands to bind the policies to the IAM roles.
    ``` shell
    $ aws iam attach-role-policy \
    --role-name ecs-exec-demo-task-execution-role \
    --policy-arn "arn:aws:iam::aws:policy/service-role/AmazonECSTaskExecutionRolePolicy"
    $ aws iam put-role-policy \
    --role-name ecs-exec-demo-task-role \
    --policy-name ecs-exec-demo-task-role-policy \
    --policy-document file://ecs-exec-demo-task-role-policy.json
    ```
    <br />
1. Create ECS Task definintion with the NGINX container and give the name as `ecs-exec-demo.json`
    ``` shell
    {
        "family": "ecs-exec-demo",
        "networkMode": "awsvpc",
        "executionRoleArn": "arn:aws:iam::<ACCOUNT_ID>:role/ecs-exec-demo-task-execution-role",
        "taskRoleArn": "arn:aws:iam::<ACCOUNT_ID>:role/ecs-exec-demo-task-role",
        "containerDefinitions": [
            {"name": "nginx",
                "image": "nginx",
                "linuxParameters": {
                    "initProcessEnabled": true
                },            
                "logConfiguration": {
                    "logDriver": "awslogs",
                        "options": {
                           "awslogs-group": "/aws/ecs/ecs-exec-demo",
                           "awslogs-region": "<AWS_REGION>",
                           "awslogs-stream-prefix": "container-stdout"
                        }
                }
            }
        ],
        "requiresCompatibilities": [
            "FARGATE"
        ],
        "cpu": "256",
        "memory": "512"
    }
    ```
    <br />
1. Register the task that we created above
    ``` shell
    $ aws ecs register-task-definition \
    --cli-input-json file://ecs-exec-demo.json \
    --region $AWS_REGION
    ```
    <br />
1. Run the task inside the cluster we created earlier with all required parameters
    ``` shell
    $ aws ecs run-task \
        --cluster ecs-exec-demo-cluster  \
        --task-definition ecs-exec-demo \
        --network-configuration awsvpcConfiguration="{subnets=[$PUBLIC_SUBNET1, $PUBLIC_SUBNET2],securityGroups=[$ECS_EXEC_DEMO_SG_ID],assignPublicIp=ENABLED}" \
        --enable-execute-command \
        --launch-type FARGATE \
        --tags key=environment,value=production \
        --platform-version '1.4.0' \
        --region $AWS_REGION
    ```
    <br />
1. Find the `taskArn` in which the last part is the `task-id` that we need to SSH with `aws ecs execute-command`. You can find it when you run the above command. You can find the example below, where the one between the two *'s is the `task-id`
    ``` json
    "taskArn": "arn:aws:ecs:AWS_REGION:ACCOUNT_ID:task/ecs-exec-demo-cluster/*ef6260ed8aab49cf926667ab0c52c313*"
    ```
    <br />
1. Describe the task to ensure that `ExecuteCommandAgent` is in `RUNNING` status and `enableExecuteCommand` is `true`. Fetch the **<TASK-ID>** from above step
    ``` shell
    $ aws ecs describe-tasks \
    --cluster ecs-exec-demo-cluster \
    --region $AWS_REGION \
    --tasks <TASK-ID>
    ```
  <br />
1. Let's SSH into the container's shell now with the below command. 
    ``` shell
        $ aws ecs execute-command  \
          --region $AWS_REGION \
          --cluster ecs-exec-demo-cluster \
          --task ef6260ed8aab49cf926667ab0c52c313 \
          --container nginx \
          --command "/bin/bash" \
          --interactive
  
        The Session Manager plugin was installed successfully. Use the AWS CLI to start a session.
        Starting session with SessionId: ecs-execute-command-0122b68a67f39258e
        This session is encrypted using AWS KMS.
        root@ip-172-31-32-237:/# ls
        bin   dev docker-entrypoint.sh  home  lib64 media  opt   root  sbin  sys  usr
        boot  docker-entrypoint.d  etc lib   managed-agents  mnt    proc  run   srv   tmp  var
    ```
  <br />
1. You can `curl` on the public ip of your container. To know the public ip of your container from `step 14` output you should find the `<eni-id>` which you can substitue in the below command. The below command will show the `public ip` of your container to which you can curl
    ``` shell
    $ aws ec2 describe-network-interfaces --network-interface-ids <eni-id>
    ```
    ``` shell
    curl http://13.122.128.136/
    ```
    <br />
1. Optional: You can also run one off command instead of SSH into the container. The below will output for the `ls` command
    ``` shell
    $ aws ecs execute-command  \
        --region $AWS_REGION \
        --cluster ecs-exec-demo-cluster \
        --task <task-id> \
        --container nginx \
        --command "ls" \
        --interactive
    ```
    <br />
1. Check your `cloudtrail`, `cloudshell` and `s3` buckets for logs. What you will find interesting is that the commands you ran inside containers shows up only in the cloudwatch and s3 but not in cloudtrail. For auditing purposes, you need to customize the experience. 

<br />

## **Tear down the environment**   
    
1. It's important to tear down/ cleanup all the resources created so that we won't be charged after this session. Run the below commands to ensure all the resources we have created above are deleted. 

    ``` shell
    $ aws ecs stop-task --cluster ecs-exec-demo-cluster --region $AWS_REGION --task <task-id> 
    $ aws ecs delete-cluster --cluster ecs-exec-demo-cluster --region $AWS_REGION
    $ aws logs delete-log-group --log-group-name /aws/ecs/ecs-exec-demo --region $AWS_REGION
    # Be careful running this command. This will delete the bucket we previously created
    $ aws s3 rm s3://$ECS_EXEC_BUCKET_NAME --recursive
    $ aws s3api delete-bucket --bucket $ECS_EXEC_BUCKET_NAME
    $ aws iam detach-role-policy --role-name ecs-exec-demo-task-execution-role --policy-arn "arn:aws:iam::aws:policy/service-role/AmazonECSTaskExecutionRolePolicy"
    $ aws iam delete-role --role-name ecs-exec-demo-task-execution-role
    $ aws iam delete-role-policy --role-name ecs-exec-demo-task-role --policy-name ecs-exec-demo-task-role-policy
    $ aws iam delete-role --role-name ecs-exec-demo-task-role 
    $ aws kms schedule-key-deletion --key-id $KMS_KEY_ARN --region $AWS_REGION
    $ aws ec2 delete-security-group --group-id $ECS_EXEC_DEMO_SG_ID --region $AWS_REGION
    $ aws kms delete-alias --alias-name alias/ecs-exec-demo-kms-key
    ```

> Note: It's always recommended to double check the resources, if they are properly deleted. If not, do delete them manually. If you find any issues in deleting any resources, raise an AWS ticket immediately and get help from the AWS Support team. 
    
    
    
    
    
    
    
    
    
    
    
    
    
    
    
    
    
    
