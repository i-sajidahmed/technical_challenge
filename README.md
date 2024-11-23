# DevOps Technical Challenge
Requirements of project is to support microservices architecture consisting of frontend, backend and PostgreSQL database.

## Infrastructure Platform
For infrastructure platform, I choose AWS as it has more regions and availability zones and it provides numerous services and intuitive UI when compared with other cloud providers.

## Orchestration Technology
I choose Kubernetes(EKS) for container orchestration.
# Components of orchestration
- EKS with EC2 instances for the cluster.
- I choose Karpenter for autoscaling of the cluster. It is efficent in scaling according to the needs of the workloads. It helps in reducing costs.

- For PostgreSQL database, I choose CloudNativePG operator to manage the PostgreSQL database within the cluster. When compared to the managed service like RDS, cost savings would be huge.

- Istio for handling service mesh, canary deployments and routing traffic. Istio helps in securing the traffic between the services with mTLS and also helps in observability. Canary deployments can be implemented effectively using istio.

- AWS Secrets Manager with CSI driver to securely store and access secrets.

- Helm for simplified management of configuration, packaging application deployments.

- Argo CD for GitOps based approach of Continous Deployment of applications to the cluster.


## Infrastructure Automation
I choose Terraform for automating infrastructure deployment. Using Terraform, we can create modular code for reusability. It has declarative approach is efficient in creating and managing the resources.

### Given below is the snippets of terraform for vpc
```
resource "aws_vpc" "main" {
  cidr_block           = var.vpc_cidr
  enable_dns_hostnames = true
  enable_dns_support   = true

  tags = {
    Name        = "${var.environment}-vpc"
    Environment = var.environment
  }
}

resource "aws_subnet" "private" {
  count             = 3
  vpc_id            = aws_vpc.main.id
  cidr_block        = cidrsubnet(var.vpc_cidr, 4, count.index)
  availability_zone = data.aws_availability_zones.available.names[count.index]

  tags = {
    Name                              = "${var.environment}-private-${count.index + 1}"
    "kubernetes.io/role/internal-elb" = "1"
  }
}

resource "aws_subnet" "public" {
  count             = 3
  vpc_id            = aws_vpc.main.id
  cidr_block        = cidrsubnet(var.vpc_cidr, 4, count.index + 3)
  availability_zone = data.aws_availability_zones.available.names[count.index]

  tags = {
    Name                     = "${var.environment}-public-${count.index + 1}"
    "kubernetes.io/role/elb" = "1"
  }
}

```

### Given below is the snippets of terraform for EKS cluster
```
resource "aws_eks_cluster" "main" {
  name     = var.cluster_name
  role_arn = aws_iam_role.eks_cluster.arn
  version  = "1.31"

  vpc_config {
    subnet_ids = var.private_subnet_ids
  }

  depends_on = [
    aws_iam_role_policy_attachment.eks_cluster_policy
  ]
}

resource "aws_eks_node_group" "main" {
  cluster_name    = aws_eks_cluster.main.name
  node_group_name = "${var.cluster_name}-node-group"
  node_role_arn   = aws_iam_role.eks_node.arn
  subnet_ids      = var.private_subnet_ids

  scaling_config {
    desired_size = 2
    max_size     = 5
    min_size     = 1
  }

  instance_types = ["t3.medium"]
}

```

## Microservices Deployment
For automating microservices deployment, I choose Github Actions and Argo CD.



