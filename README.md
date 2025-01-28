# Devops-Project
In this project we will deploy the monitoring application in EKS cluster.
The monitoring application is written in python using the flask frame work. 

**Steps to perform the project:**

***Step 1.***

We need to develop python application which flask framework.necessary dependencies need to be installed using requirement.txt and monitoring template in index.html

**Installing the depencies:**
pip3 install -r requirements.txt

**Running the application:**
python3 app.py

app.py :

```
import psutil
from flask import Flask , render_template
app = Flask(__name__)
@app.route("/")
def index():
    cpu_percent = psutil.cpu_percent()
    mem_percent = psutil.virtual_memory().percent
    Message = None
    if cpu_percent > 80 or mem_percent > 80 :
        Message = "High CPU or memory utilization detected ,please scale up"
    return render_template("index.html", cpu_metric= cpu_percent, mem_metric= mem_percent, message=Message)
if __name__ ==  '__main__' :
    app.run(debug=True , host='0.0.0.0')
```    



The application is running in the localmachine,since this a python application it will run on port 5000.
We can access the application as http://localhost:5000


**_Step 2._**

We need to dockerize the application :

**1. creating the docker file :**

```
FROM python:3.9-slim-buster

WORKDIR /app

COPY . .

RUN pip3 install --no-cache-dir -r requirements.txt

ENV FLASK_RUN_HOST=0.0.0.0

EXPOSE 5000

CMD ["flask", "run"]

```

we need to build the above dockerfile to create a docker image as below 

**docker build -t flask-app .**

we can create container from the image as below 

**docker run -p 5000:5000 <image-id>**

we can able to access the monitoring application from local machine http://localhost:5000 


<img width="1728" alt="Screenshot 2025-01-24 at 10 22 21" src="https://github.com/user-attachments/assets/4e72c106-4783-4997-84d0-7ba841d793ff" />


**_Step 3._**

Pushing the docker image to container registry , in this case we have chosen Amazon Container Registry and for that we need to create container registry.

**Creation of Amazon container Registry :**

Through aws-cli we can create AWS ECR as below :

 aws ecr create-repository  --repository-name monitoring-eks  --region us-east-1 

Once ECR created we need to authenicate from aws-cli as below :

**aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin <repositoryUri>**

After successful authentication we need to tag with repository-uri 
**docker tag image:tag <repositoryUri>/image:tag**

The push the image to AWS repository :
**docker push <repositoryUri>/image:tag**

_**Step 4.**_
 
In this step we will create a eks cluster using eksctl command line and fargate
 
Creating cluster in EKS using eksctl :

**eksctl create cluster --name monitoring-cluster --region us-east-1 --fargate**

Downloading configuaration file :

**aws eks update-kubeconfig --name monitoring-cluster --region us-east-1**

Creating fargate profiles :
**eksctl create fargateprofile --cluster monitoring-cluster --name monitoring-app --namespace monitoring**

Deplyong through manifest file :

```
---
apiVersion: v1
kind: Namespace
metadata:
  name: monitoring

---
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: monitoring
  name: monitoring-deploy
  namespace: monitoring
spec:
  replicas: 1
  selector:
    matchLabels:
      app: monitoring
  template:
    metadata:
      labels:
        app: monitoring
    spec:
      containers:
      - image: 713881788228.dkr.ecr.us-east-1.amazonaws.com/monitoring-eks:latest
        name: monitoring-eks
---
apiVersion: v1
kind: Service
metadata:
  labels:
    app: monitoring
  name: monitoring-deploy
  namespace: monitoring
spec:
  ports:
  - port: 5000
    protocol: TCP
    targetPort: 5000
  selector:
    app: monitoring
  type: NodePort
```
this deployment need to be deployed as 
kubectl apply -f deployment.yaml

_**Step 5.**_

creating IAM role and policy and attaching the same to the cluster
downloading the policy.json:

```
eksctl utils associate-iam-oidc-provider --region=us-east-1 --cluster=monitoring-cluster --approve
```

```
eksctl create iamserviceaccount \
  --cluster=monitoring-cluster \
  --namespace=kube-system \
  --name=aws-load-balancer-controller \
  --role-name AmazonEKSLoadBalancerControllerRole \
  --attach-policy-arn=arn:aws:iam::713881788228:policy/AWSLoadBalancerControllerIAMPolicy \
  --approve
```
```
curl -O https://raw.githubusercontent.com/aws-samples/amazon-eks-fluent-logging-examples/mainline/examples/fargate/cloudwatchlogs/permissions.json
```
aws iam create-policy --policy-name eks-fargate-logging-policy --policy-document file://permissions.json
aws iam attach-role-policy \
  --policy-arn arn:aws:iam::713881788228:policy/eks-fargate-logging-policy \
  --role-name AmazonEKSFargatePodExecutionRole

pod-execution-role-trust-policy.json

{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Condition": {
         "ArnLike": {
            "aws:SourceArn": "arn:aws:eks:us-east-1:713881788228:fargateprofile/monitoring-cluster/*"
         }
      },
      "Principal": {
        "Service": "eks-fargate-pods.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}


aws iam create-role \
  --role-name AmazonEKSFargatePodExecutionRole \
  --assume-role-policy-document file://"pod-execution-role-trust-policy.json"

aws iam attach-role-policy \
  --policy-arn arn:aws:iam::aws:policy/AmazonEKSFargatePodExecutionRolePolicy \
  --role-name AmazonEKSFargatePodExecutionRole  

Step 6.

Creating configmaps
1.creating namespace
   **aws-observability-namespace.yaml**
   kind: Namespace
   apiVersion: v1
   metadata:
     name: aws-observability
     labels:
       aws-observability: enabled
**aws-logging-cloudwatch-configmap.yaml**
kind: ConfigMap
apiVersion: v1
metadata:
  name: aws-logging
  namespace: aws-observability
data:
  flb_log_cw: "false"  # Set to true to ship Fluent Bit process logs to CloudWatch.
  filters.conf: |
    [FILTER]
        Name parser
        Match *
        Key_name log
        Parser crio
    [FILTER]
        Name kubernetes
        Match kube.*
        Merge_Log On
        Keep_Log Off
        Buffer_Size 0
        Kube_Meta_Cache_TTL 300s
  output.conf: |
    [OUTPUT]
        Name cloudwatch_logs
        Match   kube.*
        region us-east-1
        log_group_name my-logs
        log_stream_prefix from-fluent-bit-
        log_retention_days 60
        auto_create_group true
  parsers.conf: |
    [PARSER]
        Name crio
        Format Regex
        Regex ^(?<time>[^ ]+) (?<stream>stdout|stderr) (?<logtag>P|F) (?<log>.*)$
        Time_Key    time
        Time_Format %Y-%m-%dT%H:%M:%S.%L%z

     ```   
  
