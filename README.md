# karpenter-installation

This guide assumes that you have aws-cli installed, an already existing cluster and kubectl installed. 

1.  Set a variable for your cluster name.
```
KARPENTER_NAMESPACE=kube-system #you can write any namespace of your choice
CLUSTER_NAME=<your cluster name>
#Please replace <your cluster name> with the actual name of your cluster
```

2.  Set other variables from your cluster configuration.
```
AWS_PARTITION="aws" # if you are not using standard partitions, you may need to configure to aws-cn / aws-us-gov
AWS_REGION="$(aws configure list | grep region | tr -s " " | cut -d" " -f3)"
OIDC_ENDPOINT="$(aws eks describe-cluster --name "${CLUSTER_NAME}" \
    --query "cluster.identity.oidc.issuer" --output text)"
AWS_ACCOUNT_ID=$(aws sts get-caller-identity --query 'Account' \
    --output text)
K8S_VERSION=1.28
ARM_AMI_ID="$(aws ssm get-parameter --name /aws/service/eks/optimized-ami/${K8S_VERSION}/amazon-linux-2-arm64/recommended/image_id --query Parameter.Value --output text)"
AMD_AMI_ID="$(aws ssm get-parameter --name /aws/service/eks/optimized-ami/${K8S_VERSION}/amazon-linux-2/recommended/image_id --query Parameter.Value --output text)"
GPU_AMI_ID="$(aws ssm get-parameter --name /aws/service/eks/optimized-ami/${K8S_VERSION}/amazon-linux-2-gpu/recommended/image_id --query Parameter.Value --output text)"
```

3.  Create IAM roles
To get started with our migration we first need to create two new IAM roles for nodes provisioned with Karpenter and the Karpenter controller.

To create the Karpenter node role we will use the following policy and commands.
```
echo '{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "Service": "ec2.amazonaws.com"
            },
            "Action": "sts:AssumeRole"
        }
    ]
}' > node-trust-policy.json

aws iam create-role --role-name "KarpenterNodeRole-${CLUSTER_NAME}" \
    --assume-role-policy-document file://node-trust-policy.json
```

4.  Now attach the required policies to the role
```
aws iam attach-role-policy --role-name "KarpenterNodeRole-${CLUSTER_NAME}" \
    --policy-arn "arn:${AWS_PARTITION}:iam::aws:policy/AmazonEKSWorkerNodePolicy"

aws iam attach-role-policy --role-name "KarpenterNodeRole-${CLUSTER_NAME}" \
    --policy-arn "arn:${AWS_PARTITION}:iam::aws:policy/AmazonEKS_CNI_Policy"

aws iam attach-role-policy --role-name "KarpenterNodeRole-${CLUSTER_NAME}" \
    --policy-arn "arn:${AWS_PARTITION}:iam::aws:policy/AmazonEC2ContainerRegistryReadOnly"

aws iam attach-role-policy --role-name "KarpenterNodeRole-${CLUSTER_NAME}" \
    --policy-arn "arn:${AWS_PARTITION}:iam::aws:policy/AmazonSSMManagedInstanceCore"
```

5.  create an IAM role that the Karpenter controller will use to provision new instances. The controller will be using IAM Roles for Service Accounts (IRSA) which requires an OIDC endpoint.
```
cat << EOF > controller-trust-policy.json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "Federated": "arn:${AWS_PARTITION}:iam::${AWS_ACCOUNT_ID}:oidc-provider/${OIDC_ENDPOINT#*//}"
            },
            "Action": "sts:AssumeRoleWithWebIdentity",
            "Condition": {
                "StringEquals": {
                    "${OIDC_ENDPOINT#*//}:aud": "sts.amazonaws.com",
                    "${OIDC_ENDPOINT#*//}:sub": "system:serviceaccount:${KARPENTER_NAMESPACE}:karpenter"
                }
            }
        }
    ]
}
EOF

aws iam create-role --role-name "KarpenterControllerRole-${CLUSTER_NAME}" \
    --assume-role-policy-document file://controller-trust-policy.json

cat << EOF > controller-policy.json
{
    "Statement": [
        {
            "Action": [
                "ssm:GetParameter",
                "ec2:DescribeImages",
                "ec2:RunInstances",
                "ec2:DescribeSubnets",
                "ec2:DescribeSecurityGroups",
                "ec2:DescribeLaunchTemplates",
                "ec2:DescribeInstances",
                "ec2:DescribeInstanceTypes",
                "ec2:DescribeInstanceTypeOfferings",
                "ec2:DeleteLaunchTemplate",
                "ec2:CreateTags",
                "ec2:CreateLaunchTemplate",
                "ec2:CreateFleet",
                "ec2:DescribeSpotPriceHistory",
                "pricing:GetProducts"
            ],
            "Effect": "Allow",
            "Resource": "*",
            "Sid": "Karpenter"
        },
        {
            "Action": "ec2:TerminateInstances",
            "Condition": {
                "StringLike": {
                    "ec2:ResourceTag/karpenter.sh/nodepool": "*"
                }
            },
            "Effect": "Allow",
            "Resource": "*",
            "Sid": "ConditionalEC2Termination"
        },
        {
            "Effect": "Allow",
            "Action": "iam:PassRole",
            "Resource": "arn:${AWS_PARTITION}:iam::${AWS_ACCOUNT_ID}:role/KarpenterNodeRole-${CLUSTER_NAME}",
            "Sid": "PassNodeIAMRole"
        },
        {
            "Effect": "Allow",
            "Action": "eks:DescribeCluster",
            "Resource": "arn:${AWS_PARTITION}:eks:${AWS_REGION}:${AWS_ACCOUNT_ID}:cluster/${CLUSTER_NAME}",
            "Sid": "EKSClusterEndpointLookup"
        },
        {
            "Sid": "AllowScopedInstanceProfileCreationActions",
            "Effect": "Allow",
            "Resource": "*",
            "Action": [
            "iam:CreateInstanceProfile"
            ],
            "Condition": {
            "StringEquals": {
                "aws:RequestTag/kubernetes.io/cluster/${CLUSTER_NAME}": "owned",
                "aws:RequestTag/topology.kubernetes.io/region": "${AWS_REGION}"
            },
            "StringLike": {
                "aws:RequestTag/karpenter.k8s.aws/ec2nodeclass": "*"
            }
            }
        },
        {
            "Sid": "AllowScopedInstanceProfileTagActions",
            "Effect": "Allow",
            "Resource": "*",
            "Action": [
            "iam:TagInstanceProfile"
            ],
            "Condition": {
            "StringEquals": {
                "aws:ResourceTag/kubernetes.io/cluster/${CLUSTER_NAME}": "owned",
                "aws:ResourceTag/topology.kubernetes.io/region": "${AWS_REGION}",
                "aws:RequestTag/kubernetes.io/cluster/${CLUSTER_NAME}": "owned",
                "aws:RequestTag/topology.kubernetes.io/region": "${AWS_REGION}"
            },
            "StringLike": {
                "aws:ResourceTag/karpenter.k8s.aws/ec2nodeclass": "*",
                "aws:RequestTag/karpenter.k8s.aws/ec2nodeclass": "*"
            }
            }
        },
        {
            "Sid": "AllowScopedInstanceProfileActions",
            "Effect": "Allow",
            "Resource": "*",
            "Action": [
            "iam:AddRoleToInstanceProfile",
            "iam:RemoveRoleFromInstanceProfile",
            "iam:DeleteInstanceProfile"
            ],
            "Condition": {
            "StringEquals": {
                "aws:ResourceTag/kubernetes.io/cluster/${CLUSTER_NAME}": "owned",
                "aws:ResourceTag/topology.kubernetes.io/region": "${AWS_REGION}"
            },
            "StringLike": {
                "aws:ResourceTag/karpenter.k8s.aws/ec2nodeclass": "*"
            }
            }
        },
        {
            "Sid": "AllowInstanceProfileReadActions",
            "Effect": "Allow",
            "Resource": "*",
            "Action": "iam:GetInstanceProfile"
        }
    ],
    "Version": "2012-10-17"
}
EOF

aws iam put-role-policy --role-name "KarpenterControllerRole-${CLUSTER_NAME}" \
    --policy-name "KarpenterControllerPolicy-${CLUSTER_NAME}" \
    --policy-document file://controller-policy.json
```

6.  Add tags to subnets and security groups. 
We need to add tags to our nodegroup subnets so Karpenter will know which subnets to use.
```
for NODEGROUP in $(aws eks list-nodegroups --cluster-name "${CLUSTER_NAME}" --query 'nodegroups' --output text); do
    aws ec2 create-tags \
        --tags "Key=karpenter.sh/discovery,Value=${CLUSTER_NAME}" \
        --resources $(aws eks describe-nodegroup --cluster-name "${CLUSTER_NAME}" \
        --nodegroup-name "${NODEGROUP}" --query 'nodegroup.subnets' --output text )
done
```

7.  Add tags to our security groups. This command only tags the security groups for the first nodegroup in the cluster
```
NODEGROUP=$(aws eks list-nodegroups --cluster-name "${CLUSTER_NAME}" \
    --query 'nodegroups[0]' --output text)

LAUNCH_TEMPLATE=$(aws eks describe-nodegroup --cluster-name "${CLUSTER_NAME}" \
    --nodegroup-name "${NODEGROUP}" --query 'nodegroup.launchTemplate.{id:id,version:version}' \
    --output text | tr -s "\t" ",")

# If your EKS setup is configured to use only Cluster security group, then please execute -

SECURITY_GROUPS=$(aws eks describe-cluster \
    --name "${CLUSTER_NAME}" --query "cluster.resourcesVpcConfig.clusterSecurityGroupId" --output text)

aws ec2 create-tags \
    --tags "Key=karpenter.sh/discovery,Value=${CLUSTER_NAME}" \
    --resources "${SECURITY_GROUPS}"
```

8.  Update aws-auth ConfigMap
We need to allow nodes that are using the node IAM role we just created to join the cluster. To do that we have to modify the aws-auth ConfigMap in the cluster.
```
kubectl edit configmap aws-auth -n kube-system
```
You will need to add a section to the mapRoles that looks something like this. Replace the ${AWS_PARTITION} variable with the account partition, ${AWS_ACCOUNT_ID} variable with your account ID, and ${CLUSTER_NAME} variable with the cluster name, but do not replace the {{EC2PrivateDNSName}}

```
- groups:
  - system:bootstrappers
  - system:nodes
  rolearn: arn:${AWS_PARTITION}:iam::${AWS_ACCOUNT_ID}:role/KarpenterNodeRole-${CLUSTER_NAME}
  username: system:node:{{EC2PrivateDNSName}}
#replace ${AWS_PARTITION}, ${AWS_ACCOUNT_ID} and ${CLUSTER_NAME} with the actual value.
#The full aws-auth configmap should have two groups. One for your Karpenter node role and one for your existing node group.
```

9.  Deploy Karpenter.
First set the Karpenter release you want to deploy.

```
export KARPENTER_VERSION="1.1.1"
```

We can now generate a full Karpenter deployment yaml from the Helm chart.

```
helm template karpenter oci://public.ecr.aws/karpenter/karpenter --version "${KARPENTER_VERSION}" --namespace "${KARPENTER_NAMESPACE}" \
    --set "settings.clusterName=${CLUSTER_NAME}" \
    --set "settings.interruptionQueue=${CLUSTER_NAME}" \
    --set "serviceAccount.annotations.eks\.amazonaws\.com/role-arn=arn:${AWS_PARTITION}:iam::${AWS_ACCOUNT_ID}:role/KarpenterControllerRole-${CLUSTER_NAME}" \
    --set controller.resources.requests.cpu=1 \
    --set controller.resources.requests.memory=1Gi \
    --set controller.resources.limits.cpu=1 \
    --set controller.resources.limits.memory=1Gi > karpenter.yaml
```

10.  Set node affinity
Edit the karpenter.yaml file and find the karpenter deployment affinity rules. Modify the affinity so karpenter will run on one of the existing node group nodes.

The rules should look something like this. Modify the value to match your $NODEGROUP, one node group per line.

```
affinity:
  nodeAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      nodeSelectorTerms:
      - matchExpressions:
        - key: karpenter.sh/nodepool
          operator: DoesNotExist
        - key: eks.amazonaws.com/nodegroup
          operator: In
          values:
          - ${NODEGROUP}
  podAntiAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      - topologyKey: "kubernetes.io/hostname"

change the ${nodegroup} to the actual name of our EKS node group. you can check the aws console to get the node group name.
```

11. Now that our deployment is ready we can create the karpenter namespace

create the NodePool CRD, and then deploy the rest of the karpenter resources.

```
kubectl create namespace "${KARPENTER_NAMESPACE}" || true
kubectl create -f \
    "https://raw.githubusercontent.com/aws/karpenter-provider-aws/v${KARPENTER_VERSION}/pkg/apis/crds/karpenter.sh_nodepools.yaml"
kubectl create -f \
    "https://raw.githubusercontent.com/aws/karpenter-provider-aws/v${KARPENTER_VERSION}/pkg/apis/crds/karpenter.k8s.aws_ec2nodeclasses.yaml"
kubectl create -f \
    "https://raw.githubusercontent.com/aws/karpenter-provider-aws/v${KARPENTER_VERSION}/pkg/apis/crds/karpenter.sh_nodeclaims.yaml"
kubectl apply -f karpenter.yaml
```

12.  Create default NodePool 
We need to create a default NodePool so Karpenter knows what types of nodes we want for unscheduled workloads

```
cat <<EOF | envsubst | kubectl apply -f -
apiVersion: karpenter.sh/v1
kind: NodePool
metadata:
  name: default
spec:
  template:
    spec:
      requirements:
        - key: kubernetes.io/arch
          operator: In
          values: ["amd64"]
        - key: kubernetes.io/os
          operator: In
          values: ["linux"]
        - key: karpenter.sh/capacity-type
          operator: In
          values: ["on-demand"]
        - key: karpenter.k8s.aws/instance-category
          operator: In
          values: ["t"]
        - key: karpenter.k8s.aws/instance-generation
          operator: Lt
          values: ["3"]
      nodeClassRef:
        group: karpenter.k8s.aws
        kind: EC2NodeClass
        name: default
      expireAfter: 720h # 30 * 24h = 720h
  limits:
    cpu: 1000
  disruption:
    consolidationPolicy: WhenEmptyOrUnderutilized
    consolidateAfter: 1m
---
apiVersion: karpenter.k8s.aws/v1
kind: EC2NodeClass
metadata:
  name: default
spec:
  amiFamily: AL2 # Amazon Linux 2
  role: "KarpenterNodeRole-${CLUSTER_NAME}" # replace with your cluster name
  subnetSelectorTerms:
    - tags:
        karpenter.sh/discovery: "${CLUSTER_NAME}" # replace with your cluster name
  securityGroupSelectorTerms:
    - tags:
        karpenter.sh/discovery: "${CLUSTER_NAME}" # replace with your cluster name
  amiSelectorTerms:
    - id: "${ARM_AMI_ID}"
    - id: "${AMD_AMI_ID}"
#   - id: "${GPU_AMI_ID}" # <- GPU Optimized AMD AMI 
#   - name: "amazon-eks-node-${K8S_VERSION}-*" # <- automatically upgrade when a new AL2 EKS Optimized AMI is released. This is unsafe for production workloads. Validate AMIs in lower environments before deploying them to production.
EOF

# The above will set create a nodepool and ec2nodeclass , the nodepool is designed to handle "t" types of instances and the generation is less than 3.
```

13.  lets deploy inflate app.

the inflate app is to test if karpenter is working. this will create 20 pods and karpenter will spin more nodes and schedule the pods on the node.

```
#create a file for this and run the comand kubectl apply -f <file-name>
apiVersion: apps/v1
kind: Deployment
metadata:
  name: inflate
spec:
  replicas: 20
  selector:
    matchLabels:
      app: inflate
  template:
    metadata:
      labels:
        app: inflate
    spec:
      terminationGracePeriodSeconds: 0
      securityContext:
        runAsUser: 1000
        runAsGroup: 3000
        fsGroup: 2000
      containers:
      - name: inflate
        image: public.ecr.aws/eks-distro/kubernetes/pause:3.7
        resources:
          requests:
            cpu: 1
        securityContext:
          allowPrivilegeEscalation: false

```



