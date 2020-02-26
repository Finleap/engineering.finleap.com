---
title: Running a K8s cluster on EKS with Fargate and Terraform
date: 2020-02-27T12:42:03+01:00
draft: false,
summary: A simple example on how to run a dockerized application in a K8s cluster with the help of AWS EKS and Fargate, all defined in Terraform
author: Andreas Jantschnig
author_position: Engineering Lead
author_email: andreas.jantschnig@finleap.com
---

As described in my previous post (which you can find [here](https://engineering.finleap.com/posts/2020-02-20-ecs-fargate-terraform/)), I recently started exploring the possibilities of IaC. Upon finishing my ECS setup, it was time to try the same thing with a system that seems to be one of the most widely used container orchestration systems: Terraform.

The target setup should be the same as in the previous ECS example, meaning the pods should run in a private subnet, communicating with the outside world via a load balancer placed in the public subnet.

TLDR; the resulting template repo can be found here: https://github.com/finleap/tf-eks-fargate-tmpl

## Step 1 - VPC
This step I won't describe much here, as it is actually pretty much same setup as in my previous ECS example, which you can find [here](https://engineering.finleap.com/posts/2020-02-20-ecs-fargate-terraform/#vpc)

The only difference when working with EKS, we have to add special tags to the subnets in order for K8s to know what the subnets should be used for:

- `kubernetes.io/cluster/{clustername}: shared` tag needs to be added to all subnets that the cluster should be able to use
- `kubernetes.io/role/elb: 1` tag needs to be added to the public subnets so that K8s knows to use only these subnets for public loadbalancers
- `kubernetes.io/role/internal-elb: 1` tag needs to be added to the private subnets, so that K8s knows to use these subnets for internal loadbalancers (actually not part of this example)

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

The `data` fields in the above setup will read the necessary data for initializing the `kubernetes` provider after the cluster was created via the aws provider.

As you can also see, we also need to attach a role to the cluster, which will give it the necessary permission for interacting with the nodes. The setup looks as follows:

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
Now we have the cluster setup and ready, but for now we have no nodes yet to run any pods on. It is actually possible to run the entire cluster fully on Fargate, but it requires some tweaking of the CoreDNS deployment (more on that [here](https://docs.aws.amazon.com/eks/latest/userguide/fargate-getting-started.html#fargate-gs-coredns) and [here](https://github.com/terraform-providers/terraform-provider-aws/issues/11327)).

For the sake of also using the Node Groups feature of EKS, is setup a node group for the `kube-system` namespace, which is used to run any pods necessary for operating the K8s cluster.

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

You can see that `desired_size` is set to `4`, which is because I chose to use `t2.micro` instances for this demo as they are free-tier elligable, but they also only provide max. 2 IP instances, which means you can only run 2 pods on each instance.

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
Now the cluster is ready to be used for deploying some pods to it. But one thing is missing for managing and monitoring the cluster: We need to enable `kubectl` to connect to this cluster, and for this we need to write a kube-config file. In order to do this, we first need two new provider in our Terraform setup:

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
      apiVersion: client.authentication.k8s.io/v1alpha1
      command: aws-iam-authenticator
      args:
        - "token"
        - "-i"
        - "${clustername}"
{{< / highlight >}}

With the following resources, we fill the template file with the values of our newly generated cluster and write it to the filesystem for `kubectl` to use it:

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

Note that `~/.kube/config` is the default path where `kubectl` looks for a configuration file. For more fine granular information about how to use `kubeconfig` files you can have a look [here](https://kubernetes.io/docs/concepts/configuration/organize-cluster-access-kubeconfig/).

## Step 3 - deploying a container to the cluster and running it on Fargate
The following section is basically the terraform-ed version of (this)[https://docs.aws.amazon.com/eks/latest/userguide/alb-ingress.html] example on how to deploy a simple webapp to an EKS cluster and running it on Fargate while exposing it to the outside world with the help of an ingress controller. The ingress controller is necessary as the available Terraform ressources to create ingress cannot communicate with pods running on fargate instances.

### OIDC provider
But one step at a time. First we need to add another resource to the cluster in order for the inress controller to be able to route traffic to the pods running the demo application, an OIDC provider:

{{< highlight hcl >}}
resource "aws_iam_openid_connect_provider" "main" {
  client_id_list  = ["sts.amazonaws.com"]
  thumbprint_list = [data.external.thumbprint.result.thumbprint]
  url             = data.aws_eks_cluster.cluster.identity[0].oidc[0].issuer
}
{{< / highlight >}}

The tricky part here was that while when creating the OIDC provider with `eksctl` the IODS thumbprint is added automatically, with Terraform we have to take care of this ourselves (which took me quiet some frustrating time to figure out). And as Terraform has no data provider to get the thumbprint, we need to fetch it with an external bash script. To do this, we first need to add a new provider:

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
In order to run pods in a fargate, aka "serverless", configuration, we first need to create a Fargate profile. This profile defines namespaces and selectors, which are used to identify which pods should be run on the Fargate nodes.

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

As you can see, with the above profile any pods in the namespace `default` or `2048-game` are run on fargate nodes. Like in the exapmle this setup is based on ([this one](https://docs.aws.amazon.com/eks/latest/userguide/alb-ingress.html)), we will deploy a dockerized version of the 2048 game.

### Deployment
Creating a deployment of the said docker image to our cluster is pretty straight forward, all we need to do is create what in K8s terms is called a "deployment" to run the pod(s) and a "service" to make it accessible for the ingress controller.

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
After having played with ECS before, EKS is a different kind of monster. While with ECS everything is nicely playing together and relatively easy to understand and configure, it took me quiet some time to get this example up and running. I actually also wanted to add autoscaling to the example, but in order to do this you also have to deploy a `metrics-server` to you cluster, which for some reason didn't work very well for me.

But this also a great example for the more simple setup with ECS, because there things like autoscaling work basically out of the box, while with K8s you have to fight with the deployment of a separate metrics server in order for the horizontal pod autoscaler to get the metrics it can base up/downscale decissions on.

I feel K8s is a very powerful system to run highly available applications, but I also feel that until you can confidently operate it, it takes a lot more time than with ECS.