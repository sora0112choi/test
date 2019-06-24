# Configure EKS

## Module Objectives

1. Deploy a basic EKS cluster
1. Create Worker Nodes
1. Enable worker nodes to join cluster
1. Cluster Upgrades
1. Deploy the Kubernetes Dashboard

---

## Deploy a Basic EKS Cluster

1. Create a EKS cluster.

    ```sh
    aws eks create-cluster \
      --name k8s-workshop \
      --role-arn $EKS_SERVICE_ROLE \
      --resources-vpc-config subnetIds=${EKS_SUBNET_IDS},securityGroupIds=${EKS_SECURITY_GROUPS} \
      --kubernetes-version 1.10
    ```

    It will take ~10 minutes for AWS to create a cluster for you. You can query the status of your cluster with the following command. When your cluster status is `ACTIVE`, you can proceed.

    ```sh
    aws eks describe-cluster --name k8s-workshop --query cluster.status --output text
    ```

    > Note: If your 'create cluster' fails with an error like:

    ```sh
    aws: error: argument --role-arn: expected one argument
    ```
    > Please confirm the following environment variables are set before executing the 'create cluster' command:

    ```sh
    echo $EKS_SERVICE_ROLE
    echo $EKS_SUBNET_IDS
    echo $EKS_SECURITY_GROUPS
    ```

    > If any of those environment variables are blank, please re-run the "Build Script" section of the [Cloud9 Environment Setup](./01-get-started.md#build-script).

1. Get the cluster credentials.

    ```sh
    aws eks update-kubeconfig --name k8s-workshop
    ```

    Now you can use the `kubectl` utility to connect to the cluster. For example, let's verify that the cluster is up and running by listing its services.

    ```sh
    kubectl get svc
    ```

## Create Worker Nodes

Now that your EKS master nodes are created, you can launch and configure your worker nodes.

Set the [optimised worker AMI](https://docs.aws.amazon.com/eks/latest/userguide/eks-optimized-ami.html) to use depending on selected region

For Tokyo (ap-northeast-1) Kubernetes v1.10:

```sh
export EKS_WORKER_AMI="ami-06398bdd37d76571d"
```

    > Please confirm the following environment variables are set before executing the next 'create stack' command:

    ```sh
    echo $AWS_STACK_NAME
    echo $EKS_WORKER_AMI
    echo $EKS_VPC_ID
    echo $EKS_SUBNET_IDS
    echo $EKS_SECURITY_GROUPS
    ```

If environment variables are unset, please open a new Terminal window and retry. The `AWS_STACK_NAME`, `EKS_WORKER_AMI`, `EKS_VPC_ID`, `EKS_SUBNET_IDS`, and `EKS_SECURITY_GROUPS` environment variables should have been set during the [Cloud9 Environment Setup](./01-get-started.md#build-script).

To launch your worker nodes, run the following AWS CLI command:

```sh
aws cloudformation create-stack \
  --stack-name k8s-workshop-worker-nodes \
  --template-url https://s3.amazonaws.com/aws-eks-workshop-altoros-artifacts/amazon-eks-nodegroup.yaml \
  --capabilities "CAPABILITY_IAM" \
  --parameters "[{\"ParameterKey\": \"KeyName\", \"ParameterValue\": \"${AWS_STACK_NAME}\"},
                 {\"ParameterKey\": \"NodeImageId\", \"ParameterValue\": \"${EKS_WORKER_AMI}\"},
                 {\"ParameterKey\": \"ClusterName\", \"ParameterValue\": \"k8s-workshop\"},
                 {\"ParameterKey\": \"NodeGroupName\", \"ParameterValue\": \"k8s-workshop-nodegroup\"},
                 {\"ParameterKey\": \"ClusterControlPlaneSecurityGroup\", \"ParameterValue\": \"${EKS_SECURITY_GROUPS}\"},
                 {\"ParameterKey\": \"VpcId\", \"ParameterValue\": \"${EKS_VPC_ID}\"},
                 {\"ParameterKey\": \"Subnets\", \"ParameterValue\": \"${EKS_SUBNET_IDS}\"}]"
```


Node provisioning usually takes less than 5 minutes. You can query the status of your cluster with the following command. When your cluster status is CREATE_COMPLETE, you can proceed.

```sh
aws cloudformation describe-stacks --stack-name k8s-workshop-worker-nodes --query 'Stacks[0].StackStatus' --output text
```

## Enable worker nodes to join cluster

To enable worker nodes to join your cluster, download and run the `aws-auth-cm.sh` script.

```sh
aws s3 cp s3://aws-kubernetes-artifacts/v0.5/aws-auth-cm.sh . && \
chmod +x aws-auth-cm.sh && \
./aws-auth-cm.sh
```

In another terminal watch the stream of events from the nodes and wait for them to reach the `Ready` status.
Watch the status of your nodes and wait for them to reach the `Ready` status.

```sh
kubectl get nodes --watch
```

## Cluster Upgrades

If we are running an old version it would make sense to upgrade.

There are 2 types of upgrades in EKS:

* Master upgrades (upgrades the version of Kubernetes master components)
* Node upgrades (upgrades worker nodes)

These types of upgrades should be executed separately. Let's first try to upgrade the master version.

```sh
aws eks update-cluster-version --name k8s-workshop --kubernetes-version 1.11
```

> Note: This may take a few minutes.

You can check the status of the upgrade by checking the container operation.

```sh
aws eks describe-update --name k8s-workshop \
  --update-id $(aws eks list-updates --name k8s-workshop --query 'updateIds[0]' --output text)
```

Look for the following keys in the output:

```sh
"status": "InProgress"
```

After you are done with your master upgrade you can upgrade your workers. Today, EKS does not update your Kubernetes worker nodes when you update the EKS control plane. You are responsible for updating EKS worker nodes.

To update an existing worker node group:

Determine your cluster's DNS provider.

```sh
kubectl get deployments -l k8s-app=kube-dns -n kube-system
```

If your current deployment is running fewer than 2 replicas, scale out the deployment to 2 replicas.

```sh
kubectl scale deployments/kube-dns --replicas=2 -n kube-system
```

Update your worker nodes stack with a new AMI.

```sh
aws cloudformation update-stack \
  --stack-name k8s-workshop-worker-nodes \
  --template-url https://s3.amazonaws.com/aws-eks-workshop-altoros-artifacts/amazon-eks-nodegroup.yaml \
  --capabilities "CAPABILITY_IAM" \
  --parameters "[{\"ParameterKey\": \"KeyName\", \"ParameterValue\": \"${AWS_STACK_NAME}\"},
                 {\"ParameterKey\": \"NodeImageId\", \"ParameterValue\": \"$(aws ec2 describe-images --filters 'Name=name,Values=amazon-eks-node-1.11-v20190109' 'Name=state,Values=available' --output json | jq -r '.Images | sort_by(.CreationDate) | last(.[]).ImageId')\"},
                 {\"ParameterKey\": \"ClusterName\", \"ParameterValue\": \"k8s-workshop\"},
                 {\"ParameterKey\": \"NodeGroupName\", \"ParameterValue\": \"k8s-workshop-nodegroup\"},
                 {\"ParameterKey\": \"ClusterControlPlaneSecurityGroup\", \"ParameterValue\": \"${EKS_SECURITY_GROUPS}\"},
                 {\"ParameterKey\": \"VpcId\", \"ParameterValue\": \"${EKS_VPC_ID}\"},
                 {\"ParameterKey\": \"Subnets\", \"ParameterValue\": \"${EKS_SUBNET_IDS}\"}]"
```

> Note: This may take a few minutes.

```sh
watch kubectl get nodes
```

Scale `kube-dns` deployment to 1 replica.

```sh
kubectl scale deployments/kube-dns --replicas=1 -n kube-system
```

---

Next: [Pods and Services](04-pods-and-services.md)
