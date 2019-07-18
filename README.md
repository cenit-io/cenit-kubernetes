# CENIT WITH KUBERNETES 

![Kubernetes Logo](https://raw.githubusercontent.com/kubernetes-sigs/kubespray/master/docs/img/kubernetes-logo.png)

![cenit_io](https://user-images.githubusercontent.com/4213488/40578188-bcbf8a58-60c4-11e8-96d7-19842c348c5e.png)

## INTRODUCTION

Kubernetes is a platform for managing application containers across multiple hosts. It provides lots of management features for container-oriented applications, such as auto scaling, rolling deployment, compute resource, and volume management. Same as the nature of containers, it’s designed to run anywhere, so we’re able to run it on a bare metal, in our data center, on the public cloud, or even hybrid cloud. As part of our services we have developed configurations using this technology so that our clients can deploy our **CENIT** integration platform in their customized clusters.

## Tools needed

### Amazon

Amazon Elastic Kubernetes Service (Amazon EKS) makes it easy to deploy, manage, and scale containerized applications using [Kubernetes on AWS](https://aws.amazon.com/kubernetes). Exist two ways to create and interactuate with a Kubernetes cluster in Amazon EKS. The first is \textbf{eksctl}, a simple command line utility for creating and managing Kubernetes clusters on Amazon EKS and the second choice is Amazon EKS in the [AWS Management Console](https://docs.aws.amazon.com/en\_en/eks/latest/userguide/getting-started.html).

### Instaling eksctl

To install **eksctl** first you need to install AWS Command Line Interface (AWS CLI). You can install the AWS CLI and its dependencies on most Linux distributions by using pip, a package manager for Python.

```bash
pip3 install awscli --upgrade
```

Both **eksctl** and the AWS CLI require that you have AWS credentials configured in your environment. The aws configure command is the fastest way to set up your AWS CLI installation for general use. 

```bash
aws configure
AWS Access Key ID [None]: AKIAIOSFODNN7EXAMPLE
AWS Secret Access Key [None]: wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
Default region name [None]: us-west-2
Default output format [None]: json
```

To install **eksctl** download and extract the latest release of **eksctl** with the following command:

```bash
curl --silent --location "https://github.com/weaveworks/eksctl/releases/download/latest_release/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
```

Move the extracted binary to **_/usr/local/bin_**:

```bash
sudo mv /tmp/eksctl /usr/local/bin
```

Kubernetes uses a command line utility called \textbf{kubectl} for communicating with the cluster API server. The **kubectl** binary is available in many operating system package managers, and this option is often much easier than a manual download and install process. You can follow the instructions for your specific operating system or package manager in the [Kubernetes documentation](https://kubernetes.io/docs/tasks/tools/install-kubectl/) to install.

>**NOTE:**
You must use a **kubectl** version that is within one minor version difference of your Amazon EKS cluster control plane . For example, a 1.11 **kubectl** client should work with Kubernetes 1.10, 1.11, and 1.12 clusters.

Download the Amazon EKS-vended kubectl binary for your cluster's Kubernetes version from Amazon S3:

```bash
curl -o kubectl https://amazon-eks.s3-us-west-2.amazonaws.com/1.12.9/2019-06-21/bin/linux/amd64/kubectl
```

Apply execute permissions to the binary:

```bash
chmod +x ./kubectl
```

Copy the binary to a folder in your **_PATH_**. If you have already installed a version of **kubectl**, then we recommend creating a **_$HOME/bin/kubectl_** and ensuring that **_$HOME/bin_** comes first in your **_$PATH_**. 

```bash
mkdir -p $HOME/bin && cp ./kubectl $HOME/bin/kubectl && export PATH=$HOME/bin:$PATH
```

## Creating cluster

Now we can create our Amazon EKS cluster and a worker node group with the \textbf{eksctl} command line utility or using [AWS Management Console](https://docs.aws.amazon.com/en_en/eks/latest/userguide/getting-started-console.html).

>**NOTE:**
The choice of how the cluster should be built is left to you.

### Deployin Kubernetes configuration of Cenit

For deploy all Cenit configuration just run the following command:

```bash
kubectl apply -f ./
```

### Configuration  files description

We have eight files configuration that represent eight kubernetes objects. Three Deployment, two ClusterIp, two PVC and a LoadBalancer objects. The most relevant objects are described below:

>mongo-pvc.yml

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mongo-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
```

The PersistentVolumeClaim (PVC) are configuration objects to store all our data, in this case to store all MongoDB data (we use the same filosophie for RabbitMQ).

>**NOTE:**
For develop purpose we test all our configuration with an instance of a containerized MongoDB service. But we offer the version of our product that use an external provider of database like mLab or other services.

>mongo-clusterip-service.yml

```yaml
apiVersion: v1
kind: Service
metadata:
  name: rabbitmq-cluster-ip-service
spec:
  type: ClusterIP
  selector:
    amqp: rabbitmq
  ports:
  - name: consumers-port
    port: 5672
    targetPort: 5672
  - name: gui-port
    port: 15672
    targetPort: 15672
```

Services are Kubernetes objects that provide the reliable way to access applications running on the pods. Services are what makes pods consistently accessible. Services connect Pods together, or provide access outside of the cluster. The ClusterIP service type is the default, and only provides access internally on a cluster internal IP. We use this service type for accessing internal traffic only.

>cenit-deployment.yml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cenit-deployment
spec:
  replicas: 2
  selector:
    matchLabels:
      platform: cenit
  template:
    metadata:
      labels:
        platform: cenit
    spec:
      containers:
      - name: cenit
        image: alejandro92/cenit:uri
        env:
          - name: MONGODB_URI
            value: mongodb+srv://cenit-dev@cenitio-dev-mkcsx.mongodb.net/cenit_prod
          - name: ENABLE_RERECAPTCHA
            value: "false"
          - name: UNICORN_WORKERS
            value: "5"
          - name: MAXIMUM_UNICORN_CONSUMERS
            value: "3"
          - name: RABBITMQ_BIGWIG_TX_URL
            value: amqp://cenit_rabbit:cenit_rabbit@rabbitmq-cluster-ip-service/cenit_rabbit_vhost
        ports:
        - containerPort: 80
```

Deployment object is a resource of Kubernetes that can run a set of identical pods (one or more), monitors the state of each Pod and update it as necessary, among other. Our main configuration file is cenit-deployment.yml. The most important configurations inside of this yml are the image label and environment variables. The image label allows us to change the Cenit image to use, we have two versions (**_cenit:uri_** and **_cenit:local_store_**) available, the one shown in this manual is the containerized version prepared to use an external service of MongoDB. Finally, the environment variables influence the operation and configurations of each container.

>**NOTE:**
If change the container image you need to change associated environment variables too , because each container have its own environment variables.

>cenit-loadbalancer-service.yml

```yaml
apiVersion: v1
kind: Service
metadata:  
  name: cenit-loadbalancer-service
spec:
  selector:
    platform: cenit
  type: LoadBalancer
  ports:  
  - port: 80
    targetPort: 80
    protocol: TCP
```

With a LoadBalancer (this is a classic load balancer) object service we can route traffic based on the request host or path using Ingress Controllers and rules. This allows for centralization of many services to a single point and exposed to the "outside world" the contents of the Cenit image.

>**NOTE:**
After apply the changes of LoadBalancer configuration you need to use and configurate Amazon Route 53. Amazon Route 53 is a highly available and scalable cloud Domain Name System (DNS) web service. It is designed to give developers and businesses an extremely reliable and cost effective way to route end users. Amazon Route 53 effectively connects user requests to infrastructure running in AWS – such as Amazon EC2 instances, Elastic Load Balancing load balancers, or Amazon S3 buckets – and can also be used to route users to infrastructure outside of AWS.

### Https 

For this purpose we need to add a new label to our LoadBalancer configuration. We need to add: ```service.beta.kubernetes.io/aws-load-balancer-ssl-cert``` and set it value with your SSL certificate ARN. You can get the ARN from AWS Certificate Manager Console.

>cenit-loadbalancer-service-ssl.yml

```yaml
apiVersion: v1
kind: Service
metadata:  
  name: cenit-loadbalancer-service
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-ssl-cert: arn:aws:acm:eu-west-1:xxx:certificate/xxx
spec:
  selector:
    platform: cenit
  type: LoadBalancer
  ports:  
  - port: 443
    targetPort: 80
    protocol: TCP
```

If you want to listen port 80 too in the Load balancer, you must give a name to all ports and configure the config file like this:

```yaml
apiVersion: v1
kind: Service
metadata:  
  name: cenit-loadbalancer-service
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-ssl-cert: arn:aws:acm:eu-west-1:xxx:certificate/xxx
    service.beta.kubernetes.io/aws-load-balancer-ssl-ports: "443"
spec:
  selector:    
    platform: cenit
  type: LoadBalancer
  ports:  
  - name: http
    protocol: TCP
    port: 80
    targetPort: 80
  - name: https
    protocol: TCP
    port: 443
    targetPort: 80
```
