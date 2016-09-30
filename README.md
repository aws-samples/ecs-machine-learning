
# Orchestrating GPU-Accelerated Workloads on Amazon ECS

## Background
AWS Solutions Architects are seeing an emerging type of application for ECS: GPU-accelerated workloads, or, more specifically, workloads that need to leverage large amounts of GPUs across many nodes. For example, at Amazon.com, the Amazon Personalization Team runs significant Machine Learning workloads that leverage many GPUs on Amazon ECS. Let’s take a look at how ECS enables GPU workloads. 

## Solution

In order to run GPU-enabled work on an ECS cluster, a Docker image configured with [Nvidia CUDA drivers][1], which allow the container to communicate with the GPU hardware, is built and stored in EC2 Container Registry. An [ECS Task Definition][2] is used to point to the container image in ECR and specify configuration for the container at runtime, like how much CPU and memory each container should use, the command to run inside the container, if a data volume should be mounted in the container, where the source dataset lives in S3, and so on. 

Once the ECS Tasks are run, the ECS [scheduler][3] finds a suitable place to run the containers by identifying an instance in the cluster with available resources. As shown in the below architecture diagram, ECS can place containers into the cluster of GPU instances (“GPU slaves” in the diagram)

<img src="https://s3.amazonaws.com/ecs-machine-learning/architecture.png" width="450" height="316">

## Deploying the architecture

In this template, we spin up an ECS Cluster with a single GPU instance in an autoscaling group. You can, however, adjust the ASG desired capacity to run a larger cluster if you’d like. The instance is configured with all of the necessary software, like Nvidia drivers, that DSSTNE requires for interaction with the underlying GPU hardware. We also install some development tools, like Make and GCC, so that we can compile the DSSTNE library at boot time. We then build a Docker container with the DSSTNE library packaged up and upload it to EC2 Container Registry. We grab the URL of the resulting container image in ECR and build an ECS Task Definition that points to the container. 

Once the Cloudformation template completes, take a look at the “Outputs” tab to get an idea of where to look for your new resources. 

### Prerequisistes: accepting terms

0. Accept AWS Marketplace terms for Amazon Linux AMI with NVIDIA GRID GPU Driver by going to the [MarketPlace page][9]

1. Click Continue on the right.

2. Click on the Manual Launch tab and click on the Accept Software Terms button.

3. Wait for an email confirmation that your marketplace subscription is active.

### Launch the stack

2. Choose **Launch Stack** to launch the template in the us-east-1 region in your account:
[![Launch ECS Machine Learning into North Virginia with CloudFormation](http://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/images/cloudformation-launch-stack-button.png)](https://console.aws.amazon.com/cloudformation/home?region=us-east-1#/stacks/new?stackName=ecs-service-discovery&templateURL=https://s3.amazonaws.com/ecs-machine-learning/machinelearning.template)

(The template will build a DSSTNE container on the ECS cluster instance. Note this can take 30-45 minutes and the cloudformation stack will not report completion until the entire build process is done.)

2. Give a Stack Name and select your preferred key name. If you do not have a key available, see [Amazon EC2 Key Pairs][4].

### Run the model

3. Find name of DSSTNE ECS Task Definition in cloudformation stack outputs

4. Run task on DSSTNE ECS Cluster (defaults should work fine). By running this task, you are essentially running the DSSTNE sample modeling as described on the [amazon-dsstne GitHub page][5].

### Collect predictions

5. Find name of [CloudWatch][7] Logs Group in cloudformation stack outputs

6. Look at the task logs for details and output from the task run and location of results file in S3

7. Find name of S3 Bucket for the results in the cloudformation stack outputs

8. View the results file in the S3 Bucket

### Bonus steps
9. Bonus step 1: Repeat step 2 in the *Run the model* section, but change the config URL and training command by overriding the task definition environment variables to perform a [benchmark][6].

10. Bonus step 2: Modify the cfntemplate (or launch a new stack) and use a g2.8xlarge instead of g2.2xlarge. Repeat step 2 in the *Run the model* section, but override the training command in the task definition environment variables to use MPI to take advantage of all 4 GPUs: mpirun -np 4 train -c ....

11. In both bonus steps, look at the CloudWatch Logs to view the task logs (different training commands, taking advantage of multiple GPUs, etc.)

## Conclusion

Unfortunately, it would take far too much page space to explain the details of the Machine Learning specifics in this post, so head on over to the [AWS Big Data blog][8] to learn more about how DSSTNE interacts with Apache Spark, trains models, generates predictions, and other fun Machine Learning concepts. 


[1]: http://www.nvidia.com/object/cuda_home_new.html
[2]: http://docs.aws.amazon.com/AmazonECS/latest/developerguide/task_defintions.html
[3]: http://docs.aws.amazon.com/AmazonECS/latest/developerguide/scheduling_tasks.html
[4]: http://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-key-pairs.html
[5]: https://github.com/amznlabs/amazon-dsstne/blob/master/docs/getting_started/examples.md
[6]: https://github.com/amznlabs/amazon-dsstne/blob/master/benchmarks/Benchmark.md
[7]: https://console.aws.amazon.com/cloudwatch/home?region=us-east-1
[8]: https://blogs.aws.amazon.com/bigdata/post/TxGEL8IJ0CAXTK/Generating-Recommendations-at-Amazon-Scale-with-Apache-Spark-and-Amazon-DSSTNE
[9]: https://aws.amazon.com/marketplace/pp/B00FYCDDTE