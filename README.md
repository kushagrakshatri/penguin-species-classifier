# Penguin Species Classifier

In this project, we create an end-to-end ML system to classify Penguin species in real-time on AWS SageMaker

## Setup Instructions

Start by forking this GitHub Repository and clone it on your local computer.

Create and activate a virtual environment:

```
$ python3 -m venv .venv
$ source .venv/bin/activate
```

Once the virtual environment is active, you can update pip and install the libraries in the ```requirements.txt``` file:

```
$ python -m pip install --upgrade pip
$ pip install -r requirements.txt
```

We’ll use Jupyter Notebooks during the program. Using the following command, you can install a Jupyter kernel in the virtual environment. If you use Visual Studio Code, you can point your kernel to the virtual environment, and it will install it automatically:

```
$ python -m ipykernel install --user --name=.venv
```

Install Docker. You’ll find installation instructions on their site for your particular environment. After you install it, you can verify Docker is running using the following command:

```
$ docker ps
```

At this point you can open the project using Visual Studio Code or your favorite IDE. Make sure you point the Jupyter kernel to the virtual environment that you created before.

### Configuring AWS
If you don’t have one yet, create a new AWS account. If you don’t have one yet, create a new AWS account. We’ll need access to a minimum of 3 ```ml.m5.xlarge``` instances

You’ll need access to AWS from your local environment. Install the AWS CLI and configure it with your ```aws_access_key_id``` and ```aws_secret_access_key```.

After you finish configuring the CLI, create a new S3 bucket where we will store the data and every resource we are going to create during the program. The name of the bucket must be unique:

```
$ aws s3api create-bucket --bucket [YOUR-BUCKET-NAME]
```

Upload the dataset to the S3 bucket you just created:

```
$ aws s3 cp program/penguins.csv s3://[YOUR-BUCKET-NAME]/penguins/data/data.csv
```

### Configuring Sagemaker

If you don’t have one yet, create a SageMaker domain. The Getting Started on Amazon SageMaker Studio video will walk you through the process.

After you are done, run the following command to return the Domain Id and the User Profile Name of your SageMaker domain:

```
$ aws sagemaker list-user-profiles | grep -E '"DomainId"|"UserProfileName"' \
    | awk -F'[:,"]+' '{print $2":"$3 $4 $5}'
```

Use the DomainId and the UserProfileName from the response and replace them in the following command that we’ll return the execution role attached to the user:

```
$ aws sagemaker describe-user-profile \
    --domain-id [YOUR-DOMAIN-ID] \
    --user-profile-name [YOUR-USER-PROFILE-NAME] \
    | grep -E "ExecutionRole" | awk -F'["]' '{print $2": "$4}'
```

Create an .env file in the root directory of your repository with the following content. Make sure you replace the value of each variable with the correct value:

```
BUCKET=[YOUR-BUCKET-NAME]
DOMAIN_ID=[YOUR-DOMAIN-ID]
USER_PROFILE=[YOUR-USER-PROFILE]
ROLE=[YOUR-EXECUTION-ROLE]
```

Open the Amazon IAM service, find the Execution Role from before and edit the custom Execution Policy assigned to it. Edit the permissions of the Execution Policy and replace them with the JSON below. These permissions will give the Execution Role access to the resources we’ll use during the program:

```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "IAM0",
            "Effect": "Allow",
            "Action": [
                "iam:CreateServiceLinkedRole"
            ],
            "Resource": "*",
            "Condition": {
                "StringEquals": {
                    "iam:AWSServiceName": [
                        "autoscaling.amazonaws.com",
                        "ec2scheduled.amazonaws.com",
                        "elasticloadbalancing.amazonaws.com",
                        "spot.amazonaws.com",
                        "spotfleet.amazonaws.com",
                        "transitgateway.amazonaws.com"
                    ]
                }
            }
        },
        {
            "Sid": "IAM1",
            "Effect": "Allow",
            "Action": [
                "iam:CreateRole",
                "iam:DeleteRole",
                "iam:PassRole",
                "iam:AttachRolePolicy",
                "iam:DetachRolePolicy",
                "iam:CreatePolicy"
            ],
            "Resource": "*"
        },
        {
            "Sid": "Lambda",
            "Effect": "Allow",
            "Action": [
                "lambda:CreateFunction",
                "lambda:DeleteFunction",
                "lambda:InvokeFunctionUrl",
                "lambda:InvokeFunction",
                "lambda:UpdateFunctionCode",
                "lambda:InvokeAsync",
                "lambda:AddPermission",
                "lambda:RemovePermission"
            ],
            "Resource": "*"
        },
        {
            "Sid": "SageMaker",
            "Effect": "Allow",
            "Action": [
                "sagemaker:UpdateDomain",
                "sagemaker:UpdateUserProfile"
            ],
            "Resource": "*"
        },
        {
            "Sid": "CloudWatch",
            "Effect": "Allow",
            "Action": [
                "cloudwatch:PutMetricData",
                "cloudwatch:GetMetricData",
                "cloudwatch:DescribeAlarmsForMetric",
                "logs:CreateLogStream",
                "logs:PutLogEvents",
                "logs:CreateLogGroup",
                "logs:DescribeLogStreams"
            ],
            "Resource": "*"
        },
        {
            "Sid": "ECR",
            "Effect": "Allow",
            "Action": [
                "ecr:GetAuthorizationToken",
                "ecr:BatchCheckLayerAvailability",
                "ecr:GetDownloadUrlForLayer",
                "ecr:BatchGetImage"
            ],
            "Resource": "*"
        },
        {
            "Sid": "S3",
            "Effect": "Allow",
            "Action": [
                "s3:CreateBucket",
                "s3:ListBucket",
                "s3:GetBucketLocation",
                "s3:PutObject",
                "s3:GetObject",
                "s3:DeleteObject"
            ],
            "Resource": "arn:aws:s3:::*"
        },
        {
            "Sid": "EventBridge",
            "Effect": "Allow",
            "Action": [
                "events:PutRule",
                "events:PutTargets"
            ],
            "Resource": "*"
        }
    ]
}
```

Finally, find the Trust relationships section under the same Execution Role, edit the configuration, and replace it with the JSON below:

```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "Service": [
                    "sagemaker.amazonaws.com", 
                    "events.amazonaws.com"
                ]
            },
            "Action": "sts:AssumeRole"
        }
    ]
}
```

### Apple Silicon

If your local environment is running on Apple silicon, you need to build a TensorFlow docker image to run some of the pipeline steps on your local computer. This is because SageMaker doesn’t provide out-of-the-box TensorFlow images compatible with Apple silicon.

You can build the image running the following command:

```
$ docker build -t sagemaker-tensorflow-toolkit-local container/.
```

After building this Docker image, the notebook will automatically use it when running the pipeline in Local Mode on your Mac. There’s nothing else you need to do.

