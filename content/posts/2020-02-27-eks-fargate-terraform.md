---
title: "Running a Kubernetes cluster on EKS with Fargate and Terraform"
date: 2020-02-27T09:00:00+01:00
draft: false
summary: A simple example on how to run a dockerized application in a Kubernetes cluster with the help of AWS EKS and Fargate, all defined in Terraform
author: Andreas Jantschnig
author_position: Engineering Lead
author_email: andreas.jantschnig@finleap.com
---

As described in my previous post (which you can find [here](https://engineering.finleap.com/posts/2020-02-20-ecs-fargate-terraform/)), I recently started exploring the possibilities of IaC. Upon finishing my ECS setup, it was time to try the same thing with a system that seems to be one of the most widely used container management systems: Terraform.

The target setup should be the same as in the previous ECS example, meaning the pods should run in a private subnet, communicating with the outside world via a load balancer placed in the public subnet.

TLDR; the resulting template repo can be found here: https://github.com/finleap/tf-eks-fargate-tmpl

## Step 1 - VPC
This step I won't describe much here, as it is actually pretty much same setup as in my previous ECS example, which you can find [here](https://engineering.finleap.com/posts/2020-02-20-ecs-fargate-terraform/#vpc).

The only difference when working with EKS is that we have to add special tags to the subnets in order for Kubernetes to know what the subnets should be used for:

- `kubernetes.io/cluster/{clustername}: shared` tag needs to be added to all subnets that the cluster should be able to use
- `kubernetes.io/role/elb: 1` tag needs to be added to the public subnets so that Kubernetes knows to use only these subnets for public loadbalancers
- `kubernetes.io/role/internal-elb: 1` tag needs to be added to the private subnets, so that Kubernetes knows to use these subnets for internal loadbalancers

You can read more about the Cluster VPC requirements [here](https://docs.aws.amazon.com/eks/latest/userguide/network_reqs.html), the full Terraform definition of the VPC can be found [here](https://github.com/Finleap/tf-eks-fargate-tmpl/blob/master/vpc/main.tf).


## Step 2 - The EKS cluster
To create a cluster within EKS, the following setup is necessary with Terraform:

{{< highlight hcl >}}
resource "aws_eks_cluster" "main" {
  name     = "${var.name}-${var.environment}"
  role_arn = aws_iam_role.eks_cluster_role.arn

  enabled_cluster_log_types = ["api", "audit", "authenticator", "controllerManager", "scheduler"]

  vpc_config {
    subnet_ids = concat(var.public_subnets.*.id, var.private_subnets.*.id)
  }

  timeouts {
    delete = "30m"
  }

  depends_on = [
    aws_cloudwatch_log_group.eks_cluster,
    aws_iam_role_policy_attachment.AmazonEKSClusterPolicy,
    aws_iam_role_policy_attachment.AmazonEKSServicePolicy
  ]
}
{{< / highlight >}}

We are still using the `aws` provider to create the cluster, but for further Kubernetes specific resources, we also need to add a `kubernetes` provider like this:

{{< highlight hcl >}}
provider "kubernetes" {
  host                   = data.aws_eks_cluster.cluster.endpoint
  cluster_ca_certificate = base64decode(data.aws_eks_cluster.cluster.certificate_authority[0].data)
  token                  = data.aws_eks_cluster_auth.cluster.token
  load_config_file       = false
  version                = "~> 1.10"
}

data "aws_eks_cluster" "cluster" {
  name = aws_eks_cluster.main.id
}

data "aws_eks_cluster_auth" "cluster" {
  name = aws_eks_cluster.main.id
}
{{< / highlight >}}

The `data` fields in the above setup will read the necessary data for initializing the `kubernetes` provider after the cluster was created via the `aws` provider.

As you can see, we also need to attach a role to the cluster, which will give it the necessary permission for interacting with the nodes. The setup looks as follows:

{{< highlight hcl >}}
resource "aws_iam_policy" "AmazonEKSClusterCloudWatchMetricsPolicy" {
  name   = "AmazonEKSClusterCloudWatchMetricsPolicy"
  policy = <<EOF
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Action": [
                "cloudwatch:PutMetricData"
            ],
            "Resource": "*",
            "Effect": "Allow"
        }
    ]
}
EOF
}

resource "aws_iam_policy" "AmazonEKSClusterNLBPolicy" {
  name   = "AmazonEKSClusterNLBPolicy"
  policy = <<EOF
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Action": [
                "elasticloadbalancing:*",
                "ec2:CreateSecurityGroup",
                "ec2:Describe*"
            ],
            "Resource": "*",
            "Effect": "Allow"
        }
    ]
}
EOF
}

resource "aws_iam_role" "eks_cluster_role" {
  name                  = "${var.name}-eks-cluster-role"
  force_detach_policies = true

  assume_role_policy = <<POLICY
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": [
          "eks.amazonaws.com",
          "eks-fargate-pods.amazonaws.com"
          ]
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
POLICY
}

resource "aws_iam_role_policy_attachment" "AmazonEKSClusterPolicy" {
  policy_arn = "arn:aws:iam::aws:policy/AmazonEKSClusterPolicy"
  role       = aws_iam_role.eks_cluster_role.name
}

resource "aws_iam_role_policy_attachment" "AmazonEKSServicePolicy" {
  policy_arn = "arn:aws:iam::aws:policy/AmazonEKSServicePolicy"
  role       = aws_iam_role.eks_cluster_role.name
}

resource "aws_iam_role_policy_attachment" "AmazonEKSCloudWatchMetricsPolicy" {
  policy_arn = aws_iam_policy.AmazonEKSClusterCloudWatchMetricsPolicy.arn
  role       = aws_iam_role.eks_cluster_role.name
}

resource "aws_iam_role_policy_attachment" "AmazonEKSCluserNLBPolicy" {
  policy_arn = aws_iam_policy.AmazonEKSClusterNLBPolicy.arn
  role       = aws_iam_role.eks_cluster_role.name
}
{{< / highlight >}}

And as we now already have attached the necessary policies for adding CloudWatch metrics, let's also add a log group:

{{< highlight hcl >}}
resource "aws_cloudwatch_log_group" "eks_cluster" {
  name              = "/aws/eks/${var.name}-${var.environment}/cluster"
  retention_in_days = 30

  tags = {
    Name        = "${var.name}-${var.environment}-eks-cloudwatch-log-group"
    Environment = var.environment
  }
}
{{< / highlight >}}

### Node groups
Now we have the cluster setup and ready, but we don't have any nodes yet to run our pods on. It is actually possible to run the entire cluster fully on Fargate, but it requires some tweaking of the CoreDNS deployment (more on that [here](https://docs.aws.amazon.com/eks/latest/userguide/fargate-getting-started.html#fargate-gs-coredns) and [here](https://github.com/terraform-providers/terraform-provider-aws/issues/11327)).

So instead, we create a node group for the `kube-system` namespace, which is used to run any pods necessary for operating the Kubernetes cluster.

{{< highlight hcl >}}
resource "aws_eks_node_group" "main" {
  cluster_name    = aws_eks_cluster.main.name
  node_group_name = "kube-system"
  node_role_arn   = aws_iam_role.eks_node_group_role.arn
  subnet_ids      = var.private_subnets.*.id

  scaling_config {
    desired_size = 4
    max_size     = 4
    min_size     = 4
  }

  instance_types  = ["t2.micro"]

  tags = {
    Name        = "${var.name}-${var.environment}-eks-node-group"
    Environment = var.environment
  }

  # Ensure that IAM Role permissions are created before and deleted after EKS Node Group handling.
  # Otherwise, EKS will not be able to properly delete EC2 Instances and Elastic Network Interfaces.
  depends_on = [
    aws_iam_role_policy_attachment.AmazonEKSWorkerNodePolicy,
    aws_iam_role_policy_attachment.AmazonEKS_CNI_Policy,
    aws_iam_role_policy_attachment.AmazonEC2ContainerRegistryReadOnly,
  ]
}
{{< / highlight >}}

You can see that `desired_size` is set to `4`, which is because I chose to use `t2.micro` instances for this demo as they are free-tier eligible, but they also only provide max. 2 IP instances, which means you can only run 2 pods on each instance.

Again, the node group requires an attached role in order to communicate with the pods running on it, which is setup as follows:

{{< highlight hcl >}}
resource "aws_iam_role" "eks_node_group_role" {
  name                  = "${var.name}-eks-node-group-role"
  force_detach_policies = true

  assume_role_policy = <<POLICY
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": [
          "ec2.amazonaws.com"
          ]
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
POLICY
}

resource "aws_iam_role_policy_attachment" "AmazonEKSWorkerNodePolicy" {
  policy_arn = "arn:aws:iam::aws:policy/AmazonEKSWorkerNodePolicy"
  role       = aws_iam_role.eks_node_group_role.name
}

resource "aws_iam_role_policy_attachment" "AmazonEKS_CNI_Policy" {
  policy_arn = "arn:aws:iam::aws:policy/AmazonEKS_CNI_Policy"
  role       = aws_iam_role.eks_node_group_role.name
}

resource "aws_iam_role_policy_attachment" "AmazonEC2ContainerRegistryReadOnly" {
  policy_arn = "arn:aws:iam::aws:policy/AmazonEC2ContainerRegistryReadOnly"
  role       = aws_iam_role.eks_node_group_role.name
}
{{< / highlight >}}

### kubeconfig
Now the cluster is ready to be used for deploying some pods to it. But one thing is missing for managing and monitoring the cluster: We need to enable `kubectl` to connect to this cluster, and for this we need to write a `kube-config` file. In order to do this, we first need two new providers in our Terraform setup:

{{< highlight hcl >}}
provider "local" {
  version = "~> 1.4"
}

provider "template" {
  version = "~> 2.1"
}
{{< / highlight >}}

The `template` provider lets us use a template file and fill in the needed values to create a valid `kubeconfig` file, while the `local` provider enables us to write this file on our local disk.

The used template file looks like this (and can also be found [here](https://github.com/Finleap/tf-eks-fargate-tmpl/blob/master/eks/templates/kubeconfig.tpl)):

{{< highlight yaml >}}
apiVersion: v1
preferences: {}
kind: Config

clusters:
- cluster:
    server: ${endpoint}
    certificate-authority-data: ${cluster_auth_base64}
  name: ${kubeconfig_name}

contexts:
- context:
    cluster: ${kubeconfig_name}
    user: ${kubeconfig_name}
  name: ${kubeconfig_name}

current-context: ${kubeconfig_name}

users:
- name: ${kubeconfig_name}
  user:
    exec:
      apiVersion: client.authentication.Kubernetes.io/v1alpha1
      command: aws-iam-authenticator
      args:
        - "token"
        - "-i"
        - "${clustername}"
{{< / highlight >}}

With the following resources, we fill the template file with the values of our newly created cluster and write it to the filesystem for `kubectl` to use it:

{{< highlight hcl >}}
data "template_file" "kubeconfig" {
  template = file("${path.module}/templates/kubeconfig.tpl")

  vars = {
    kubeconfig_name           = "eks_${aws_eks_cluster.main.name}"
    clustername               = aws_eks_cluster.main.name
    endpoint                  = data.aws_eks_cluster.cluster.endpoint
    cluster_auth_base64       = data.aws_eks_cluster.cluster.certificate_authority[0].data
  }
}

resource "local_file" "kubeconfig" {
  content  = data.template_file.kubeconfig.rendered
  filename = pathexpand("~/.kube/config")
}
{{< / highlight >}}

> Note that `~/.kube/config` is the default path where `kubectl` looks for a configuration file. For more fine granular information about how to use `kubeconfig` files you can have a look [here](https://kubernetes.io/docs/concepts/configuration/organize-cluster-access-kubeconfig/).

## Step 3 - deploying a container to the cluster and running it on Fargate
The following section is basically the terraform-ed version of [this](https://docs.aws.amazon.com/eks/latest/userguide/alb-ingress.html) example on how to deploy a simple webapp to an EKS cluster and running it on Fargate while exposing it to the outside world with the help of an ingress controller.

### OIDC provider
First we need to add another resource to the cluster in order for the inress controller to be able to route traffic to the pods running the demo application, an OIDC provider:

{{< highlight hcl >}}
resource "aws_iam_openid_connect_provider" "main" {
  client_id_list  = ["sts.amazonaws.com"]
  thumbprint_list = [data.external.thumbprint.result.thumbprint]
  url             = data.aws_eks_cluster.cluster.identity[0].oidc[0].issuer
}
{{< / highlight >}}

The tricky part here was that while when creating the OIDC provider with `eksctl` the OIDC thumbprint is added automatically, with Terraform we have to take care of this ourselves (which took me quiet some frustrating time to figure out). And as Terraform has no data provider to get the thumbprint, we need to fetch it with an external bash script. To do this, we first need to add a new provider:

{{< highlight hcl >}}
provider "external" {
  version = "~> 1.2"
}
{{< / highlight >}}

The external script (which you can also find [here](https://github.com/Finleap/tf-eks-fargate-tmpl/blob/master/eks/oidc_thumbprint.sh)) looks like this:

{{< highlight bash >}}
#!/bin/bash

if [[ -z "${ATLAS_WORKSPACE_SLUG}" ]]; then
  APP="tail -r"
else
  APP="tac"
fi

THUMBPRINT=$(echo | openssl s_client -servername oidc.eks.${1}.amazonaws.com -showcerts -connect oidc.eks.${1}.amazonaws.com:443 2>&- | ${APP} | sed -n '/-----END CERTIFICATE-----/,/-----BEGIN CERTIFICATE-----/p; /-----BEGIN CERTIFICATE-----/q' | ${APP} | openssl x509 -fingerprint -noout | sed 's/://g' | awk -F= '{print tolower($2)}')
THUMBPRINT_JSON="{\"thumbprint\": \"${THUMBPRINT}\"}"
echo $THUMBPRINT_JSON
{{< / highlight >}}

Now we can use the external script to fetch the OIDC thumbprint:

{{< highlight hcl >}}
data "external" "thumbprint" {
  program =    ["${path.module}/oidc_thumbprint.sh", var.region]
  depends_on = [aws_eks_cluster.main]
}
{{< / highlight >}}

### Fargate profile
In order to run pods in a Fargate (aka "serverless") configuration, we first need to create a Fargate profile. This profile defines namespaces and selectors, which are used to identify which pods should be run on the Fargate nodes.

> **Note**: By the time of writing this post, Fargate for EKS is available only in the following regions:
>- US East (N. Virginia)
>- US East (Ohio)
>- Europe (Ireland)
>- Asia Pacific (Tokyo)

{{< highlight hcl >}}
resource "aws_eks_fargate_profile" "main" {
  cluster_name           = aws_eks_cluster.main.name
  fargate_profile_name   = "fp-default"
  pod_execution_role_arn = aws_iam_role.fargate_pod_execution_role.arn
  subnet_ids             = var.private_subnets.*.id

  selector {
    namespace = "default"
  }

  selector {
    namespace = "2048-game"
  }
}
{{< / highlight >}}

The Fargate profile also needs a pod execution role, which is basically the same as the task execution role in an ECS setup, meaning it is the role that lets the fargate controller make calls to the AWS API on your behalf.

{{< highlight hcl >}}
resource "aws_iam_role_policy_attachment" "AmazonEKSFargatePodExecutionRolePolicy" {
  policy_arn = "arn:aws:iam::aws:policy/AmazonEKSFargatePodExecutionRolePolicy"
  role       = aws_iam_role.fargate_pod_execution_role.name
}

resource "aws_iam_role" "fargate_pod_execution_role" {
  name                  = "${var.name}-eks-fargate-pod-execution-role"
  force_detach_policies = true

  assume_role_policy = <<POLICY
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": [
          "eks.amazonaws.com",
          "eks-fargate-pods.amazonaws.com"
          ]
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
POLICY
}
{{< / highlight >}}

As you can see, with the above profile any pods in the namespace `default` or `2048-game` are run on fargate nodes. Like in the example this setup is based on ([this one](https://docs.aws.amazon.com/eks/latest/userguide/alb-ingress.html)), we will deploy a dockerized version of the 2048 game.

### Deployment
Creating a deployment of the said docker image to our cluster is pretty straight-forward: all we need to do is create what in Kubernetes terms is called a "deployment" to run the pod(s) and a "service" which will act as internal load balancer and make the deployment accessible for the ingress controller.

{{< highlight hcl >}}
resource "kubernetes_deployment" "app" {
  metadata {
    name      = "deployment-2048"
    namespace = "2048-game"
    labels    = {
      app = "2048"
    }
  }

  spec {
    replicas = 2

    selector {
      match_labels = {
        app = "2048"
      }
    }

    template {
      metadata {
        labels = {
          app = "2048"
        }
      }

      spec {
        container {
          image = "alexwhen/docker-2048"
          name  = "2048"

          port {
            container_port = 80
          }
        }
      }
    }
  }

  depends_on = [aws_eks_fargate_profile.main]
}

resource "kubernetes_service" "app" {
  metadata {
    name      = "service-2048"
    namespace = "2048-game"
  }
  spec {
    selector = {
      app = kubernetes_deployment.app.metadata[0].labels.app
    }

    port {
      port        = 80
      target_port = 80
      protocol    = "TCP"
    }

    type = "NodePort"
  }

  depends_on = [kubernetes_deployment.app]
}
{{< / highlight >}}

### Ingress controller
As mentioned before, in order to be able to access the deployed webapp now from a browser, we need an ALB to connect us to any running pod. In order to create this ALB and also register the available target pods (available through the added service) with the ALB, we now need to add an ingress controller.

> **Note**: The ingress controller is actually the most sophisticated way of creating an ALB and gives way more routing options than needed for the given example. But by the time of writing this post, the ingress controller was the only way to connect the ALB with the running pods, because of their Fargate configuration.

For the ingress controller to have access rights to create the ALB and also (de-)register target pods at the ALB, we need to create a policy first that will allow that.

{{< highlight hcl >}}
resource "aws_iam_policy" "ALBIngressControllerIAMPolicy" {
  name   = "ALBIngressControllerIAMPolicy"
  policy = <<POLICY
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "acm:DescribeCertificate",
        "acm:ListCertificates",
        "acm:GetCertificate"
      ],
      "Resource": "*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "ec2:AuthorizeSecurityGroupIngress",
        "ec2:CreateSecurityGroup",
        "ec2:CreateTags",
        "ec2:DeleteTags",
        "ec2:DeleteSecurityGroup",
        "ec2:DescribeAccountAttributes",
        "ec2:DescribeAddresses",
        "ec2:DescribeInstances",
        "ec2:DescribeInstanceStatus",
        "ec2:DescribeInternetGateways",
        "ec2:DescribeNetworkInterfaces",
        "ec2:DescribeSecurityGroups",
        "ec2:DescribeSubnets",
        "ec2:DescribeTags",
        "ec2:DescribeVpcs",
        "ec2:ModifyInstanceAttribute",
        "ec2:ModifyNetworkInterfaceAttribute",
        "ec2:RevokeSecurityGroupIngress"
      ],
      "Resource": "*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "elasticloadbalancing:AddListenerCertificates",
        "elasticloadbalancing:AddTags",
        "elasticloadbalancing:CreateListener",
        "elasticloadbalancing:CreateLoadBalancer",
        "elasticloadbalancing:CreateRule",
        "elasticloadbalancing:CreateTargetGroup",
        "elasticloadbalancing:DeleteListener",
        "elasticloadbalancing:DeleteLoadBalancer",
        "elasticloadbalancing:DeleteRule",
        "elasticloadbalancing:DeleteTargetGroup",
        "elasticloadbalancing:DeregisterTargets",
        "elasticloadbalancing:DescribeListenerCertificates",
        "elasticloadbalancing:DescribeListeners",
        "elasticloadbalancing:DescribeLoadBalancers",
        "elasticloadbalancing:DescribeLoadBalancerAttributes",
        "elasticloadbalancing:DescribeRules",
        "elasticloadbalancing:DescribeSSLPolicies",
        "elasticloadbalancing:DescribeTags",
        "elasticloadbalancing:DescribeTargetGroups",
        "elasticloadbalancing:DescribeTargetGroupAttributes",
        "elasticloadbalancing:DescribeTargetHealth",
        "elasticloadbalancing:ModifyListener",
        "elasticloadbalancing:ModifyLoadBalancerAttributes",
        "elasticloadbalancing:ModifyRule",
        "elasticloadbalancing:ModifyTargetGroup",
        "elasticloadbalancing:ModifyTargetGroupAttributes",
        "elasticloadbalancing:RegisterTargets",
        "elasticloadbalancing:RemoveListenerCertificates",
        "elasticloadbalancing:RemoveTags",
        "elasticloadbalancing:SetIpAddressType",
        "elasticloadbalancing:SetSecurityGroups",
        "elasticloadbalancing:SetSubnets",
        "elasticloadbalancing:SetWebACL"
      ],
      "Resource": "*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "iam:CreateServiceLinkedRole",
        "iam:GetServerCertificate",
        "iam:ListServerCertificates"
      ],
      "Resource": "*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "cognito-idp:DescribeUserPoolClient"
      ],
      "Resource": "*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "waf-regional:GetWebACLForResource",
        "waf-regional:GetWebACL",
        "waf-regional:AssociateWebACL",
        "waf-regional:DisassociateWebACL"
      ],
      "Resource": "*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "tag:GetResources",
        "tag:TagResources"
      ],
      "Resource": "*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "waf:GetWebACL"
      ],
      "Resource": "*"
    }
  ]
}
POLICY
}

resource "aws_iam_role" "eks_alb_ingress_controller" {
  name        = "eks-alb-ingress-controller"
  description = "Permissions required by the Kubernetes AWS ALB Ingress controller to do it's job."

  force_detach_policies = true

  assume_role_policy = <<ROLE
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::${data.aws_caller_identity.current.account_id}:oidc-provider/${replace(data.aws_eks_cluster.cluster.identity[0].oidc[0].issuer, "https://", "")}"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "${replace(data.aws_eks_cluster.cluster.identity[0].oidc[0].issuer, "https://", "")}:sub": "system:serviceaccount:kube-system:alb-ingress-controller"
        }
      }
    }
  ]
}
ROLE
}

resource "aws_iam_role_policy_attachment" "ALBIngressControllerIAMPolicy" {
  policy_arn = aws_iam_policy.ALBIngressControllerIAMPolicy.arn
  role       = aws_iam_role.eks_alb_ingress_controller.name
}
{{< / highlight >}}

Now in order to connect this IAM role to the cluster, we also need a cluster role for the ingress controller, a service account that is bound to this role and has the previously created IAM role attached.

{{< highlight hcl >}}
resource "kubernetes_cluster_role" "ingress" {
  metadata {
    name = "alb-ingress-controller"
    labels = {
      "app.kubernetes.io/name"       = "alb-ingress-controller"
      "app.kubernetes.io/managed-by" = "terraform"
    }
  }

  rule {
    api_groups = ["", "extensions"]
    resources  = ["configmaps", "endpoints", "events", "ingresses", "ingresses/status", "services"]
    verbs      = ["create", "get", "list", "update", "watch", "patch"]
  }

  rule {
    api_groups = ["", "extensions"]
    resources  = ["nodes", "pods", "secrets", "services", "namespaces"]
    verbs      = ["get", "list", "watch"]
  }
}

resource "kubernetes_cluster_role_binding" "ingress" {
  metadata {
    name = "alb-ingress-controller"
    labels = {
      "app.kubernetes.io/name"       = "alb-ingress-controller"
      "app.kubernetes.io/managed-by" = "terraform"
    }
  }
  role_ref {
    api_group = "rbac.authorization.Kubernetes.io"
    kind      = "ClusterRole"
    name      = kubernetes_cluster_role.ingress.metadata[0].name
  }
  subject {
    kind      = "ServiceAccount"
    name      = kubernetes_service_account.ingress.metadata[0].name
    namespace = kubernetes_service_account.ingress.metadata[0].namespace
  }

  depends_on = [kubernetes_cluster_role.ingress]
}

resource "kubernetes_service_account" "ingress" {
  automount_service_account_token = true
  metadata {
    name      = "alb-ingress-controller"
    namespace = "kube-system"
    labels    = {
      "app.kubernetes.io/name"       = "alb-ingress-controller"
      "app.kubernetes.io/managed-by" = "terraform"
    }
    annotations = {
      "eks.amazonaws.com/role-arn" = aws_iam_role.eks_alb_ingress_controller.arn
    }
  }
}
{{< / highlight>}}

And with this, we can now deploy the ingress controller into our cluster.

> **Note**: I used the `kube-system` namespace here, but this is not mandatory - you could also run it in a separate namespace.

{{< highlight hcl >}}
resource "kubernetes_deployment" "ingress" {
  metadata {
    name      = "alb-ingress-controller"
    namespace = "kube-system"
    labels    = {
      "app.kubernetes.io/name"       = "alb-ingress-controller"
      "app.kubernetes.io/version"    = "v1.1.5"
      "app.kubernetes.io/managed-by" = "terraform"
    }
  }

  spec {
    replicas = 1

    selector {
      match_labels = {
        "app.kubernetes.io/name" = "alb-ingress-controller"
      }
    }

    template {
      metadata {
        labels = {
          "app.kubernetes.io/name"    = "alb-ingress-controller"
          "app.kubernetes.io/version" = "v1.1.5"
        }
      }

      spec {
        dns_policy                       = "ClusterFirst"
        restart_policy                   = "Always"
        service_account_name             = kubernetes_service_account.ingress.metadata[0].name
        termination_grace_period_seconds = 60

        container {
          name              = "alb-ingress-controller"
          image             = "docker.io/amazon/aws-alb-ingress-controller:v1.1.5"
          image_pull_policy = "Always"
          
          args = [
            "--ingress-class=alb",
            "--cluster-name=${data.aws_eks_cluster.cluster.id}",
            "--aws-vpc-id=${var.vpc_id}",
            "--aws-region=${var.region}",
            "--aws-max-retries=10",
          ]

          volume_mount {
            mount_path = "/var/run/secrets/kubernetes.io/serviceaccount"
            name       = kubernetes_service_account.ingress.default_secret_name
            read_only  = true
          }

          port {
            name           = "health"
            container_port = 10254
            protocol       = "TCP"
          }

          readiness_probe {
            http_get {
              path   = "/healthz"
              port   = "health"
              scheme = "HTTP"
            }

            initial_delay_seconds = 30
            period_seconds        = 60
            timeout_seconds       = 3
          }

          liveness_probe {
            http_get {
              path   = "/healthz"
              port   = "health"
              scheme = "HTTP"
            }

            initial_delay_seconds = 60
            period_seconds        = 60
          }
        }

        volume {
          name = kubernetes_service_account.ingress.default_secret_name

          secret {
            secret_name = kubernetes_service_account.ingress.default_secret_name
          }
        }
      }
    }
  }

  depends_on = [kubernetes_cluster_role_binding.ingress]
}
{{< / highlight >}}

With the ingress controller deployed, now we can create the ALB for the webapp.

{{< highlight hcl >}}
resource "kubernetes_ingress" "app" {
  metadata {
    name      = "2048-ingress"
    namespace = "2048-game"
    annotations = {
      "kubernetes.io/ingress.class"           = "alb"
      "alb.ingress.kubernetes.io/scheme"      = "internet-facing"
      "alb.ingress.kubernetes.io/target-type" = "ip"
    }
    labels = {
        "app" = "2048-ingress"
    }
  }

  spec {
    rule {
      http {
        path {
          path = "/*"
          backend {
            service_name = kubernetes_service.app.metadata[0].name
            service_port = kubernetes_service.app.spec[0].port[0].port
          }
        }
      }
    }
  }

  depends_on = [kubernetes_service.app]
}
{{< / highlight >}}

## Step 4 - see if it works
When all these ressources are deployed, we can use `kubectl` to see if everything is up and running. Run the following command in your terminal:

`kubectl get ingress/2048-ingress -n 2048-game`

The output should look something like this:

```bash
NAME           HOSTS   ADDRESS                                                                 PORTS      AGE
2048-ingress   *       example-2048game-2048ingr-6fa0-352729433.us-west-2.elb.amazonaws.com   80      24h
```

If you copy the address in your web browser, you should see the 2048 game. Enjoy!

## Conclusion
After having played with ECS before, EKS is a different kind of monster. While with ECS everything is nicely playing together and relatively easy to understand and configure, it took me quite some time to get this example up and running.

I actually also wanted to add autoscaling to the example, but in order to do this you also have to deploy a `metrics-server` to you cluster, which for some reason didn't work very well for me.

This also a great example for the more simple setup with ECS because there things like autoscaling work basically out of the box. With Kubernetes you have to fight with the deployment of a separate metrics server in order for the horizontal pod autoscaler to get the metrics it can base up/downscale decisions on.

I feel Kubernetes is a very powerful system to run highly available applications, but I also feel that until you can confidently operate it, it takes a lot more time than with ECS.