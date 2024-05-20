# Creating-a-simple-CI-CD-pipeline-using-Cloud-Source-Repositories-Artifact-Registry-and-Cloud-Build.
#### Implementing DevOps Workflow in Google Cloud.

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
export PROJECT_ID=
```
OR 
```
export PROJECT_ID=$(gcloud config get-value project)
```

```
gcloud projects add-iam-policy-binding $PROJECT_ID \
--member=serviceAccount:$(gcloud projects describe $PROJECT_ID \
--format="value(projectNumber)")@cloudbuild.gserviceaccount.com --role="roles/container.developer"
```
3. Configuring Git in Cloud Shell:
```
export EMAIL=
export NAME=
```
And
```
git config --global user.email $EMAIL
git config --global user.name $NAME
```
4. Creating Artifact Registry Docker Registry in your region to store our container images:
```
export ARTIFACT_REGISTRY=
export REGION=
```
And
```
gcloud artifacts repositories create $ARTIFACT_REGISTRY \
    --repository-format=docker \
    --location=$REGION \
    --description="Creating Artifact Registry (Docker)repository"
```
5. Creating a GKE Stadard Cluster named hello-cluster with following configurations:
```
export CLUSTER_NAME=
export ZONE=
```
And
```
gcloud beta container --project "$PROJECT_ID" clusters create "$CLUSTER_NAME" --zone "$ZONE" --no-enable-basic-auth --cluster-version latest --release-channel "regular" --machine-type "e2-medium" --image-type "COS_CONTAINERD" --disk-type "pd-balanced" --disk-size "100" --metadata disable-legacy-endpoints=true  --logging=SYSTEM,WORKLOAD --monitoring=SYSTEM --enable-ip-alias --network "projects/$PROJECT_ID/global/networks/default" --subnetwork "projects/$PROJECT_ID/regions/$REGION/subnetworks/default" --no-enable-intra-node-visibility --default-max-pods-per-node "110" --enable-autoscaling --min-nodes "2" --max-nodes "6" --location-policy "BALANCED" --no-enable-master-authorized-networks --addons HorizontalPodAutoscaling,HttpLoadBalancing,GcePersistentDiskCsiDriver --enable-autoupgrade --enable-autorepair --max-surge-upgrade 1 --max-unavailable-upgrade 0 --enable-shielded-nodes --node-locations "$ZONE"
```
6. Create prod and dev namespaces on the cluster:
```
kubectl create namespace prod
kubectl create namespace dev
```
## Task-2: Create a repository in Cloud Source Repositories.
1. Creating a empty repository in Cloud Source Repository:
```
export REPOSITORY=
```
And
```
gcloud source repos create $REPOSITORY
```
2. Cloning the sample-app Cloud Source Repository in Cloud Shell.
```
git clone https://source.developers.google.com/p/$PROJECT_ID/r/sample-app
```
3. Copying the sample code of Go-Application into our sample-app directory.
```
cd sample-app
gsutil cp -r gs://spls/gsp330/sample-app/* .
```
4. Automatically replacing the <your-region> and <your-zone> placeholders in the cloudbuild-dev.yaml and cloudbuild.yaml files with the assigned region and zone of our project.
```
export REGION=
export ZONE=
```
And
```
for file in sample-app/cloudbuild-dev.yaml sample-app/cloudbuild.yaml; do
    sed -i "s/<your-region>/${REGION}/g" "$file"
    sed -i "s/<your-zone>/${ZONE}/g" "$file"
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
-> The first trigger listens for changes on the master branch and builds a Docker image of our application, pushes it to Google Artifact Registry and deploys the latest version of the image to the prod namespace in our GKE cluster.

-> Second trigger is for dev branch to deploy the latest version image to dev namespace.

1. Creating Cloud Build Trigger named sample-app-prod-deploy with below configurations:
```
1. Event: Push to a branch
2. Source Repository: sample-app
3. Branch: ^master$
4. Cloud Build Configuration File: cloudbuild.yaml
```
2. Creating sample-app-dev-deploy Cloud Build Trigger with configurations:
```
1. Event: Push to a branch
2. Source Repository: sample-app
3. Branch: ^dev$
4. Cloud Build Configuration File: cloudbuild-dev.yaml
```
#### *Note: After setting up the triggers, any changes to the branches triggers the corresponding Cloud Build Pipeline, which builds and deploy the application as specified in the cloudbuild.yaml files.

## Task-4: Deploy the first version of the application.

### Build the First development deployment.
1. In cloud shell, inspect the cloudbuild-dev.yaml file and replace the <version> on lines 9 and 13 with v1.0
2. Navigate to the dev/deployment.yaml file and update the <todo> on line 17 with the correct container image name.
#### *Note: Make sure you have same container image name in cloudbuild-dev.yaml file and dev/deployment.yaml file.

3. Make a commit with your changes on the dev branch and push changes to trigger the sample-app-dev-deploy build job:
```
git add .
git commit -m "First development deployment of First version"
git push -u origin dev
```
4. Verify your build executed successfully in Cloud Build History page, and verify the development-deployment application was deployed onto the dev namespace of the cluster.
5. Expose the development-deployment deployment to a LoadBalancer service named dev-deployment-service on port 8080, and set the target port of the container to the one specified in the Dockerfile.
6. Navigate to the LoadBalance IP of the service and add the /blue entry point of the end of the URL to verify the application is up and running. It should resemble samething like the following: http://34.135.97.199:8080/blue.

### Build the First production deployment.

1. Switch to the master branch. Inspect cloudbuild.yaml located in sample-app and replace the <version> on lines 11 and 16 with v1.0:
```
git checkout master
git branch
```
2. Navigate to the pod/deployment.yaml file and update the <todo> on line 17 with the correct container image name.
3. Make a commit with your changes on the master branch and push changes to trigger the sample-app-prod-deploy build job:
```
git add .
git commit -m "First production deployment of first version"
git push -u origin master
```
4. Verify the build executed successfully in cloud history page, and verify the production-deployment application was deployed onto the prod namespace of the cluster.
5. Expose the production-deployment on the prod namespace to a loadBalancer service named prod-deployment-service on port 8080, to the one specified in the Dockerfile.
6. Navigate to the LoadBalancer IP of the service and add the /blue entry point of the end of the URL to verify the application is up and running. It should resemble something like the following: http://34.135.245.19:8080/blue

## Task-5: Deploy the second version of the application.
### Build the Second development deployment.
1. Switch back to the dev branch.
```
git checkout dev
git branch
```
2. In the main.go file, update the main() function to the following:
```
func main() {
	http.HandleFunc("/blue", blueHandler)
	http.HandleFunc("/red", redHandler)
	http.ListenAndServe(":8080", nil)
}
```
3. Add the following function inside of the main.go file:
```
func redHandler(w http.ResponseWriter, r *http.Request) {
	img := image.NewRGBA(image.Rect(0, 0, 100, 100))
	draw.Draw(img, img.Bounds(), &image.Uniform{color.RGBA{255, 0, 0, 255}}, image.ZP, draw.Src)
	w.Header().Set("Content-Type", "image/png")
	png.Encode(w, img)
}
```
4. Inspect the cloudbuild-dev.yaml file to see the steps in the build process. Update the version of the Docker image to v2.0.
5. Navigate to the dev/deployment.yaml file and update the container image name to the new version (v2.0).
6. Make a commit with your changes on the dev branch and push changes to trigger the sample-app-dev-deploy build job.
```
git add .
git commit -m "First development deployment of second version" 
git push -u origin dev
```
7. Verify your build executed successfully in Cloud build History page, and verify the development-deployment application was deployed onto the dev namespace of the cluster and is using the v2.0 image.
8. Navigate to the Load Balancer IP of the service and add the /red entry point at the end of the URL to verify the application is up and running. It should resemble something like the following: http://34.135.97.199:8080/red.

### Build the Second production deployment
1. Switch to the master branch.
```
git checkout master
git branch
```
2. In the main.go file, update the main() function to the following:
```
func main() {
	http.HandleFunc("/blue", blueHandler)
	http.HandleFunc("/red", redHandler)
	http.ListenAndServe(":8080", nil)
}
```
3. Add the following function inside of the main.go file:
```
func redHandler(w http.ResponseWriter, r *http.Request) {
	img := image.NewRGBA(image.Rect(0, 0, 100, 100))
	draw.Draw(img, img.Bounds(), &image.Uniform{color.RGBA{255, 0, 0, 255}}, image.ZP, draw.Src)
	w.Header().Set("Content-Type", "image/png")
	png.Encode(w, img)
}
```
4. Inspect the cloudbuild.yaml file to see the steps in the build process. Update the version of the Docker image to v2.0.
5. Navigate to the prod/deployment.yaml file and update the container image name to the new version (v2.0).
6. Make a commit with your changes on the master branch and push changes to trigger the sample-app-prod-deploy build job.
```
git add .
git commit -m "Second production deployment of second version" 
git push -u origin master
```
7. Verify your build executed successfully in Cloud build History page, and verify the production-deployment application was deployed onto the prod namespace of the cluster and is using the v2.0 image.
8. Navigate to the Load Balancer IP of the service and add the /red entry point at the end of the URL to verify the application is up and running. It should resemble something like the following: http://34.135.245.19:8080/red.

# Task-6: Roll back the production deployment
1. Roll back the production-deployment to use the v1.0 version of the application.
```
In the Cloud console, go to Cloud Build > Dashboard.
Click on View all link under Build History for the sample-app-prod-deploy trigger.
Click on the second most recent build available (or the v1.0 build that executed successfully)
Click Rebuild.
```
Other method to roll back using kubectl tool:
Run the following command to roll back the production deployment:
```
kubectl -n prod rollout undo deployment/production-deployment
```
Run the following command to check version status:
```
kubectl -n prod get pods -o jsonpath --template='{range .items[*]}{.metadata.name}{"\t"}{"\t"}{.spec.containers[0].image}{"\n"}{end}'
```
2. Navigate to the Load Balancer IP of the service and add the /red entry point at the end of the URL of the production deployment and response on the page should be 404.
