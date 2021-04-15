#GitHub Actions deploy Flask to Azure Container Registry and Azure Container Instances

Status of Last Deployment:<br>

<img src="https://github.com/kiselev-it/flask-container-action/workflows/Docker-Image-CI/CD/badge.svg?branch=main"><br>

This demo shows how to build a Docker container from a Python Flask app, push it to an Azure Container Registry, and then run the container on an Azure Container Instance.

## Pre-requisites
- Created a free account Azure
- Created a resource group (Azure services)
- An Azure Container Repository resource created in the resource group
- Enabled the Admin User Login/Password in the Azure Container Registry

## Step 1: Create a simple Flask app
app.py
```sh
from flask import Flask

app = Flask(__name__)

@app.route('/')
def hello():
    return "Hello World!"

if __name__ == '__main__':
    app.run(host="0.0.0.0", port=80)
```
## Step 2: Create a simple Unit test
test_application.py 
```sh
from app import app
with app.test_client() as c:
    response = c.get('/')
    assert response.data == b'Hello World!'
    assert response.status_code == 200
```
## Step 3: Create a Dockerfile
The Dockerfile work for on Ubuntu.
Dockerfile
```sh
FROM ubuntu:16.04
RUN apt-get update -y
RUN apt-get install -y python-pip python-dev build-essential
COPY . /app
WORKDIR /app
RUN pip install -r requirements.txt
ENTRYPOINT ["python"]
CMD ["app.py"]
```

## Step 4: Create a Service Principal in Azure
Used this comand in the Azure portal from the Cloud Shell (Bash).
```sh
az ad sp create-for-rbac --name "rc-az-action" --sdk-auth --role contributor --scopes /subscriptions/xxx-xxx-xxx-xxx-xxx/resourceGroups/Demo
```

`az ad sp create-for-rbac --name "rc-az-action" --sdk-auth --role contributor --scopes /subscriptions/xxx-xxx-xxx-xxx-xxx/resourceGroups/Demo`

Here, the **create-for-rbac** command is used to create a Service Principal called rc-az-action, and it's granted **contributor** rights on the scope of the **Demo** Resource Group in the subscription specified. Replace the "xxx.." with your actual subscription id.

Copy the resulting JSON response, and save it in a GitHub secret in your repository, as shown below:

![JSON Response](https://github.com/marlinspike/flask-container-action/blob/master/img/JSON_Response.png)

![GitHub Secret](https://github.com/marlinspike/flask-container-action/blob/master/img/create_secret.jpg)

The **Demo** Resource Group is where I've created the Azure Container Repository.

## Step 6: Create GitHub Secrets
GitHub Secrets to create:
- "AZ_CREDS": JSON Response from step Above
- "REGISTRY_PASSWORD": The Password for the Azure Container Registry
- "REGISTRY_USERNAME": The Username for the Azure Container Registry

## Step 7: Create the GitHub Action YAML Script
The script supplied in the workflows folder should work if you've followed the directions accurately.
