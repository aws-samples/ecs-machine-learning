
# Orchestrating GPU-Accelerated Workloads on Amazon ECS

## Background
AWS Solutions Architects are seeing an emerging type of application for ECS: GPU-accelerated workloads, or, more specifically, workloads that need to leverage large amounts of GPUs across many nodes. For example, at Amazon.com, the Amazon Personalization Team runs significant Machine Learning workloads that leverage many GPUs on Amazon ECS. Let’s take a look at how ECS enables GPU workloads. 

## Solution

In order to run GPU-enabled work on an ECS cluster, a Docker image configured with [Nvidia CUDA drivers][1], which allow the container to communicate with the GPU hardware, is built and stored in EC2 Container Registry. An [ECS Task Definition][2] is used to point to the container image in ECR and specify configuration for the container at runtime, like how much CPU and memory each container should use, the command to run inside the container, if a data volume should be mounted in the container, where the source dataset lives in S3, and so on. 

Once the ECS Tasks are run, the ECS [scheduler][3] finds a suitable place to run the containers by identifying an instance in the cluster with available resources. As shown in the below architecture diagram, ECS can place containers into the cluster of GPU instances (“GPU slaves” in the diagram)

<img src="https://s3.amazonaws.com/ecs-machine-learning/architecture.png" width="450" height="316">

## Deploying the architecture

In this template, we spin up an ECS Cluster with a single GPU instance in an autoscaling group. You can, however, adjust the ASG desired capacity to run a larger cluster if you’d like. The instance is configured with all of the necessary software, like Nvidia drivers, that DSSTNE requires for interaction with the underlying GPU hardware. We also install some development tools, like Make and GCC, so that we can compile the DSSTNE library at boot time. We then build a Docker container with the DSSTNE library packaged up and upload it to EC2 Container Registry. We grab the URL of the resulting container image in ECR and build an ECS Task Definition that points to the container. 

Once the CloudFormation template completes, take a look at the “Outputs” tab to get an idea of where to look for your new resources. 

### Prerequisistes

#### Network configuration
The instances launched will need to have access to the internet hence either be in a public subnet and provided a public IP or in a private subnet with acess to a NAT gateway.

#### Accepting terms

0. Accept AWS Marketplace terms for Amazon Linux AMI with NVIDIA GRID GPU Driver by going to the [MarketPlace page][9]

1. Click Continue on the right.

2. Click on the Manual Launch tab and click on the Accept Software Terms button.

3. Wait for an email confirmation that your marketplace subscription is active.

### Launch the stack

2. Choose **Launch Stack** to launch the template in the us-east-1 region in your account:
[![Launch ECS Machine Learning into North Virginia with CloudFormation](http://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/images/cloudformation-launch-stack-button.png)](https://console.aws.amazon.com/cloudformation/home?region=us-east-1#/stacks/new?stackName=ecs-machine-learning&templateURL=https://s3.amazonaws.com/ecs-machine-learning/machinelearning.template)

(The template will build a DSSTNE container on the ECS cluster instance. Note this can take up to 20 minutes and the CloudFormation stack will not report completion until the entire build process is done.)

2. Give a Stack Name and select your preferred key name. If you do not have a key available, see [Amazon EC2 Key Pairs][4].

### Run the model

3. Find the name of the DSSTNE ECS Task Definition in CloudFormation stack outputs. It will start with "arn:aws[...]" and contain the CloudFormation template name right after "task-defintion/".

4. Go to the [ECS Console][10], click on Task Definitions (left column) and find the one you spotted in the step above. 

5. Tick the one revision you see. Click on the Actions drop-down menu and hit Run task. Make sure to select the  ECS Cluster that was brought up by the CloudFormation template. By running this task, you are essentially running the DSSTNE sample modeling as described on the [amazon-dsstne GitHub page][5].

6. You can easily check that the  GPU is being used by logging to the EC2 instance and running `watch -n1 nvidia-smi`

### Collect predictions

5. You should be able to find the name of the relevant [CloudWatch][7] Logs Group in CloudFormation stack outputs

6. Look at the task logs for details and output from the task run and location of results file in S3

8. Navigate to this S3 bucket via the [S3 Console][11]. This is where you will be able to access the results file and confirm that this GPU-enabled Machine Learning run was successful.

### Bonus activities
- Bonus activity #1: Repeat step 2 in the *Run the model* section, but change the config URL and training command by overriding the task definition environment variables to perform a [benchmark][6].
- Bonus activity #2: Modify the CloudFormation template (or launch a new stack) and use a g2.8xlarge instead of g2.2xlarge (you could also try a [P2 instance][12]...) Repeat step 2 in the *Run the model* section, but override the training command in the task definition environment variables to use MPI to take advantage of all 4 GPUs: add `mpirun -np <NUMBER OF CPU>` in front of the `train` command. For this you will have to create a new revision of the task definition.

In both bonus steps, look at the CloudWatch Logs to view the task logs (different training commands, taking advantage of multiple GPUs, etc.)

## Conclusion

You should now have a good grasp on how to leverage ECS and GPU-optimized EC2 instances for your Machine Learning needs. Head on over to the [AWS Big Data blog][8] to learn more about how DSSTNE interacts with Apache Spark, trains models, generates predictions, and other fun Machine Learning concepts. 


[1]: http://www.nvidia.com/object/cuda_home_new.html
[2]: http://docs.aws.amazon.com/AmazonECS/latest/developerguide/task_defintions.html
[3]: http://docs.aws.amazon.com/AmazonECS/latest/developerguide/scheduling_tasks.html
[4]: http://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-key-pairs.html
[5]: https://github.com/amznlabs/amazon-dsstne/blob/master/docs/getting_started/examples.md
[6]: https://github.com/amznlabs/amazon-dsstne/blob/master/benchmarks/Benchmark.md
[7]: https://console.aws.amazon.com/cloudwatch/home
[8]: https://blogs.aws.amazon.com/bigdata/post/TxGEL8IJ0CAXTK/Generating-Recommendations-at-Amazon-Scale-with-Apache-Spark-and-Amazon-DSSTNE
[9]: https://aws.amazon.com/marketplace/pp/B00FYCDDTE
[10]: https://console.aws.amazon.com/ecs/
[11]: https://console.aws.amazon.com/s3/home
[12]: https://aws.amazon.com/ec2/instance-types/p2/
