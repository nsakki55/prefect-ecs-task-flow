# Custom Container
How to create and use a custom container image that includes dependencies and Flow code to be executed in an ECS Task.

## Create Custom Image
Create a Dockerfile that includes the dependencies and the Flow file, which should be place d in the `opt/prefect/flows` directory

```bash
FROM prefecthq/prefect:2.8.7-python3.8

COPY requirements.txt requirements.txt
RUN pip install -r requirements.txt

COPY my_flow.py /opt/prefect/flows/my_flow.py
```

Build the image and push it to ECR with the name `prefect-custom-container`
```bash
$ docker build -t prefect-custom-container .
$ docker tag prefect-custom-container:latest *****.dkr.ecr.ap-northeast-1.amazonaws.com/prefect-custom-container:latest
$ docker push *****.dkr.ecr.ap-northeast-1.amazonaws.com/prefect-custom-container:latest
```

## Create ECS Task Block
Create an ECS Task Block that specifies the custom image you just pushed to ECR.   
You'll need to provide the ECR image URI.

```python
from prefect_aws.ecs import ECSTask

ecs = ECSTask(
    image="*****.dkr.ecr.ap-northeast-1.amazonaws.com/prefect-custom-container:latest",
    cluster="arn:aws:ecs:ap-northeast-1:*****:cluster/prefect-ecs",
    cpu="256",
    memory="512",
    stream_output=True,
    configure_cloudwatch_logs=True,
    execution_role_arn="arn:aws:iam::*****:role/prefectEcsTaskExecutionRole",
    task_role_arn="arn:aws:iam::*****:role/prefectEcsTaskRole",
    vpc_id="vpc-*****",
    task_customizations=[
  {
    "op": "replace",
    "path": "/networkConfiguration/awsvpcConfiguration/assignPublicIp",
    "value": "DISABLED"
  },
  {
    "op": "add",
    "path": "/networkConfiguration/awsvpcConfiguration/subnets",
    "value": [
      "subnet-*****"
    ]
  }
]
)
ecs.save("custom-container-ecs-task-block", overwrite=True)


```

## Register Deployment
Create a Deployment that uses the ECS Task Block you just created.   
You'll need to import your Flow and provide a name for the Deployment:

```python
from my_flow import etl_flow
from prefect.deployments import Deployment
from prefect_aws.ecs import ECSTask

ecs_task_block = ECSTask.load("custom-container-ecs-task-block")

deployment = Deployment.build_from_flow(
    flow=etl_flow,
    name="custom-container-deployment",
    work_pool_name="ecs-pool",
    infrastructure=ecs_task_block,
)

deployment.apply()
```