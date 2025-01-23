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
   
