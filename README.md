### Jenkins Pipeline for Simple Todo Python based application using SonarQube, Argo CD, Helm and Kubernetes

## CICD Architecture [GitHub -> Jenkins -> k8s Manifests -> Argo CD -> k8s cluster]

![screenshot_3](https://github.com/keerthanav-19/jenkins-project/blob/main/images/screenshot_3.png)

Here are the step-by-step details to set up an end-to-end Jenkins pipeline for a Python application using SonarQube, Argo CD, Helm, and Kubernetes:

Prerequisites:

   -  Python application code hosted on a Git repository
   -  Jenkins server
   -  SonarQube server
   -  Kubernetes cluster
   -  Argo CD

### Jenkins Pipeline Stages

![screenshot_2](https://github.com/keerthanav-19/jenkins-project/blob/main/images/screenshot_2.PNG)

### Explanation of each stage in the Jenkins pipeline script
Checkout: Checks out the source code from a GitHub repository.

Build and Test: Install dependencies, runs tests, and ensures the Python application's functionality.

Sonarqube Analysis: Analyzes the code quality using SonarQube, including static code analysis and code coverage.

Build and Push Docker Image - Python App: Builds and pushes the Docker image for the Python application to a Docker registry.

Deployment - Python App: Manages the deployment of the Python application to a Kubernetes cluster. It checks if the deployment exists and updates it if necessary. If the deployment does not exist, it creates a new deployment using ArgoCD.

### To install ArgoCD using Helm, follow these steps:
Add ArgoCD Helm Repository
```bash
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update
```
Install ArgoCD
```bash
kubectl create namespace argocd
helm install argocd argo/argo-cd --namespace argocd
```
After the installation completes, you can access the ArgoCD UI using a port-forwarding command
```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
```
Then, open a web browser and go to http://localhost:8080 to access the ArgoCD UI

Log in to ArgoCD using the default username and password. Retrieve the default password using the following command
```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```

A simple todo app project 

![screenshot_5](https://github.com/keerthanav-19/jenkins-project/blob/main/images/screenshot_5.PNG)

### Execute locally and access the application
Make sure to install the dependencies of the project through the requirements.txt file.
```bash
pip install -r requirements.txt
```

Once you have installed django and other packages, go to the cloned repo directory and run the following command
```bash
python manage.py makemigrations
```

This will create all the migrations file (database migrations) required to run this App.Now, to apply this migrations run the following command
```bash
python manage.py migrate
```

### options
Project it self has the user creation form but still in order to use the admin you need to create a super user.you can use the createsuperuser option to make a super user. And lastly let's make the App run. We just need to start the server now and then we can start using our simple todo App. Start the server by following command
```bash
python manage.py createsuperuser
python manage.py runserver
```

Once the server is up and running, head over to http://127.0.0.1:8000 for the App.