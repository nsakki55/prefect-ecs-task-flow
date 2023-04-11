# Builtin Container + S3
use the S3 Block with the Prefect official image to manage Flow code.

## Create an S3 Block
To manage the Flow code with S3, create an S3 Block.   
When deploying to Prefect Cloud, the Flow code will be automatically uploaded to S3.

Specify the bucket name as `prefect-etl`, and create the S3 Block named `etl-s3-block` as shown below:
```python
from prefect.filesystems import S3
import os

block = S3(bucket_path="prefect-etl",
           aws_access_key_id=os.environ['AWS_ACCESS_KEY_ID'],
           aws_secret_access_key=os.environ['AWS_SECRET_ACCESS_KEY']
)
block.save("etl-s3-block", overwrite=True)
```

## Create ECS Task Block
To run the Flow as an ECS Task, create an ECS Task Block.   
If you don't specify a launch image, the official image for the Prefect and Python version of the ECS Agent is used.

Set the `EXTRA_PIP_PACKAGES` environment variable to install the dependencies required for Flow execution when the image is launched.

The network configuration is set to the same values as the ECS Service running the Agent.
```python
from prefect_aws.ecs import ECSTask

ecs = ECSTask(
    env={"EXTRA_PIP_PACKAGES": "s3fs pandas boto3"},
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
ecs.save("builtin-image-ecs-task-block", overwrite=True)
```

## Register Deployment
Create a Deployment with S3 Block for storage and ECS Task Block for infrastructure.

Specify the S3 Block and ECS Task Block created above.

```python
from my_flow import etl_flow
from prefect.deployments import Deployment
from prefect_aws.ecs import ECSTask
from prefect.filesystems import S3

s3_block = S3.load("etl-s3-block")
ecs_task_block = ECSTask.load("base-image-ecs-task-block")

deployment = Deployment.build_from_flow(
    flow=etl_flow,
    name="base-image-s3-deployment",
    work_pool_name="ecs-pool",
    storage=s3_block,
    infrastructure=ecs_task_block,
)

deployment.apply()
```