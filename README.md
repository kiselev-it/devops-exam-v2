# GitHub Actions deploy Flask to Azure Container Registry and Azure Container Instances

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
Saved the resulting Json. It will need in the next step.

## Step 5: Create GitHub Secrets
GitHub Secrets (Settings > Secrets > New Repository secret):
- "AZ_CREDS": JSON Response from step above
- "REGISTRY_PASSWORD": The Password for the Azure Container Registry
- "REGISTRY_USERNAME": The Username for the Azure Container Registry

## Step 6: Create the GitHub Action YAML Script
The YAML script created in the /workflow folder. 
```sh
name: Docker-Image-CI/CD

env:
  IMAGE_NAME: flask-container-action
  IMAGE_TAG: latest

#on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

jobs:

  build:
    runs-on: ubuntu-16.04
    steps:
    - uses: azure/login@v1
      with:
        creds: ${{ secrets.AZ_CREDS }}
    
    - uses: actions/checkout@v2
      
    - name: Set up Python 3.6
      uses: actions/setup-python@v1
      with:
          python-version: "3.6"
          
    - name: Install dependencies
      run: |
        python -m pip install --upgrade pip
        pip install -r requirements.txt
          
    - name: Run unit tests
      run: python test_application.py
    
    - uses: azure/docker-login@v1
      with:
        login-server: rcdemocontainerreg.azurecr.io 
        username: ${{ secrets.REGISTRY_USERNAME }}
        password: ${{ secrets.REGISTRY_PASSWORD }}
    - run:
        docker build . -t rcdemocontainerreg.azurecr.io/rcapps/flask-container-action:latest
    - run:
        docker push rcdemocontainerreg.azurecr.io/rcapps/flask-container-action:latest

    - name: 'Deploy to Azure Container Instances'
      uses: 'azure/aci-deploy@v1'
      with:
        resource-group: Demo
        dns-name-label: flaskdemo
        image: rcdemocontainerreg.azurecr.io/rcapps/flask-container-action:latest
        registry-login-server: rcdemocontainerreg.azurecr.io
        registry-username: ${{ secrets.REGISTRY_USERNAME }}
        registry-password: ${{ secrets.REGISTRY_PASSWORD }}
        name: democi
        location: 'east us'
```
