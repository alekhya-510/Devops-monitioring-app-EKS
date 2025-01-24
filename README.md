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
