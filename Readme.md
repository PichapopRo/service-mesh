# **Consul Multi-Cloud Service Mesh Project**

### Demo project accompanying a [Consul crash course video](https://www.youtube.com/watch?v=s3I1kKKfjtQ) on YouTube

This project demonstrates how to architect a **Consul Service Mesh** to manage a complex microservices application distributed across a multi-cloud environment (**AWS EKS** and **Linode LKE**). It focuses on solving challenges related to service connectivity, security (Mutual TLS), and cross-cluster failover.

---

## **1. Infrastructure Setup (Terraform)**

The `/terraform` directory contains scripts to provision a production-ready **AWS EKS cluster**. Using these scripts ensures a consistent environment with the necessary networking for a service mesh.

*   **`vpc.tf`**: Creates a VPC with **public and private subnets** and a public endpoint for the Kubernetes API server.
*   **`eks.tf` & `node-groups.tf`**: Configures the control plane and a managed node group with **three worker nodes**.
*   **Security Groups**: Includes rules to **open specific ports** (like those required for the Consul control plane and data plane) to allow components to communicate.
*   **EBS CSI Driver**: Configures the Amazon Elastic Block Storage add-on, which is required because Consul is a **stateful application** that must persist data.

### **Terraform Execution Commands**
```sh
# initialise project & download providers
terraform init

# preview what will be created with apply & see if any errors
terraform plan

# execute with preview
terraform apply -var-file terraform.tfvars

# execute without preview
terraform apply -var-file terraform.tfvars -auto-approve

# destroy everything
terraform destroy

# show resources and components from current state
terraform state list
```
*Note: You must set your `access_key` and `secret_key` in `terraform.tfvars` before running these commands.*

---

## **2. Cluster Access (AWS CLI)**

Once the infrastructure is provisioned, use these commands to configure your local environment to interact with the EKS cluster.

```sh
# install and configure awscli with access creds
aws configure

# check existing clusters list
aws eks list-clusters --region eu-central-1 --output table --query 'clusters'

# check config of specific cluster
aws eks describe-cluster --region eu-central-1 --name myapp-eks-cluster --query 'cluster.resourcesVpcConfig'

# create kubeconfig file for cluster in ~/.kube
aws eks update-kubeconfig --region eu-central-1 --name myapp-eks-cluster

# test configuration
kubectl get node
```
*Successfully running `kubectl get node` should show three worker nodes running the specified Kubernetes version.*

---

## **3. Microservices & Consul Configuration**

The `/kubernetes` directory contains the application manifests and the service mesh logic.

### **The Application**
The project uses the **Google Cloud Open Source Microservices Demo**, which includes services like a product catalog, checkout, and email service.
*   **`config.yaml`**: Standard Kubernetes manifests for the application.
*   **`config-consul.yaml`**: A version of the app integrated with Consul using **annotations**:
    *   `consul.hashicorp.com/connect-inject`: Automatically injects the **Envoy sidecar proxy**.
    *   `consul.hashicorp.com/connect-service-upstreams`: Defines which services a pod needs to reach via `localhost` through the mesh.

### **Consul Helm & CRDs**
*   **`consul-values.yaml`**: Configures the mesh via Helm, enabling **TLS** for security, **Peering** for multi-cluster connectivity, and the **Mesh Gateway** to act as a "guard" for cross-cluster traffic.
*   **`intentions.yaml`**: Defines **micro-network segmentation** policies, acting as a service-level firewall to control which services can communicate.
*   **`mesh.yaml`**: A CRD used to tell Consul to route peering traffic specifically through the Mesh Gateways.
*   **`exported-services.yaml`**: Shares a service from the secondary cluster (LKE) with the primary cluster (EKS).
*   **`service-resolver.yaml`**: Defines **failover logic**, automatically redirecting traffic to the peer cluster if a local service fails.

---

## **4. Operational Workflows**

1.  **Multi-Cloud Connectivity**: The clusters are connected by generating a **peering token** in the EKS UI/CLI and establishing the connection in the LKE cluster.
2.  **Mutual TLS (mTLS)**: Once deployed, Consul handles all encryption and decryption between proxies, providing **end-to-end security** without modifying application code.
3.  **Service Failover**: If the `shipping-service` in AWS is deleted, the `service-resolver` automatically reroutes traffic to the `shipping-service` in Linode, ensuring the application remains functional.