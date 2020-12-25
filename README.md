# amazon-ecs-deployment-circuit-breaker-sample
"Announcing Amazon ECS deployment circuit breaker" example


## Setting

```
pip3 install --upgrade aws-cdk.core
pip3 install --upgrade aws-cdk.aws_ecs
pip3 install --upgrade aws-cdk.aws_ec2
```

```
$ git clone git@github.com:daisukeshimizu/amazon-ecs-deployment-circuit-breaker-sample.git
```


```
export region=ap-northeast-1
export AWS_PROFILE=dev
export account_id=$(aws sts get-caller-identity --output text --query Account)\
export ECR_REPO=$(aws cloudformation describe-stacks --stack-name circuit-breaker-demo --query 'Stacks[].Outputs[?ExportName == `EcrRepoUri`].OutputValue' --output text)\
export ECR_IMAGE="${ECR_REPO}:working"\
export EXECUTIONROLEARN=$(aws cloudformation describe-stacks --stack-name circuit-breaker-demo --query 'Stacks[].Outputs[?ExportName == `IAMRoleArn`].OutputValue' --output text)\
export SUBNETS=$(aws cloudformation describe-stacks --stack-name circuit-breaker-demo --query 'Stacks[].Outputs[?ExportName == `PublicSubnets`].OutputValue' --output text)\
export SECGRP=$(aws cloudformation describe-stacks --stack-name circuit-breaker-demo --query 'Stacks[].Outputs[?ExportName == `SecurityGroupId`].OutputValue' --output text)
aws ecr get-login-password \\
  --region $region \\
  | docker login \\
    --username AWS \\
    --password-stdin $account_id.dkr.ecr.$region.amazonaws.com
 ```
 
 ```
 docker build -t ${ECR_IMAGE} . && docker push ${ECR_IMAGE}
 ```
 
 ```
 cdk deploy --require-approval never --app "python3 app.py
 ```
 
 ```
 echo '{\
  "containerDefinitions": [\
    {\
      "name": "cb-demo",\
      "image": "$ECR_IMAGE",\
      "essential": true,\
      "portMappings": [\
        {\
          "containerPort": 5000,\
          "hostPort": 5000,\
          "protocol": "tcp"\
        }\
      ]\
    }\
  ],\
  "executionRoleArn": "$EXECUTIONROLEARN",\
  "family": "circuit-breaker",\
  "requiresCompatibilities": [\
    "FARGATE"\
  ],\
  "networkMode": "awsvpc",\
  "cpu": "256",\
  "memory": "512"\
}' | envsubst > task_definition.json
```

```
aws ecs register-task-definition --cli-input-json file://task_definition.json
```

```
aws ecs create-service \\
  --service-name circuit-breaker-demo \\
  --cluster CB-Demo \\
  --task-definition circuit-breaker \\
  --desired-count 5 \\
  --deployment-controller type=ECS \\
  --deployment-configuration "maximumPercent=200,minimumHealthyPercent=100,deploymentCircuitBreaker={enable=true,rollback=true}" \\
  --network-configuration "awsvpcConfiguration={subnets=[$SUBNETS],securityGroups=[$SECGRP],assignPublicIp=ENABLED}" \\
  --launch-type FARGATE \\
  --platform-version 1.4.0
```


```
aws ecs update-service --service circuit-breaker-demo --cluster CB-Demo  --desired-count 0
```

```
aws ecs delete-service --service circuit-breaker-demo --cluster CB-Demo
```
