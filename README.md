# Non-Java KCL on EKS Fargate

In this example, you will see how to run non-Java consumer application for an Amazon Kinesis Data Streams on AWS Fargate using Amazon EKS. Amazon Kinesis is a fully managed and scalable data stream which enables you to ingest, buffer and process data in real-time. AWS Fargate is a serverless compute engine for containers that works with AWS container orchestration services like Amazon Elastic Kubernetes Service. This is useful in scenarios where you want to leverage Kubernetes for hosting Amazon Kinesis Data Streams consumer application.

In this example we will deploy following AWS resources

1. Python Kinesis consumer application using [Amazon Kinesis Client Library for Python](https://github.com/awslabs/amazon-kinesis-client-python) in EKS Fargate pod
2. Kinesis autoscaler application using [Kinesis Scaling Utility](https://github.com/awslabs/amazon-kinesis-scaling-utils) in EKS Fargate pod
3. Amazon EKS Fargate cluster (you can also use existing EKS cluster
4. Amazon ECR repository for hosting docker images for consumer and autoscaler application
5. CodeBuild pipeline to build docker images
6. AWS IAM Managed Policy


*Note - We will use AWS region us-west-2 in below example. Please change it accordingly with your preference.*

## How to use it?

### 1. Clone git repository to your local system

```
$ git clone https://github.com/aws-samples/amazon-kinesis-eks-fargate-consumer .
```

### 2. Deploy CloudFormation template

The CloudFormation template can be found in *cloudformation* folder. Once deployed it will create following resources in your account

1. Amazon Kinesis Data Stream
2. Docker image for Kinesis consumer application and host in Amazon Elastic Container Registry
3. Docker image for Kinesis autoscaler application and host in Amazon Elastic Container Registry
4. AWS IAM Managed policies for Kinesis consumer and autoscaler application (to be used in IAM service account later)

Note: After CloudFormation deployment is completed, confirm if the docker images are created in Amazon Elastic Container Registry. Sometime the images are not created due to Docker throttling limit.

### 3. Create Amazon EKS Fargate cluster

Using *eksctl* command create Amazon EKS Fargate cluster. In case, you already have a cluster skip to Step #4


```
$ eksctl create cluster --name eks-fargate-cluster --version 1.18 --region us-west-2 --fargate --alb-ingress-access
```

The output of this command would be as following

```
[ℹ] eksctl version 0.36.1
[ℹ] using region us-west-2
[ℹ] setting availability zones to [us-west-2d us-west-2a us-west-2c]
[ℹ] subnets for us-west-2d - public:192.168.0.0/19 private:192.168.96.0/19
[ℹ] subnets for us-west-2a - public:192.168.32.0/19 private:192.168.128.0/19
[ℹ] subnets for us-west-2c - public:192.168.64.0/19 private:192.168.160.0/19
[ℹ] using Kubernetes version 1.18
[ℹ] creating EKS cluster "eks-fargate-cluster" in "us-west-2" region with Fargate profile
[ℹ] if you encounter any issues, check CloudFormation console or try 'eksctl utils describe-stacks —region=us-west-2 —cluster=eks-fargate-cluster’
[ℹ] CloudWatch logging will not be enabled for cluster "eks-fargate-cluster" in "us-west-2"
[ℹ] you can enable it with 'eksctl utils update-cluster-logging --enable-types={SPECIFY-YOUR-LOG-TYPES-HERE (e.g. all)} —region=us-west-2 —cluster=eks-fargate-cluster’
[ℹ] Kubernetes API endpoint access will use default of {publicAccess=true, privateAccess=false} for cluster "eks-fargate-cluster" in "us-west-2"
[ℹ] 2 sequential tasks: { create cluster control plane "eks-fargate-cluster", 2 sequential sub-tasks: { create fargate profiles, create addons } }
[ℹ] building cluster stack "eksctl-eks-fargate-cluster-cluster"
[ℹ] deploying stack "eksctl-eks-fargate-cluster-cluster"
[ℹ] waiting for CloudFormation stack "eksctl-eks-fargate-cluster-cluster"
[ℹ] creating Fargate profile "fp-default" on EKS cluster "eks-fargate-cluster"
[ℹ] created Fargate profile "fp-default" on EKS cluster "eks-fargate-cluster"
[ℹ] "coredns" is now schedulable onto Fargate
[ℹ] "coredns" is now scheduled onto Fargate
[ℹ] "coredns" pods are now scheduled onto Fargate
[ℹ] waiting for the control plane availability...
[✔] saved kubeconfig as "/Users/malviyd/.kube/config"
[ℹ] no tasks
[✔] all EKS cluster resources for "eks-fargate-cluster" have been created
[ℹ] kubectl command should work with "/Users/testuser/.kube/config", try 'kubectl get nodes'
[✔] EKS cluster "eks-fargate-cluster" in "us-west-2" region is ready
```

### 4. Create Fargate Profile

The Fargate profile is use to declare pods that should run on AWS Fargate. This command will create Fargate profile dedicated to run consumer and autoscaler application in Kubernetes pod.


```
$ eksctl create fargateprofile --namespace kcl-processor --cluster eks-fargate-cluster --region us-west-2
```

The output of above command is following

```
[ℹ]  creating Fargate profile "fp-fe4a7ca1" on EKS cluster "eks-fargate-cluster"
[ℹ]  created Fargate profile "fp-fe4a7ca1" on EKS cluster "eks-fargate-cluster"
```

### 5. Create OIDC ID provider in AWS

The pods used to run consumer and autoscaler application would require access for Kinesis and other services to work properly. .The OIDC ID provider is created and associated with EKS cluster so that pods can utilize IAM roles for Service Accounts to authenticate.

```
$ eksctl utils associate-iam-oidc-provider --cluster eks-fargate-cluster —approve
```

The output of above command is as following

```
[ℹ] eksctl version 0.36.1
[ℹ] using region us-west-2
[ℹ] will create IAM Open ID Connect provider for cluster "eks-fargate-cluster" in "us-west-2"
[✔] created IAM Open ID Connect provider for cluster "eks-fargate-cluster" in "us-west-2"
```

### 6. Kubernetes service account and IAM role setup

Next, Kubernetes service account and IAM role are created for both consumer and autoscaler application. Open CloudFormation output tab from stack deployed and copy IAM policy ARN of KCL (“*KCL_POLICY_ARN*”) to replace with IAM-POLICY in this command.

```
$ eksctl create iamserviceaccount --name kcl-consumer-sa --cluster eks-fargate-cluster --namespace kcl-processor --attach-policy-arn arn:aws:iam::123456789012:policy/CustomEKSFargateKCLPolicy —approve
```

The output of above command will look like below


```
[ℹ] eksctl version 0.36.1
[ℹ] using region us-west-2
[ℹ] 1 iamserviceaccount (kcl-processor/kcl-consumer-sa) was included (based on the include/exclude rules)
[!] serviceaccounts that exists in Kubernetes will be excluded, use --override-existing-serviceaccounts to override
[ℹ] 1 task: { 2 sequential sub-tasks: { create IAM role for serviceaccount "kcl-processor/kcl-consumer-sa", create serviceaccount "kcl-processor/kcl-consumer-sa" } }
[ℹ] building iamserviceaccount stack "eksctl-eks-fargate-cluster-addon-iamserviceaccount-kcl-processor-kcl-consumer-sa"
[ℹ] deploying stack "eksctl-eks-fargate-cluster-addon-iamserviceaccount-kcl-processor-kcl-consumer-sa"
[ℹ] waiting for CloudFormation stack "eksctl-eks-fargate-cluster-addon-iamserviceaccount-kcl-processor-kcl-consumer-sa"
[ℹ] waiting for CloudFormation stack "eksctl-eks-fargate-cluster-addon-iamserviceaccount-kcl-processor-kcl-consumer-sa"
[ℹ] waiting for CloudFormation stack "eksctl-eks-fargate-cluster-addon-iamserviceaccount-kcl-processor-kcl-consumer-sa"
[ℹ] created namespace "kcl-processor"
[ℹ] created serviceaccount "kcl-processor/kcl-consumer-sa"
```

Similarly, create Kubernetes service account and IAM roles for autoscaler application. Copy IAM policy ARN of Autoscaler (“*KINESIS_AUTOSCALER_POLICY_ARN*”) to replace with IAM-POLICY in below command.

```
$ eksctl create iamserviceaccount --name kcl-autoscaler-sa --cluster eks-fargate-cluster --namespace kcl-processor --attach-policy-arn arn:aws:iam::123456789012:policy/CustomEKSFargateKinesisAutoscalerPolicy --approve
```

The output of above command will look like

```
[ℹ] eksctl version 0.36.1
[ℹ] using region us-west-2
[ℹ] 1 existing iamserviceaccount(s) (kcl-processor/kcl-consumer-sa) will be excluded
[ℹ] 1 iamserviceaccount (kcl-processor/kcl-autoscaler-sa) was included (based on the include/exclude rules)
[!] serviceaccounts that exists in Kubernetes will be excluded, use --override-existing-serviceaccounts to override
[ℹ] 1 task: { 2 sequential sub-tasks: { create IAM role for serviceaccount "kcl-processor/kcl-autoscaler-sa", create serviceaccount "kcl-processor/kcl-autoscaler-sa" } }
[ℹ] building iamserviceaccount stack "eksctl-eks-fargate-cluster-addon-iamserviceaccount-kcl-processor-kcl-autoscaler-sa"
[ℹ] deploying stack "eksctl-eks-fargate-cluster-addon-iamserviceaccount-kcl-processor-kcl-autoscaler-sa"
[ℹ] waiting for CloudFormation stack "eksctl-eks-fargate-cluster-addon-iamserviceaccount-kcl-processor-kcl-autoscaler-sa"
[ℹ] waiting for CloudFormation stack "eksctl-eks-fargate-cluster-addon-iamserviceaccount-kcl-processor-kcl-autoscaler-sa"
[ℹ] waiting for CloudFormation stack "eksctl-eks-fargate-cluster-addon-iamserviceaccount-kcl-processor-kcl-autoscaler-sa"
[ℹ] created serviceaccount "kcl-processor/kcl-autoscaler-sa"
```


The above commands will also create a namespace in Kubernetes that we’ll use in configuring pods

```
$ kubectl get ns
```

The out of above command is

```
NAME STATUS AGE
default Active 66m
kcl-processor Active 2m20s
kube-node-lease Active 66m
kube-public Active 66m
kube-system Active 66m
```

### 7. Create Consumer and Autoscaler pods

From the CloudFormation output section of deployed stack, make note of values of these fields. They will be used in consumer and autoscaler spec file.
*CONSUMER_ECR_REPOSITORY_URI*
*AUTOSCALER_ECR_REPOSITORY_URI*

Open kubernetes/consumer.yaml to replace <CONSUMER_ECR_REPOSITORY_URI> value. After these changes, deploy consumer application

```
$ kubectl apply -f consumer.yaml
```

Similarly, open kubernetes/autoscaler.yaml to replace <AUTOSCALER_ECR_REPOSITORY_URI> value.

Deploy autoscaler application

```
$ kubectl apply -f autoscaler.yaml
```

Confirm if both are pods are running successfully

```
$ kubectl get pod -n kcl-processor
NAME READY STATUS RESTARTS AGE
autoscaler 1/1 Running 0 7m15s
consumer-xxxxxxxxx-xxxx 1/1 Running 0 52s
```

Check the pod logs to verify if there are no errors and everything is running successfully.

### 8. Send records to KDS

You can send the records in the Kinesis data stream created to test the data ingestion. You can use [Amazon Kinesis Data Generator](https://awslabs.github.io/amazon-kinesis-data-generator/) to simulate Kinesis producer that write to Kinesis data stream.

The data will be ingested by Kinesis consumer application. The Kinesis consumer application used Python KCL library that uses the AWS Java Kinesis SDK’s MultiLangDaemon. The MultiLangDaemon uses STDIN and STDOUT to communicate with the record processor hence be aware of logging limitations. Due to the STDOUT limitation, the record processor logs data records to a file which is written to the container logs.
