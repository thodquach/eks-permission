# eks-permission

How to add permission to an IAM user to access all resources on a specific EKS cluster

**Step 1. Create an IAM policy attached to it that includes the `eks:AccessKubernetesApi` action.**

In `IAM Management Console => Policies => Create Policy`.
Below JSON is an example configuration of an policy in JSON format that allows a user to [View nodes](https://docs.aws.amazon.com/eks/latest/userguide/view-nodes.html) and [View workloads](https://docs.aws.amazon.com/eks/latest/userguide/view-workloads.html) for all clusters.
```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "eks:DescribeNodegroup",
                "eks:ListNodegroups",
                "eks:DescribeCluster",
                "eks:ListClusters",
                "eks:AccessKubernetesApi",
                "ssm:GetParameter",
                "eks:ListUpdates",
                "eks:ListFargateProfiles",
                "eks:AccessKubernetesApi"
            ],
            "Resource": "*"
        }
    ]
}
```
I have named above policy, `EKS-permission`

**Step 2. Attach `EKS-permission` policy to your user in AWS Console**

You need IAM permissions to access all Kubernetes resources, that why we attach the policy into your user.
In ```IAM Management Console => Users => Your user => Add Permissions => Attach existing policies directly```
Find and add `EKS-permission` policy to your user.

**Step 3.  Create `ClusterRole` and `ClusterRoleBinding` to access all resources**

For now, when you use `kubectl` command to get resource from EKS cluster, you might meet this problem: `Unauthorized: Verify you have access to the Kubernetes cluster`. The reason is the Kubernetes user or group that the IAM account or role is mapped to in the configmap must be a subject in a rolebinding or clusterrolebinding that is bound to a Kubernetes role or clusterrole that has the necessary permissions to view the Kubernetes resources.
You can download the following example manifests that create a  `clusterrole`  and  `clusterrolebinding`  or a  `role`  and  `rolebinding`:

-   **View Kubernetes resources in all namespaces**  – The group name in the file is  `eks-console-dashboard-full-access-group`, which is the group that your IAM user or role needs to be mapped to in the  `aws-auth`  configmap. You can change the name of the group before applying it to your cluster, if desired, and then map your IAM user or role to that group in the configmap. Download the file from:
    
    `https://s3.us-west-2.amazonaws.com/amazon-eks/docs/eks-console-full-access.yaml`
    
-   **View Kubernetes resources in a specific namespace**  – The namespace in this file is  `default`, so if you want to specify a different namespace, edit the file before applying it to your cluster. The group name in the file is  `eks-console-dashboard-restricted-access-group`, which is the group that your IAM user or role needs to be mapped to in the  `aws-auth`  configmap. You can change the name of the group before applying it to your cluster, if desired, and then map your IAM user or role to that group in the configmap. Download the file from:
    
    `https://s3.us-west-2.amazonaws.com/amazon-eks/docs/eks-console-restricted-access.yaml`

And you need to `kubectl apply` above yaml file.

**Step 4. Map your IAM user to above `role` you have created**

Edit configmap `aws-auth`( `kubectl edit configmap -n kube-system aws-auth`)

Fill your IAM user 's information like below snipcode.
```
apiVersion: v1
kind: ConfigMap
metadata:
  name: aws-auth
  namespace: kube-system
data:
  mapRoles: |
    - rolearn: <ARN of instance role (not instance profile)>
      username: system:node:{{EC2PrivateDNSName}}
      groups:
        - system:bootstrappers
        - system:nodes
```
In my case, I just need to add `mapUsers` below configuration:
```
mapUsers: |
  - userarn: arn:aws:iam::<project_id>:user/tho.qd@tradingfoe.com
    username: tho.qd@example.com
    groups:
    - eks-console-dashboard-full-access-group
```

**Step 5. Verify your permission, get kubernetes resources**

You need set your AWS identify first:
```
export ACCESS_KEY_ID=xxx
export SECRET_ACCESS_KEY=xxx
aws --profile default configure set aws_secret_access_key $SECRET_ACCESS_KEY
aws --profile default configure set aws_access_key_id $ACCESS_KEY_ID
```
Check who you are:
```
aws sts get-caller-identity
```
Final, get kubernetes resources:
```
kubectl -n kube-system get all
```


Reference link: [Can't see workloads or nodes and receive an error in the AWS Management Console](https://docs.aws.amazon.com/eks/latest/userguide/troubleshooting_iam.html#security-iam-troubleshoot-cannot-view-nodes-or-workloads)
