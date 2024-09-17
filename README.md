# Reverse-proxy-helm-chart

# Solution Overview

The reason we use this solution is that we have an on-premises project where we need to automate the deployment process. However, we face two main challenges:

  Limited direct connectivity: Our on-prem environment cannot directly connect to Azure DevOps. This is because we do not have an open connection between our servers and Azure DevOps, making direct communication impossible.

  Azure IP rotation: Azure DevOps uses dynamic IP addresses, which rotate frequently. This rotation complicates maintaining a secure, consistent connection from the on-prem environment, as we cannot whitelist such a fluctuating set of IPs through our firewall.

This solution involves setting up a reverse proxy using NGINX, which forwards requests received on a static IP from an external Load Balancer to a specific Azure DevOps link. This was achieved using Bitnami's Helm chart for NGINX, with custom configurations to ensure the system meets our requirements.
[https://artifacthub.io/packages/helm/bitnami/nginx](https://artifacthub.io/packages/helm/bitnami/nginx)

## 1.2 Implementation Details

### 1.2.1 Container Ports

I started by adjusting the container ports to ensure that NGINX only listens on HTTP (port 8080) and does not attempt to handle HTTPS traffic directly. This setup is deliberate because the traffic will be encrypted at the Load Balancer level, which forwards it as HTTP to NGINX.

**values.yaml**

```yaml
## Configures the ports NGINX listens on
## @param containerPorts.http Sets http port inside NGINX container
## @param containerPorts.https Sets https port inside NGINX container
##
containerPorts:
  http: 8080
  https: {}
```

### 1.2.2 Custom Server Block

To direct traffic to the Azure DevOps URL, I added a custom server block to the NGINX configuration. This block listens on port 8080 and forwards requests matching the /VFCom-oneportal/ path to the Azure DevOps package feed.

**values.yaml**

```yaml
serverBlock: |-
  server {
    listen 8080;
    server_name  localhost;
    location /VFCom-oneportal/ {
        proxy_pass https://pkgs.dev.azure.com/VFCom-oneportal/;
   }
  }
```

### 1.2.3 Configuring the Load Balancer

Next, I configured the Load Balancer to receive traffic on port 443 and route it to NGINX's HTTP port (8080). This allows the Load Balancer to handle SSL termination, providing encrypted communication between clients and the Load Balancer while simplifying NGINX’s role.

**values.yaml**

```yaml
service:
  type: LoadBalancer
  ports:
    http: 443
    https: {}
```

### 1.2.4 Annotations for Load Balancer

To properly configure the Load Balancer, I added the following annotations to ensure it uses a static IP (Elastic IP), a specific public subnet, and an SSL certificate for HTTPS traffic. These annotations are critical for ensuring that the traffic is correctly routed and secured.

**values.yaml**

```yaml
## @param service.annotations Service annotations
annotations:
  service.beta.kubernetes.io/aws-load-balancer-scheme: internet-facing
  service.beta.kubernetes.io/aws-load-balancer-type: "nlb"
  service.beta.kubernetes.io/aws-load-balancer-eip-allocations: eipalloc-xxx
  service.beta.kubernetes.io/aws-load-balancer-ssl-cert: arn:aws:acm:eu-west-1:xxx
  service.beta.kubernetes.io/aws-load-balancer-ssl-ports: "443"
  service.beta.kubernetes.io/aws-load-balancer-subnets: subnet-xxx
```

## 1.3 Deploying the Solution

To streamline and automate the deployment process, I used GitHub Actions. The workflow allows you to select the environment and automatically configures AWS credentials, installs the necessary tools, and deploys the NGINX Helm chart.

**values.yaml**

```yaml
name: Deploy Helm chart
 
on:
  workflow_dispatch:
    inputs:
      ENVIRONMENT:
        description: 'Select the environment'
        required: true
        type: choice
        options:
          - sandbox
          - maac-mgmt
   
  workflow_call:
 
permissions:
  id-token: write
  contents: write
 
jobs:   
  install_chart:
    environment: ${{ github.event.inputs.ENVIRONMENT }}
    runs-on: self-hosted
    steps:
      - uses: VFGroup-VBIT/vbitdc-opf-actions/aws/install-cli-action@main
       
      - name: Check out code
        uses: actions/checkout@v4
       
      - name: configure aws credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::${{ vars.AWS_ACCOUNT_ID }}:role/Github-Runners-Access
          role-session-name: GitHub_to_AWS_via_FederatedOIDC
          aws-region: ${{ vars.AWS_DEFAULT_REGION }}
 
      - id: private-modules
        uses: VFGroup-VBIT/vbitdc-opf-actions/private-modules@main
        with:
          org: VFGroup-VBIT
          token: ${{ secrets.GHTOKEN }}
 
      - name: Log IP
        run: curl -s https://checkip.amazonaws.com/
 
      - name: Log AWS principal
        run: aws sts get-caller-identity
 
      - name: Install Helm and kubectl
        id: install-helm
        shell: bash
        run: |
          # Install kubectl
          pip install kubernetes
          curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
          chmod +x kubectl
          mkdir -p ~/.local/bin
          mv ./kubectl ~/.local/bin/kubectl
          kubectl version --client
 
          curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
          helm version
           
          aws eks update-kubeconfig --region ${{ vars.AWS_DEFAULT_REGION }} --name ${{ vars.CLUSTER_NAME }}
 
      - name: Install Helm chart
        shell: bash
        run: |
          helm repo add bitnami https://charts.bitnami.com/bitnami
          helm repo update
 
          helm upgrade --install --create-namespace --namespace vgeco-reverse-proxy             -f ./nginxChart/config/${{ vars.AWS_ACCOUNT_NAME }}-${{ vars.CLUSTER_NAME }}.yaml             test-release bitnami/nginx
```

### 1.3.1 Add Loadbalancer DNS to the route 53 record

Run the below command to get the load balancer DNS and add it to Route53 in the hosted zone that has the certificate.

```bash
kubectl get svc test-release-nginx -n vgeco-reverse-proxy -o jsonpath="{.status.loadBalancer.ingress[0].hostname}"
```
