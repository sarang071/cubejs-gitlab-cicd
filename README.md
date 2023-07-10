#  CICD Pipeline using GitLab

## Brief Overview 
- Gitlab server on docker on Azure Virtual Machine/Local machine.
- Gitlab runner on AKS cluster deployed via Helm.
- Running the pipeline with variables defined.

### Create a Cloud  instance:
- If the local machine has enough RAM and other Resources, we can set up our GitLab server on it.
- But, it is preferable to have an Azure Virtual Machine with a minimum of 8 GB of RAM and 30 Gb of Storage with ubuntu.
- Allow 22,80,443 traffic in the Network Security Group for the machine and take ssh of the machine.

#### RUN:

apt-get update && apt install docker.io
service docker start
docker network create gitlab

export GITLAB_HOME=/srv/gitlab

docker run -d   --hostname <hostname> --network gitlab   --publish 443:443 --publish 80:80 --publish 4322:22   --name gitlab   --restart always   --volume $GITLAB_HOME/config:/etc/gitlab   --volume $GITLAB_HOME/logs:/var/log/gitlab   --volume $GITLAB_HOME/data:/var/opt/gitlab   --shm-size 256m   gitlab/gitlab-ce:latest

### The container can take up to 10-15 mins to get in ready and healthy. Once ready take the public ip of the machine and access the webpage with

http://<public_ip>

#### The user name to the login prompt must be as below:
 Username= root
 For Password, we need the initial root password stored in  the docker 
 #### Run :

 docker exec -it gitlab grep 'Password:' /etc/gitlab/initial_root_password

### Once Done able to login Sucessfully, do the below steps

- In order to import a project from a remote SCM like github, we need to configure the github access token with gitlab.
- On the Gitlab Dashboard

Go to Admin Area > Settings > General > Visibility and access controls > Scroll to Import sources.

Then select from the options for the import methods like github, bitbucket, etc.

From Github, generate an access token from settings and import the reposistory to gitlab.

Then select the repository that needs to be imported.


## Configuring a gitlab runner as a K8s Pod

Create an AKS cluster with the desired node count and configuration. Add the required traffic flow for the cluster in the network security group.

Configure the kube creds on the local terminal and follow the below steps.

Next if helm is not installed, we need to install helm


 - curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
 - chmod 700 get_helm.sh
 - ./get_helm.sh

### Next, in values.yml in the repository, do the below changes
1) On Line 52, add the ip address of your gitlab server.
2) Go to gitlab server and click on your project settings > CI/CD > Runners. Select new runner and copy the token and paste it on line 58.
3) On line 320 and 321, update your server ip and hostname.
4) Similarily on line 492.

### Then we need to install the gitlab runner and configure it with the gitlab project via helm.

Go to the terminal where AKS is configured.

Add the GitLab Helm repository:

- helm repo add gitlab https://charts.gitlab.io

If you are unable to access to the latest versions of GitLab Runner, you should update the chart. To update the chart, run:

- helm repo update gitlab

To view a list of GitLab Runner versions you have access to, run:

- helm search repo -l gitlab/gitlab-runner

Once you have configured GitLab Runner in your values.yaml file, run the following:

# For Helm 3
- helm install --namespace <NAMESPACE> gitlab-runner -f values.yaml gitlab/gitlab-runner

If you are facing errors with the status of runner add the below flags and install the runner

- helm install --create-namespace --namespace <NAMESPACE> gitlab-runner -f values.yaml gitlab/gitlab-runner --wait  --set livenessProbe.initialDelaySeconds="60" --set readinessProbe.initialDelaySeconds="60"


### Once the gitlab runner pod is up and running, check the status from the dashboard. 

## Gitlab Pipeline

### 1) Now in the pipeline, we first build the image with kaniko executor and push the image to the ACR repository.

### 2) Next, we add the pre-requisites like disk driver and nginx ingress controller to the cluster via set up stage

### 3) Then we apply the manifest files in the cluster in the deploy stage.

### 4) The last stage is the clean-up stage which will require a manual trigger which will destory the resoruces created via manifest files.


## Variables

In the project settings, we need to add variables which would hide our credentials in the code. Below are the list of the required variables.

Create a service principal in azure which will be logging into the cloud and performing the tasks.

Go to Azure Dashboard > Azure Active Directory > App Registerations > Create (or use existig)

Generate password and logging creds for the service principal and add the below variables.

- ACR_LOGIN_SERVER = Login Server Name of the ACR
- ACR_USERNAME = Username of the Azure Container Registery
- ACR_PASSWORD = Password of the ACR
- ACR_REPO = Azure Container Registery Name / Repo Name>#Example cdexgitlab.azurecr.io/cubejs
- AZURE_SP_APP_ID = Application (client) ID of the Azure Service Principal
- AZURE_SP_PASSWORD = Password of the Azure Service Principal
- TENANT_ID = Tenant ID
- RESOURCE_GROUP_NAME = Azure Resource Group Name for the cluster
- CLUSTER_NAME = Name of the AKS Cluster


## Once the repository is commited the pipeline will run and jobs will be executed as per the .gitlab-ci.yml file.

## Once all jobs are successful, the cleanup job will be awaiting manual trigger for cleaning up all the resources.
