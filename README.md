# DevOps Technical Challenge
Requirements of project is to support microservices architecture consisting of frontend, backend and PostgreSQL database.

## Infrastructure Platform
For infrastructure platform, I choose AWS as it has more regions and availability zones and it provides numerous services and intuitive UI when compared with other cloud providers.

## Orchestration Technology
I choose Kubernetes(EKS) for container orchestration.
### Components of orchestration
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

**Given below is the snippet for istio virtual service for canary deployment.**
```
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: app-vs
spec:
  hosts:
  - app.myapp.com
  gateways:
  - app-gateway
  http:
  - match:
    - headers:
        x-canary:
          exact: "true"
    route:
    - destination:
        host: app-canary
        subset: v2
        port:
          number: 80
      weight: 20
    - destination:
        host: app-stable
        subset: v1
        port:
          number: 80
      weight: 80
```
## Microservices Deployment
For automating microservices deployment, I choose Github Actions and Argo CD. CI using Github Actions and Argo CD with GitOps approach for Continous Deployment

I would use multistage builds with security best practices such as using distroless images where it is appropriate for building the container images.

**Given below is a snippet of a Dockerfile for a GO app.**
```
# Stage 1: Build
FROM golang:1.20 as builder
WORKDIR /app
COPY go.mod ./
COPY go.sum ./
RUN go mod download
COPY . ./
RUN go build -o main .

# Stage 2: Production
FROM gcr.io/distroless/static:nonroot
WORKDIR /
COPY --from=builder /app/main .
USER nonroot:nonroot
ENTRYPOINT ["/main"]
```
For CI pipeline, I would integrate security testing tools for SAST and DAST like SonarQube, trivy for image scanning etc.

**Given below is a snippet of a GitHub Actions workflow.**
```
name: CI Pipeline

on:
  push:
    branches:
      - main

jobs:
  scan-and-build:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v3

      - name: Run Trivy Image Scan
        uses: aquasecurity/trivy-action@v0.11.0
        with:
          image-ref: <AWS_ACCOUNT_ID>.dkr.ecr.us-east-1.amazonaws.com/backend:latest

      - name: SonarQube Scan
        uses: sonarsource/sonarcloud-github-action@v2
        with:
          args: >
            -Dsonar.projectKey=my-project-key
            -Dsonar.organization=my-org
            -Dsonar.host.url=https://sonarcloud.io
            -Dsonar.login=${{ secrets.SONAR_TOKEN }}

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v2

      - name: Login to Amazon ECR
        run: aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin <AWS_ACCOUNT_ID>.dkr.ecr.us-east-1.amazonaws.com

      - name: Build and Push Docker Image
        run: |
          docker build -t <AWS_ACCOUNT_ID>.dkr.ecr.us-east-1.amazonaws.com/backend:latest .
          docker push <AWS_ACCOUNT_ID>.dkr.ecr.us-east-1.amazonaws.com/backend:latest
```
## Release Lifecycle
The release lifecycle would include the following.
- Development:
  - Feature branches
  - Local testing
  - PR 
- CI Pipeline:
  - Code validation
  - Unit tests
  - Integration tests
  - Container image build
  - Security scanning
  - Container image pushed to AWS ECR
- CD Pipeline
  - ArgoCD environment promotion
  - Staging deployment
  - Integration testing
  - Production deployment
- Post-deployment
  - Health checks
  - Canary deployment using Istio
  - Full rollout after monitoring

## Testing Approach
- Infrastructure Testing:
  - Validate Terraform configurations using Terratest
- Microservices Testing:
  - Using SonarQube to test for bugs/vulnerabilities
  - Unit Tests
  - Using Trivy to scan container images
  - Integration Tests

## Observability Strategy
Opting for open source tools here would greatly reduce our costs compared to offerings like Cloudwatch or Datadog.

For observability, I choose Prometheus to scrape metrics from the cluster and microservices, Grafana for dashboarding and ELK stack for aggregating logs and analyzing them. 

### Monitoring Areas
- Infrastructure Metrics
  - Node CPU/Memory utilization
  - Network 
  - Disk usage

- Application Metrics

  - Request latency
  - Error rates
  - Transaction throughput
  - Custom metrics

- Logs
  - Application logs
  - System logs
  - Audit logs
  - Security logs