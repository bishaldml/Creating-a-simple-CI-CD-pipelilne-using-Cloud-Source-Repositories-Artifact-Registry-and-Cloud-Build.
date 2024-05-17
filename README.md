# Creating-a-simple-CI-CD-pipelilne-using-Cloud-Source-Repositories-Artifact-Registry-and-Cloud-Build.
Implementing DevOps Workflow in Google Cloud.

## Challenge's:
1. Creating a GKE Cluster based on a set of configurations needed.
2. Creating a Google Source Repository to host our Go-Application Code.
3. Creating Cloud Build Triggers that deploy a production and development application.
4. Pushing updates to the app and creating a new builds.
5. Rolling back the production application to a previous version.

## Task-1: Creating the lab resources.
1. Enabling API's for GKE, Cloud Build and Cloud Source Repositories:
```
gcloud services enable container.googleapis.com \
    cloudbuild.googleapis.com \
    sourcerepo.googleapis.com
```
2. Adding K8s Developer role for Cloud Build Service account:
```
export PROJECT_ID=$(gcloud config get-value project)

gcloud projects add-iam-policy-binding $PROJECT_ID \
--member=serviceAccount:$(gcloud projects describe $PROJECT_ID \
--format="value(projectNumber)")@cloudbuild.gserviceaccount.com --role="roles/container.developer"
```
3. Configuring Git in Cloud Shell:
```
git config --global user.email <email>
git config --global user.name <name>
```
4. Creating Artifact Registry Docker Registry in your region to store our container images:
```
glcoud artifacts repositories create <repo-name> \
--repository-format=docker \
--location=<your-region>
```
5. Creating a GKE Stadard Cluster named hello-cluster with following configurations:
```
gcloud container clusters create hello-cluster \
--zone <your-region>
--release-channel regular \
--cluster-version latest \
--enable-autoscaling \
--num-nodes 3 \
--min-nodes 2 \
--max-nodes 6
```
6. Create prod and dev namespaces on the cluster:
```
kubectl create namespace prod
kubectl create namespace dev
```
## Task-2: Create a repository in Cloud Source Repositories.
1. Creating a empty repository in Cloud Source Repository:
```
gcloud source repos create <repo-name>
```
2. Cloning the sample-app Cloud Source Repository in Cloud Shell.
```
git clone <url-of-repository>
```
3. Copying the sample code of Go-Application into our sample-app directory.
```
cd sample-app
gsutil cp -r gs://spls/gsp330/sample-app/* .
```
4. Automatically replacing the <your-region> and <your-zone> placeholders in the cloudbuild-dev.yaml and cloudbuild.yaml files with the assigned region and zone of our project.
```
export REGION="your-region"
export ZONE="your-zone"

for file in sample-app/cloudbuild-dev.yaml sample-app/cloudbuild.yaml; do
  sed -i 's/<your_region>/${REGION}/g' "$file"
  sed -i 's/<your_zone>/${ZONE}/g' "$file"
done
```

```
### *Note: sed syntax
1. sed -i 's/search_text/replacement/FLAGS' FILE
i = in-place
FLAGS: g = global replacement
       n = replace "n" occurance placeholder
s = substitute
```
5. Make your first commit and push the changes to master branch.
```
git init
git add .
git commit -m "msg-1"
git push -u origin master
```
6. Creating new_branch name 'dev' and pushing the changes to dev branch.
```
git branch dev
git checkout dev
git push -u origin dev
```
## Task-3: Create Cloud Build Triggers
