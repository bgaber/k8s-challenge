## Setup EKS Management Console

The purpose of the EKS Management Console (EMC) is to manage the EKS Cluster and EKS worker nodes. The EMC is built on a small EC2 instance. I used t3a.micro. Among many functions it is the EMC that creates, updates and deletes all Clusters and Node Groups. The EMC is created by installing the AWS CLI, kubectl, eksctl and helm software packages.

### Create IAM User

Create IAM user with Programmatic access only and attach the AdministratorAccess policy. Copy the Access Key ID and Secret Access Key to be used when setting up the AWS CLI.

### Install Required Software

***Install the AWS CLI version 2***

```
$ curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install --bin-dir /usr/bin --install-dir /usr/bin/aws-cli --update
```

Configure the AWS CLI using the Access Key ID and Secret Access Key.

```
aws configure
```

***Install kubectl***

```
curl -O https://s3.us-west-2.amazonaws.com/amazon-eks/1.29.0/2024-01-04/bin/linux/amd64/kubectl
sudo chmod +x ./kubectl
sudo mv kubectl /usr/local/bin
kubectl version
```

***Install eksctl***

```
curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
sudo mv /tmp/eksctl /usr/local/bin
eksctl version
```

***Install helm*** Helm is a Kubernetes deployment tool for automating creation, packaging, configuration, and deployment of applications and services to Kubernetes clusters.

```
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
chmod 700 get_helm.sh
./get_helm.sh
which helm
helm version
```

## Create EKS Cluster

There are many options for creating an EKS cluster using eksctl. It can be created using a k8s manifest file or from the command line. From the CLI there are also many options as shown by running the eksctl create cluster --help command. I will just show two variations: one that creates a new VPC and required components (subnets, etc) and one that uses an existing VPC and subnets.

### Deploy Cluster To New VPC From CLI

The following command creates a new EKS cluster in a new VPC and required components (subnets, etc):

```
eksctl create cluster --name k8s-challenge --version 1.29 --region us-east-1 --zones us-east-1a,us-east-1b,us-east-1c,us-east-1d,us-east-1f --nodegroup-name standard-workers --node-type t3a.xlarge --nodes 2 --nodes-min 1 --nodes-max 4 --managed
```

### Deploy Cluster To Existing VPC From CLI

The following command creates a new EKS cluster in an existing VPC and subnets with the worker nodes only having private IP addresses:

```
eksctl create cluster --name k8s-challenge --version 1.29 --region us-east-1 --vpc-public-subnets=subnet-42267f68,subnet-c2565fb4 --vpc-private-subnets=subnet-2a590000,subnet-63696015 --nodegroup-name standard-workers --node-type t3a.xlarge --nodes 2 --nodes-min 1 --nodes-max 4 --node-private-networking --managed
```

### Deploy Cluster To Existing VPC With eskctl Manifest

The following k8s manifest performs the same EKS deployment as the eksctl command above. That is, it creates a new EKS cluster in an existing VPC and subnets with the worker nodes only having private IP addresses:

```
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig
metadata:
  name: k8s-challenge
  region: us-east-1
  version: "1.29"
managedNodeGroups:
- amiFamily: AmazonLinux2
  desiredCapacity: 2
  iam:
    withAddonPolicies:
      ebs: true
      awsLoadBalancerController: true
      cloudWatch: true
  instanceType: t3a.xlarge
  labels:
    alpha.eksctl.io/cluster-name: k8s-challenge
    alpha.eksctl.io/nodegroup-name: standard-workers
  maxSize: 3
  minSize: 1
  name: k8s-challenge-std-workers
  privateNetworking: false
  tags:
    alpha.eksctl.io/nodegroup-name: standard-workers
    alpha.eksctl.io/nodegroup-type: managed
vpc:
  id: vpc-f069ee94
  manageSharedNodeSecurityGroupRules: true
  nat:
    gateway: Disable
  subnets:
    public:
      us-east-1a:
        id: subnet-817978aa
      us-east-1b:
        id: subnet-3cc8344a
      us-east-1c:
        id: subnet-a8abbcf1
      us-east-1d:
        id: subnet-f4aa3191
      us-east-1f:
        id: subnet-80f87c8c
```

The EKS cluster is created with this manifest using this command:

eksctl create cluster -f k8s-challenge.yaml

You can monitor the build from CloudFormation in the AWS Console. As well you will see output to the eksctl command similar to the following:

```
2024-03-13 09:13:24 [ℹ]  eksctl version 0.173.0
2024-03-13 09:13:24 [ℹ]  using region us-east-1
2024-03-13 09:13:24 [✔]  using existing VPC (vpc-87e65ae0) and subnets (private:map[us-east-1b:{subnet-2a590000 us-east-1b 10.251.36.0/22 0} us-east-1c:{subnet-63696015 us-east-1c 10.251.40.0/22 0}] public:map[us-east-1b:{subnet-42267f68 us-east-1b 10.251.32.0/23 0} us-east-1c:{subnet-c2565fb4 us-east-1c 10.251.34.0/23 0}])
2024-03-13 09:13:24 [!]  custom VPC/subnets will be used; if resulting cluster doesn\'t function as expected, make sure to review the configuration of VPC/subnets
2024-03-13 09:13:24 [ℹ]  nodegroup "standard-workers" will use "" [AmazonLinux2/1.29]
2024-03-13 09:13:24 [ℹ]  using Kubernetes version 1.29
2024-03-13 09:13:24 [ℹ]  creating EKS cluster "k8s-challenge" in "us-east-1" region with managed nodes
2024-03-13 09:13:24 [ℹ]  1 nodegroup (standard-workers) was included (based on the include/exclude rules)
2024-03-13 09:13:24 [ℹ]  will create a CloudFormation stack for cluster itself and 0 nodegroup stack(s)
2024-03-13 09:13:24 [ℹ]  will create a CloudFormation stack for cluster itself and 1 managed nodegroup stack(s)
2024-03-13 09:13:24 [ℹ]  if you encounter any issues, check CloudFormation console or try \'eksctl utils describe-stacks --region=us-east-1 --cluster=k8s-challenge\'
2024-03-13 09:13:24 [ℹ]  Kubernetes API endpoint access will use default of {publicAccess=true, privateAccess=false} for cluster "k8s-challenge" in "us-east-1"
2024-03-13 09:13:24 [ℹ]  CloudWatch logging will not be enabled for cluster "k8s-challenge" in "us-east-1"
2024-03-13 09:13:24 [ℹ]  you can enable it with \'eksctl utils update-cluster-logging --enable-types={SPECIFY-YOUR-LOG-TYPES-HERE (e.g. all)} --region=us-east-1 --cluster=k8s-challenge\'
2024-03-13 09:13:24 [ℹ]
2 sequential tasks: { create cluster control plane "k8s-challenge",
    2 sequential sub-tasks: {
        wait for control plane to become ready,
        create managed nodegroup "standard-workers",
    }
}
2024-03-13 09:13:24 [ℹ]  building cluster stack "eksctl-k8s-challenge-cluster"
2024-03-13 09:13:25 [ℹ]  deploying stack "eksctl-k8s-challenge-cluster"
2024-03-13 09:13:55 [ℹ]  waiting for CloudFormation stack "eksctl-k8s-challenge-cluster"
2024-03-13 09:14:25 [ℹ]  waiting for CloudFormation stack "eksctl-k8s-challenge-cluster"
2024-03-13 09:15:25 [ℹ]  waiting for CloudFormation stack "eksctl-k8s-challenge-cluster"
2024-03-13 09:16:25 [ℹ]  waiting for CloudFormation stack "eksctl-k8s-challenge-cluster"
2024-03-13 09:17:25 [ℹ]  waiting for CloudFormation stack "eksctl-k8s-challenge-cluster"
2024-03-13 09:18:25 [ℹ]  waiting for CloudFormation stack "eksctl-k8s-challenge-cluster"
2024-03-13 09:19:25 [ℹ]  waiting for CloudFormation stack "eksctl-k8s-challenge-cluster"
2024-03-13 09:20:25 [ℹ]  waiting for CloudFormation stack "eksctl-k8s-challenge-cluster"
2024-03-13 09:21:25 [ℹ]  waiting for CloudFormation stack "eksctl-k8s-challenge-cluster"
2024-03-13 09:22:25 [ℹ]  waiting for CloudFormation stack "eksctl-k8s-challenge-cluster"
2024-03-13 09:23:25 [ℹ]  waiting for CloudFormation stack "eksctl-k8s-challenge-cluster"
2024-03-13 09:24:25 [ℹ]  waiting for CloudFormation stack "eksctl-k8s-challenge-cluster"
2024-03-13 18:54:40 [ℹ]  building managed nodegroup stack "eksctl-k8s-challenge-nodegroup-k8s-challenge-std-workers"
2024-03-13 18:54:40 [ℹ]  deploying stack "eksctl-k8s-challenge-nodegroup-k8s-challenge-std-workers"
2024-03-13 18:54:40 [ℹ]  waiting for CloudFormation stack "eksctl-k8s-challenge-nodegroup-k8s-challenge-std-workers"
2024-03-13 18:55:10 [ℹ]  waiting for CloudFormation stack "eksctl-k8s-challenge-nodegroup-k8s-challenge-std-workers"
2024-03-13 18:56:05 [ℹ]  waiting for CloudFormation stack "eksctl-k8s-challenge-nodegroup-k8s-challenge-std-workers"
2024-03-13 18:57:52 [ℹ]  waiting for CloudFormation stack "eksctl-k8s-challenge-nodegroup-k8s-challenge-std-workers"
2024-03-13 18:57:52 [ℹ]  waiting for the control plane to become ready
2024-03-13 18:57:52 [✔]  saved kubeconfig as "/home/ec2-user/.kube/config"
2024-03-13 18:57:52 [ℹ]  no tasks
2024-03-13 18:57:52 [✔]  all EKS cluster resources for "k8s-challenge" have been created
2024-03-13 18:57:52 [ℹ]  nodegroup "k8s-challenge-std-workers" has 2 node(s)
2024-03-13 18:57:52 [ℹ]  node "ip-172-31-21-100.ec2.internal" is ready
2024-03-13 18:57:52 [ℹ]  node "ip-172-31-54-80.ec2.internal" is ready
2024-03-13 18:57:52 [ℹ]  waiting for at least 1 node(s) to become ready in "k8s-challenge-std-workers"
2024-03-13 18:57:52 [ℹ]  nodegroup "k8s-challenge-std-workers" has 2 node(s)
2024-03-13 18:57:52 [ℹ]  node "ip-172-31-21-100.ec2.internal" is ready
2024-03-13 18:57:52 [ℹ]  node "ip-172-31-54-80.ec2.internal" is ready
2024-03-13 18:57:53 [ℹ]  kubectl command should work with "/home/ec2-user/.kube/config", try 'kubectl get nodes'
2024-03-13 18:57:53 [✔]  EKS cluster "k8s-challenge" in "us-east-1" region is ready
```

### Only Deploy Nodegroup With eskctl Manifest

There may be occasions where you only want to deploy the Nodegroup on an existing EKS Cluster and not deploy the cluster. Two possibilities are one, where you want to change one of the Nodegroup parameters, like instanceType and two, the build of the cluster was interrupted by loss of Internet or VPN during the running of the eksctl cluster create command resulting in the cluster being deployed, but not the Nodegroup.

In the first case, to change one of the Nodegroup parameters, you first have to remove the existing Nodegroup. To do this you first need to get the Nodegroup name. This can be done with the eksctl get nodegroup --cluster <CLUSTER NAME> command. For example, in our case, the following commands will delete the Nodegroup:

```
eksctl get nodegroup --cluster k8s-challenge
CLUSTER         NODEGROUP                       STATUS  CREATED                 MIN SIZE        MAX SIZE        DESIRED CAPACITY        INSTANCE TYPE   IMAGE ID        ASG NAME                 TYPE
k8s-challenge   k8s-challenge-std-workers       ACTIVE  2024-03-13T18:55:05Z    1               3               2                       t3a.xlarge      AL2_x86_64      eks-k8s-challenge-std-workers-52c71c95-816c-abb4-25ca-89a142391d31        managed
```

```
eksctl delete nodegroup --cluster k8s-challenge --name standard-workers
```

To only deploy the Nodegroup and not the EKS Cluster you need an eksctl manifest file similar to the one shown above and run the command: eksctl create nodegroup -f k8s-challenge-cluster.yaml

Notice the command this time is eksctl create nodegroup and not eksctl create cluster.

### AWS CSI Storage Add-on

The Amazon Elastic Block Store (Amazon EBS) Container Storage Interface (CSI) driver manages the lifecycle of Amazon EBS volumes as storage for the Kubernetes Volumes that you create. The Amazon EBS CSI driver makes Amazon EBS volumes for these types of Kubernetes volumes: generic ephemeral volumes and persistent volumes

***Install the CSI Storage add-on

### AWS CNI Network Add-on

### AWS Load Balancer Controller Add-on

The AWS Load Balancer Controller requires subnets with specific tags. ***Read this document for information on the subnet tags

Public subnets are used for internet-facing load balancers. These subnets must have the following tags:

```
kubernetes.io/role/elb	1 or ``
```

Private subnets are used for internal load balancers. These subnets must have the following tags:

```
kubernetes.io/role/internal-elb	1 or ``
```

In order to use an ALB, referred to as an Ingress in Kubernetes, you must first install the AWS Load Balancer Controller add-on following ***these AWS instructions.***

Ensure you create the AmazonEKSLoadBalancerControllerRole IAM Role described in Steps 1 and 2.

In step 5 of these instructions I recommend using the Helm 3 or later instructions, rather than the Kubernetes manifest instructions.

Here is the helm install command I used:

```
helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=k8s-challenge \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller \
```

As shown in step 6 of the AWS instructions, I confirmed the deployment with this command:

```
kubectl get deployment -n kube-system aws-load-balancer-controller
NAME                           READY   UP-TO-DATE   AVAILABLE   AGE
aws-load-balancer-controller   2/2     2            2           3m23s
```

If the READY column reports 0/2 for a long time then the AmazonEKSLoadBalancerControllerRole IAM Role is probably not setup correctly.

To undo the helm install command use the following command:

```
helm uninstall aws-load-balancer-controller -n kube-system
```