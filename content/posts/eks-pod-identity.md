---
title: "EKS Pod Identity"
slug: eks-pod-identity
summary: "A Practical Guide to Amazon EKS Pod Identity"
date: 2025-06-10T11:04:53+05:30
---

**TL;DR** – EKS Pod Identity (PI) lets you map an IAM Role to a Kubernetes **ServiceAccount** with a **single API call**, no OIDC-provider wrangling, and zero manifest annotations. It’s the successor to IRSA and is **GA** for clusters ≥ v1.27 (platform `eks.8` or later).

## Advantages of using EKS Pod Identity
- **Simpler than IRSA** – no OIDC provider per-cluster and no trust-policy edits each time you add a cluster.
- **Reusable roles** – one role can be associated with SAs across many clusters.
- **Lower STS load** – credentials are fetched per-node by the **EKS Pod Identity Agent** and shared with pods on that node.
- **Cleaner separation of duties** – IAM admins manage roles, cluster operators manage associations.


## Prerequisites

**EKS Cluster**: Kubernetes 1.27 / platform `eks.8`

### Install Amazon EKS Pod Identity Agent (once per cluster)

Ensure that **Amazon EKS Pod Identity Agent** Add On is installed in the cluster. If the add on is not already install, you can install it using AWS CLI or AWS Console.

To install **Amazon EKS Pod Identity Agent** Add On using AWS Console, go to the EKS Dashboard in AWS Console, click on the EKS cluster and go to the "Add-ons" tab. Click on "Get  more add-ons". Then use the search option to find **Amazon EKS Pod Identity Agent**. It should come under **AWS add-ons**. Select the Add on click **Next**. In the next window, ensure the add-on version is selected to latest. Then click on **Next**. In next window, click on **Create** to deploy this add-on. After some time, the status of the add-on will change to `Active`.

To install **Amazon EKS Pod Identity Agent** Add On using AWS CLI, run:
```bash
aws --region $REGION eks create-addon --cluster-name $CLUSTER --addon-name eks-pod-identity-agent
```

Replace `$REGION` and `$CLUSTER` with appropriate values.

## Using EKS Pod Identity

Once the **Amazon EKS Pod Identity Agent** Add On installed in the cluster, we can start using IAM role directly with pods deployed in our EKS cluster. Just like EC2 instance are able to access AWS services once they are assigned an IAM role.

### Create an IAM role
Just adding the add on is not enough. We also need to ensure that service account associated with our workload has an access entry in the EKS cluster. To do so, we would need an IAM role with appropriate permissions and following trust policy:

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "Service": "pods.eks.amazonaws.com"
            },
            "Action": [
                "sts:AssumeRole",
                "sts:TagSession"
            ]
        }
    ]
}
```

### Create a Pod Identity Association

```bash
aws --region $REGION eks create-pod-identity-association --cluster-name $CLUSTER --namespace $NAMESPACE --service-account $SERVICEACCOUNT --role-arn $ARN
```

replace `$REGION`, `$CLUSTER`, `$NAMESPACE`, `$SERVICEACCOUNT` and `$ARN` with appropriate values.

### Create the Deployment and ServiceAccount

Ensure the ServiceAccount:
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: $SERVICEACCOUNT
  namespace: $NAMESPACE
```

Use this service account in the deployment:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: awscli
  namespace: $NAMESPACE
spec:
  replicas: 1
  selector:
    matchLabels:
      app: awscli
  template:
    metadata:
      labels:
        app: awscli
    spec:
      serviceAccountName: $SERVICEACCOUNT
      containers:
        - name: awscli
          image: amazonlinux:2023
          command: ["sleep", "infinity"]
```

Apply these manifest.
```bash
# kubectl apply -f deployment.yaml
deployment.apps/awscli created
#
```

If this exact manifest was used, then we would also need to install aws cli in it. Exec into the container, install aws cli, and then check by running any aws command:
```bash
bash-5.2# /usr/local/bin/aws --version
aws-cli/2.27.34 Python/3.13.3 Linux/6.12.25 exe/x86_64.amzn.2023
bash-5.2# aws sts get-caller-identity
{
    "UserId": "<userid-replaced>",
    "Account": "<account-id-replaced>",
    "Arn": "<arn-replaced>"
}
bash-5.2#
```

## Wrap-up
EKS Pod Identity reduces IRSA overhead to three steps: install add-on, create IAM role and associate it with a service account. Give it a spin, measure the STS traffic drop, and enjoy cleaner RBAC between Kubernetes and AWS.

## Further reading
- [Pod Identities overview](https://docs.aws.amazon.com/eks/latest/userguide/pod-identities.html)
- [AWS CLI `create-pod-identity-association`](https://docs.aws.amazon.com/cli/latest/reference/eks/create-pod-identity-association.html)
- [AWS Blog: Amazon EKS Pod Identity: a new way for applications on EKS to obtain IAM credentials](https://aws.amazon.com/blogs/containers/amazon-eks-pod-identity-a-new-way-for-applications-on-eks-to-obtain-iam-credentials/)

Until next time. Peace :v:!
