# 2048 Application deployment on AWS Elastic Kubernetes Service (EKS)

This application is a game app that is deployed on aws Elastic Kubernetes Service.EKS is a cloud based platform that ochestrate a containerize appication in a robust and efficient way.

Deploying an application on **Amazon EKS (Elastic Kubernetes Service)** involves several steps, including configuring the EKS cluster, preparing the application, deploying it to the cluster, and managing the deployment. Here’s a detailed guide to deploy an application on EKS:

### Prerequisites

1. **AWS Account**: You'll need an AWS account.
2. **AWS CLI**: Install and configure the AWS CLI ([installation guide](https://docs.aws.amazon.com/cli/latest/userguide/install-cliv2.html)).
3. **kubectl**: Kubernetes command-line tool ([installation guide](https://kubernetes.io/docs/tasks/tools/install-kubectl/)).
4. **eksctl**: Command-line tool to create and manage EKS clusters ([installation guide](https://eksctl.io/)).
5. **IAM Permissions**: Ensure that you have the correct IAM permissions to create and manage resources in AWS.

---

### Step 1: Create an EKS Cluster

1. **Install eksctl** (if not installed):

   ```bash
   brew tap weaveworks/tap
   brew install weaveworks/tap/eksctl
   ```

   If using Linux:

   ```bash
   curl --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
   sudo mv /tmp/eksctl /usr/local/bin
   ```

2. **Create EKS Cluster**:
   Run the following command to create a cluster with `eksctl`:

   ```bash
   eksctl create cluster --name my-ish-cluster --region us-east-1 --fargate
   ```

   This command:

   - Creates an EKS cluster named `my-ish-cluster`.
   - Deploys fargate node in the region `us-east-1`.

   **Optional Parameters**:

   - `--node-type`: Specify the instance type for your worker nodes.
   - `--version`: Specify the Kubernetes version for the EKS cluster.

3. **Verify Cluster**:
   Check that the Kubernetes context is correctly configured by running:

   ```bash
   kubectl get svc
   ```

   This should display the services in the default namespace, indicating that your cluster is running.

---

### Step 2: Configure kubectl to Connect to EKS Cluster

After creating the EKS cluster, ensure `kubectl` is configured to communicate with it. EKS uses IAM authentication, so ensure AWS credentials are configured properly.

1. **Update `kubectl` Config**:

   ```bash
   aws eks --region us-east-1 update-kubeconfig --name my-ish-cluster
   ```

2. **Test Cluster Access**:
   Verify you can connect to the cluster:

   ```bash
   kubectl get nodes
   ```

---

### Step 3: Prepare Your Application

In our case we pull the 2048 application image from AWS development official github repo and this will factored in the deployment yaml.

public.ecr.aws/l6m2t8p7/docker-2048:latest

**_When following my 2048 app deployement move to step 4. The below is a node js app_**

                    **_ Below is an alternative way of deploying and node js application. This will help you to create your own dockerfile and build the app image _**.

                    1. **Dockerize Your Application**:
                    If your application isn't containerized yet, create a `Dockerfile` to build a container image.

                    Example `Dockerfile`:

                    ```dockerfile
                    FROM node:14
                    WORKDIR /app
                    COPY package*.json ./
                    RUN npm install
                    COPY . .
                    EXPOSE 8080
                    CMD ["npm", "start"]
                    ```

                    2. **Build the Docker Image**:

                    ```bash
                    docker build -t my-app .
                    ```

                    3. **Push the Docker Image to ECR**:
                    - **Create an ECR repository** (if not already created):
                        ```bash
                        aws ecr create-repository --repository-name my-app-repo
                        ```
                    - **Authenticate Docker to ECR**:
                        ```bash
                        aws ecr get-login-password --region us-west-2 | docker login --username AWS --password-stdin <aws_account_id>.dkr.ecr.us-west-2.amazonaws.com
                        ```
                    - **Tag and Push Image**:
                        ```bash
                        docker tag my-app:latest <aws_account_id>.dkr.ecr.us-west-2.amazonaws.com/my-app-repo:latest
                        docker push <aws_account_id>.dkr.ecr.us-west-2.amazonaws.com/my-app-repo:latest
                        ```

                    ---

### Step 4: Deploy the Application to EKS

**_Note In your alternative node js app you create above, change the app name and the image name_**

1. **Create Kubernetes Deployment YAML**:
   Define your application deployment in a `deployment.yaml` file. This deployment file consist of namespace and deployment configuration. Namespace host your application within the cluster. It is essentially a logical partitioning of the cluster, used to organize and manage resources like pods, services, and deployments in a Kubernetes environment. Namespaces provide a mechanism for isolating resources and enabling multiple users or teams to share a Kubernetes cluster without interfering with each other

   Example deployment configuration:

   ```yaml
   ---
   apiVersion: v1
   kind: Namespace
   metadata:
   name: game-2048
   ---
   apiVersion: apps/v1
   kind: Deployment
   metadata:
   namespace: game-2048
   name: deployment-2048
   spec:
   selector:
       matchLabels:
       app.kubernetes.io/name: app-2048
   replicas: 2
   template:
       metadata:
       labels:
           app.kubernetes.io/name: app-2048
       spec:
       containers:
           - image: public.ecr.aws/l6m2t8p7/docker-2048:latest
           imagePullPolicy: Always
           name: app-2048
           ports:
               - containerPort: 80
   ```

2. **Deploy to EKS**:

   ```bash
   kubectl apply -f deployment.yaml
   ```

3. **Expose the Application**:
   To expose your application to external traffic, create a service.

   below is an example service configuration:

   ```yaml
    apiVersion: v1
    kind: Service
    metadata:
    namespace: game-2048
    name: service-2048
    spec:
    ports:
        - port: 80
        targetPort: 80
        protocol: TCP
    type: NodePort
    selector:
        app.kubernetes.io/name: app-2048
   ```

4. **Deploy the Service**:

   ```bash
   kubectl apply -f service.yaml
   ```

5. **Get the External IP**: This is used if you wants to host your application of a single service. But in our case we will use ingress in step 6.
   After deploying the service, get the external IP of the load balancer:

   ```bash
   kubectl get svc my-app-service
   ```

---

6. **Using Ingress as a loadbalancer for multiple service**
   Ingress is a resource that manages external access to services within a cluster, typically HTTP and HTTPS traffic. It provides a way to expose your Kubernetes services to the outside world while offering features like load balancing, SSL termination, and name-based virtual hosting.

   Example ingress configuration:

   ```yaml
    apiVersion: networking.k8s.io/v1
    kind: Ingress
    metadata:
    namespace: game-2048
    name: ingress-2048
    annotations:
        alb.ingress.kubernetes.io/scheme: internet-facing
        alb.ingress.kubernetes.io/target-type: ip
    spec:
    ingressClassName: alb
    rules:
        - http:
            paths:
            - path: /
                pathType: Prefix
                backend:
                service:
                    name: service-2048
                    port:
                    number: 80

   ```

7. **Configure OIDC Provider**

   # commands to configure IAM OIDC provider

   ```
   export cluster_name=my-ish-cluster
   ```

   ```
   oidc_id=$(aws eks describe-cluster --name $cluster_name --query "cluster.identity.oidc.issuer" --output text | cut -d '/' -f 5)
   ```

   ## Check if there is an IAM OIDC provider configured already

   - aws iam list-open-id-connect-providers | grep $oidc_id | cut -d "/" -f4\n

   If not, run the below command

   ```
   eksctl utils associate-iam-oidc-provider --cluster $cluster_name --approve
   ```

8. **Install alb controller**

   # How to setup alb add on

   Download IAM policy

   ```
   curl -O https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.5.4/docs/install/iam_policy.json
   ```

   Create IAM Policy

   ```
   aws iam create-policy \
       --policy-name AWSLoadBalancerControllerIAMPolicy \
       --policy-document file://iam_policy.json
   ```

   Create IAM Role

   ```
   eksctl create iamserviceaccount \
   --cluster=my-ish-cluster \
   --namespace=kube-system \
   --name=aws-load-balancer-controller \
   --role-name AmazonEKSLoadBalancerControllerRole \
   --attach-policy-arn=arn:aws:iam::<your-aws-account-id>:policy/AWSLoadBalancerControllerIAMPolicy \
   --approve
   ```

   ## Deploy ALB controller

   Add helm repo

   ```
   helm repo add eks https://aws.github.io/eks-charts
   ```

   Update the repo

   ```
   helm repo update eks
   ```

   Install

   ```
   helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
   -n kube-system \
   --set clusterName=my-ish-cluster \
   --set serviceAccount.create=false \
   --set serviceAccount.name=aws-load-balancer-controller \
   --set region=us-east-1\
   --set vpcId=<your-vpc-id>
   ```

   Verify that the deployments are running.

   ```
   kubectl get deployment -n kube-system aws-load-balancer-controller
   ```

   2048 Appication Image
   ![2048 Application Image](./2048_pic.png)

### Step 5: Monitoring and Scaling

1. **Monitor Logs**:

   ```bash
   kubectl logs app-2048
   ```

2. **Scaling the Deployment**:
   You can scale the application horizontally by increasing the number of replicas:
   ```bash
   kubectl scale deployment deployment-2048 --replicas=3
   ```

---

### Step 6: Cleanup (Optional)

To delete the deployment and the cluster:

1. **Delete the Service and Deployment**:

   ```bash
   kubectl delete svc service-2048
   kubectl delete deployment deployment-2048
   kubectl delete ingress ingress-2048
   ```

2. **Delete the EKS Cluster**:
   ```bash
   eksctl delete cluster --name my-ish-cluster --region us-east-1
   ```

---

### Conclusion

This process involves creating an EKS cluster, deploying your Dockerized application, and exposing it via a load balancer. EKS abstracts much of the heavy lifting of managing Kubernetes infrastructure, while giving you powerful tools to scale and manage containerized applications.
