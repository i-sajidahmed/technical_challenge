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
I choose Terraform for automating infrastructure deployment. 

## Microservices Deployment
For automating microservices deployment, I choose Github Actions and Argo CD.



