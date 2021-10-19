

# Deploy a Locust load generator in ECS using CDK Python 

This lab is provided as part of **[AWS Innovate - Modern Applications Edition](https://aws.amazon.com/events/aws-innovate/modern-apps/)**, click [here](https://google.com) to explore the full list of hands-on labs.

ℹ️ You will run this lab in your own AWS account. Please follow directions at the end of the lab to remove resources to minimize costs.

This lab walks you through creating a CDK project in Python that will implement 
an ECS Service running [Locust.io](https://locust.io/). Using CDK constructs 
we'll create a customised Locust container image and all supporting service configuration, including: VPC, 
ECS cluster, ECS Service, Application Load Balancer, and a CloudWatch dashboard.

At a high level, the architecture will look like this:
![Architecure](/images/Architecture.png)

## Step 0: Getting Started

In order to run this lab, you'll need a development environment with Python3 and
CDK installed, and your AWS account bootstrapped for CDK. If you already have this,
please skip to Step 1.

Cloud9 comes with Python3 and CDK installed by default, so we will use Cloud9 for this lab.
Open the Cloud9 console in the region in which you will complete this lab
and create a new Environment, give it a name ``CDK Locust Lab`` and hit "Next Step"

![Cloud9 Name](/images/Cloud9_Create.png)

On the next page, select your instance size as Other Instance type > t3.micro. Then hit "Next Step"

![Cloud9 Settings](/images/Cloud9_Settings.png)

Then review your settings and hit "Create Environment"

Once your Cloud9 development environment is created, it will open the IDE. 
Whenever you need to get back into your IDE, just go to the Cloud9 console, and
click "Open IDE" on your Environment. 


## Step 1: Clone lab resources and Initialise a CDK project

1. Clone the lab resources into a local directory.
```
git clone https://github.com/roshansthomas/python-cdk-locust
```
This will create a local copy of this repository that includes this README.md,  
the Dockerfile and locust test file.


2. Enter the python-cdk-locust directory and create a new directory for your work 
```
cd python-cdk-locust
mkdir lab
cd lab
```
3. Initialise your CDK project. The initialisation must be run in an empty directory. In this case, ``~/environment/python-cdk-locust/lab``.
```
cdk init --language python
```
The initialisation creates the base directory structure for your project. 
Some important files:
 * app.py - The entry point for you app, it defines your environment  and the 
stack(s) that will be created
 * requirements.txt - Library dependencies for your Python code, in this case 
the CDK libraries that we will use
 * lab/lab.py - The file that defines the CDK stack

You can learn more about CDK from this 
[Blog post](https://aws.amazon.com/blogs/developer/getting-started-with-the-aws-cloud-development-kit-and-python/)
or from the [CDK Developer Guide](https://docs.aws.amazon.com/cdk/latest/guide/home.html)

 
## Step 2: Set the region that our stack will deploy into
Open the app.py file in Cloud9, and add an environment paramater to the stack instantiation, to set the region as ap-southeast-2. We will assume that you are deploying within the same account. However if the you are deploying to a different account, then set this with the "account" property.

```
LabStack(app, "LabStack",
    env={'region': 'ap-southeast-2'}
)
```
**Remember: Save your files after each step!**

## Step 3: Define your python dependencies
Open the file named requirements.txt in your lab directory, replace the contents with 
the following, and save it. This file defines the Python libraries that our 
lab will use. 
```
aws-cdk.core
aws-cdk.aws_ec2
aws-cdk.aws_ecs
aws-cdk.aws_ecs_patterns
aws-cdk.aws_cloudwatch
```

# Let's Build!

In CDK there are 3 levels of construct:
1. CFn Resources - These are constructs which are created directly from 
CloudFormation and work in the same way as the CloudFormation resource 
they're based upon, requiring you to explicitly configure all resource 
properties, which requires a complete understanding of the details of the 
underlying resource model.
2. AWS Constructs - The next level of constructs also represent AWS resources, 
but with a higher-level, intent-based API. AWS Constructs offer convenient 
defaults and reduce the need to know all the details about the AWS resources they represent.
3. Patterns - These constructs are designed to help you complete common tasks in 
AWS, often involving multiple kinds of resources.

You can find more information on CDK constructs in the CDK Developer Guide - 
[Constructs](https://docs.aws.amazon.com/cdk/latest/guide/constructs.html)

In this lab we'll be using a Pattern from the 
[aws_ecs_patterns](https://docs.aws.amazon.com/cdk/api/latest/python/aws_cdk.aws_ecs_patterns.README.html)
class called
[ApplicationLoadBalancedEc2Service](https://docs.aws.amazon.com/cdk/api/latest/python/aws_cdk.aws_ecs_patterns/ApplicationLoadBalancedEc2Service.html)
. However, we'll create the 
[ECS cluster](https://docs.aws.amazon.com/cdk/api/latest/python/aws_cdk.aws_ecs/Cluster.html)
independently of the Pattern.


## Step 4: Create an ECS Cluster

*All code changes for the rest of the lab will be done in the file lab/lab_stack.py*

Create an ECS cluster and add an instance to it. If we had specific requirements
around the VPC configuration, we could have created a fresh one first and passed
it to the ECS cluster via the ```vpc``` parameter, this would have allowed us to 
specify details such as IP range and the maximum number of Availability Zones to 
use (by default, it will create subnets and NAT Gateways in 2 AZs.) 

Instead, we'll just let the ECS Cluster construct create it for us. 
Replace the contents of ``lab/lab_stack.py`` as below:

```
from aws_cdk import (
    core,
    aws_ec2 as ec2,
    aws_ecs as ecs,
    aws_ecs_patterns as ecs_patterns,
    aws_cloudwatch as cw
)


# For consistency with other languages, `cdk` is the preferred import name for
# the CDK's core module.  The following line also imports it as `core` for use
# with examples from the CDK Developer's Guide, which are in the process of
# being updated to use `cdk`.  You may delete this import if you don't need it.
from aws_cdk import core

class LabStack(core.Stack):

    def __init__(self, scope: core.Construct, construct_id: str, **kwargs) -> None:
        super().__init__(scope, construct_id, **kwargs)

        # The code that defines your stack goes here
        loadgen_cluster = ecs.Cluster(
            self, "Loadgen-Cluster",
        )
        
        loadgen_cluster.add_capacity("Asg",
            desired_capacity=1,
            instance_type=ec2.InstanceType("t3.micro"),
        )
        
```
Lets activate your python virtualenv, install your dependencies, and make sure CDK is up to date. 

```
source .env/bin/activate
pip3 install -r requirements.txt
npm -g upgrade
```

***Remember:Every time you work on your CDK project, you'll need to activate your virtualenv***

If you haven't used CDK in this account before, cdk bootstrap should be run to prepare your account. 
Below command will create an S3 bucket to store a small amount of CDK resources.

```
cd ~/environment/python-cdk-locust/lab
cdk bootstrap
```

Now run ``` cdk synth``` to synthesize your CloudFormation template. You should 
see a CloudFormation template several hundred lines long defining your ECS 
cluster and its dependencies.  

If you only see a metadata resource, you've forgotten to save you lab_stack.py file.

You can now deploy your template by running ```cdk deploy``` command. 
This should take about 5 minutes to complete.

## Step 5: Create a container and task definition

Now that we've got our ECS cluster, we need something to run on it. We'll start 
by defining a task definition. Add this to the end of the ``lab/lab_stack.py``
```
        task_def = ecs.Ec2TaskDefinition(self, "locustTask",
            network_mode=ecs.NetworkMode.AWS_VPC
        )
```

Now we'll add a container to run in our task. This container definition creates 
a new container image based on a DOCKERFILE which references the official Locust.io 
image on Dockerhub and accompanying locust.py file in the ```/locust``` directory.
The locust.py file defines the set of tasks each "user" that Locust creates will 
perform - you can find more information on how to write a locust.py file in the
locust.io docs [here](https://docs.locust.io/en/stable/writing-a-locustfile.html)
. CDK will then upload the image that it creates to an ECR repository that it
creates automatically.

We also set the environment variables that Locust requires to initialise here. Add below to the end of the ``lab/lab_stack.py``

```
        locust_container = task_def.add_container(
            "locustContainer",
            image=ecs.ContainerImage.from_asset("../locust"),
            memory_reservation_mib=512,
            essential=True,
            logging=ecs.LogDrivers.aws_logs(stream_prefix="cdkLocust"),
            environment={"TARGET_URL": "127.0.0.1"}
        )
        locust_container.add_port_mappings(ecs.PortMapping(container_port=8089))
```

## Step 6: Define a service

Now that we've created all of the underlying components, it's time to put them
together into a service and run it on our ECS cluster. For this we'll use an ECS
Pattern construct called
[ApplicationLoadBalancedEc2Service](https://docs.aws.amazon.com/cdk/api/latest/python/aws_cdk.aws_ecs_patterns/ApplicationLoadBalancedEc2Service.html)
which not only creates the service, but automatically puts it behind 
an Applicaion Load Balancer for us. Add below to the end of the ``lab/lab_stack.py``
```
        locust_service = ecs_patterns.ApplicationLoadBalancedEc2Service(self, "Locust", 
            memory_reservation_mib=512, 
            task_definition=task_def, 
            cluster=loadgen_cluster
        )
```


As we're using a Pattern, that's all that is required, as CDK uses the pattern to 
take care of the rest of the details. 

Run ```cdk diff``` to see what changes this will make to the CloudFormation that CDK
will generate and deploy.

Now deploy your stack using ```cdk deploy``` - it should take about 5 minutes to 
deploy. When that completes, in a web browser go to the LocustServiceURL which 
is output at the end of the deploy process. You should see your Locust load 
generator page.

You can test this by entering Target URL in the host field, and 1 
for the number of users and hatch rate (Don't set it too high or you'll DoS 
your Target!)

![LocustUI](/images/Locust.png)

## Oops! We need a bigger instance!

Imagine having just discovered that the t3.micro instance we selected for our ECS
cluster isn't adequate to handle the load we're generating. We need to move to a
C5.large instance.

With CDK, this is easy. Simply change the ```t3.micro``` in your ecs cluster to
```c5.large``` save the file, and redeploy with ```cdk deploy```.

## Monitoring
Obviously, just having our service running is not enough, we need to monitor its
operation. Let's create a simple CloudWatch Dashboard to monitor a couple of 
metrics for our service. 

Again, we can take advantage of the abstractions built into the CDK constructs. 
We'll build a new dashboard and add graph widgets for both the ECS cluster and 
ALB. Start by creating the graph widgets based on metric objects which are properties
of our 
[ECS cluster](https://docs.aws.amazon.com/cdk/api/latest/python/aws_cdk.aws_ecs/Cluster.html)
and 
[ALB](https://docs.aws.amazon.com/cdk/api/latest/python/aws_cdk.aws_elasticloadbalancingv2/ApplicationLoadBalancer.html). 
Add below to the end of the ``lab/lab_stack.py``.
```
        ecs_widget = cw.GraphWidget(
            left=[locust_service.service.metric_cpu_utilization()], 
            right=[locust_service.service.metric_memory_utilization()],
            title="ECS Service - CPU and Memory Reservation"
        )
            
        alb_widget = cw.GraphWidget(
            left=[locust_service.load_balancer.metric_request_count()],
            right=[locust_service.load_balancer.metric_processed_bytes()],
            title="ALB - Requests and Throughput"
        )
```

Now, we'll create a
[CloudWatch Dashboard](https://docs.aws.amazon.com/cdk/api/latest/python/aws_cdk.aws_cloudwatch/Dashboard.html)
using the
[CloudWatch Construct](https://docs.aws.amazon.com/cdk/api/latest/python/aws_cdk.aws_cloudwatch.README.html)
, and add them to a dashboard

```
        dashboard = cw.Dashboard(self, "Locustdashboard")
        dashboard.add_widgets(ecs_widget)
        dashboard.add_widgets(alb_widget)
```

Save your file, and deploy the changes using ```cdk deploy```

After a couple of minutes, when that's finished deploying, go to your CloudWatch 
console, click "Dashboards" on the left, the click the dashboard starting with 
"Locustdashboard". You should now see the graphs you just created.

![CloudWatch Dashboard](/images/Dashboard.png)

## Cleanup

Cleaning up in CDK is easy! Simply run ```cdk destroy``` and CDK will delete 
all deployed resources.

Delete your Cloud9 environment by going to the Cloud9 console, select your 
environment, click the Delete button, and follow the prompts. 

If you no longer intend to use CDK in your account, you can also delete your 
bootstrap resources by going to the CloudFormation console and deleting the
CDKToolkit stack. This will delete the S3 bucket that CDK created for storing 
temporary assets. Finally, go to the Elastic Container Repository console and
delete the "aws-cdk/assets" repository.

## Survey
Please help us to provide your feedback [here](https://amazonmr.au1.qualtrics.com/jfe/form/SV_6x7UgBL9FHn59dA?Session=HOL2). Participants who complete the surveys from AWS Innovate Online Conference - Modern Applications Edition will receive a gift code for USD25 in AWS credits. AWS credits will be sent via email by 30 November, 2021.
