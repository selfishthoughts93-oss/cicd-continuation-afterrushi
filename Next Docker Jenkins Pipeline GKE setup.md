# GKE Lab -- Kubernetes Deployment and Jenkins CI/CD Integration

## 1. Lab Objective

Create a Google Kubernetes Engine (GKE) cluster, deploy a Flipkart demo
application from Docker Hub, expose it through a LoadBalancer, and
integrate deployment into the existing Jenkins CI/CD pipeline.

### Tools Used

-   Google Cloud Platform (GCP) and Google Kubernetes Engine (GKE)
-   Google Cloud CLI (`gcloud`), Kubernetes CLI (`kubectl`), and GKE
    authentication plugin
-   Jenkins, GitHub, Maven, Tomcat, Docker, and Docker Hub

### Lab Configuration

  Component               Value
  ----------------------- -----------------------------------------------
  GCP Project ID          `august2027`
  GKE Cluster             `rushi-gke`
  GKE Zone                `us-central1-a`
  Jenkins Pipeline        `cicd-pipeline2`
  Kubernetes Deployment   `flipkart-app`
  Kubernetes Service      `flipkart-service`
  Replicas                2
  Container Port          8080
  Service Port            80
  Docker Image            `devopsbyrushi/octsecurebankcicdpipleline:v1`

## 2. Install Prerequisites on the Jenkins Server

The Jenkins server in this lab uses Debian Linux. These tools were
already installed and verified on our Jenkins VM; the following steps
are included for reference when preparing a new server.

### Step 1: Connect to the Jenkins VM

Google Cloud Console → **Compute Engine → VM instances → jenkins →
SSH**.

Run these commands in the Jenkins VM terminal, not in Cloud Shell or
inside a Kubernetes Pod.

### Step 2: Update packages and install prerequisites

``` bash
sudo apt-get update
sudo apt-get install -y ca-certificates curl gnupg apt-transport-https
```

### Step 3: Add the Google Cloud CLI package repository

``` bash
sudo mkdir -p /usr/share/keyrings
curl -fsSL https://packages.cloud.google.com/apt/doc/apt-key.gpg \
  | sudo gpg --dearmor \
  -o /usr/share/keyrings/cloud.google.gpg
```

``` bash
echo "deb [signed-by=/usr/share/keyrings/cloud.google.gpg] https://packages.cloud.google.com/apt cloud-sdk main" \
  | sudo tee /etc/apt/sources.list.d/google-cloud-sdk.list
sudo apt-get update
```

### Step 4: Install Google Cloud CLI

``` bash
sudo apt-get install -y google-cloud-cli
gcloud --version
```

### Step 5: Install kubectl

``` bash
sudo apt-get install -y kubectl
kubectl version --client
```

### Step 6: Install the GKE authentication plugin

``` bash
sudo apt-get install -y google-cloud-cli-gke-gcloud-auth-plugin
gke-gcloud-auth-plugin --version
```

### Step 7: Verify all three tools

``` bash
gcloud --version
kubectl version --client
gke-gcloud-auth-plugin --version
```

**Lab result:** All three tools were already installed on the Jenkins
server and the version commands completed successfully.

Reference: [Google Cloud CLI installation
documentation](https://cloud.google.com/sdk/docs/install)

## 3. Set the GCP Project and Create the GKE Cluster

Open **Google Cloud Console → Cloud Shell** for this section.

### Step 1: Set the project

``` bash
gcloud config set project august2027
```

### Step 2: Enable the Kubernetes Engine API

``` bash
gcloud services enable container.googleapis.com
```

### Step 3: Create the cluster

``` bash
gcloud container clusters create rushi-gke \
  --project=august2027 \
  --zone=us-central1-a \
  --release-channel=regular \
  --num-nodes=2 \
  --machine-type=e2-medium \
  --disk-type=pd-balanced \
  --disk-size=30 \
  --enable-ip-alias
```

Wait for cluster creation to finish.

### Step 4: Connect kubectl to the cluster

``` bash
gcloud container clusters get-credentials rushi-gke \
  --project=august2027 \
  --zone=us-central1-a
```

### Step 5: Verify the cluster

``` bash
kubectl cluster-info
kubectl get nodes
```

Expected result: two Kubernetes nodes.

## 4. Create a Standalone NGINX Pod

### Step 1: Create the Pod

``` bash
kubectl run nginx-pod --image=nginx:latest
```

### Step 2: Check Pod status and details

``` bash
kubectl get pods
kubectl get pods -o wide
kubectl describe pod nginx-pod
kubectl logs nginx-pod
```

### Step 3: Delete the Pod

``` bash
kubectl delete pod nginx-pod
kubectl get pods
```

**Key concept:** A standalone Pod created with `kubectl run` is not
automatically recreated after deletion. A Deployment maintains the
desired number of replicas.

## 5. Deploy the Flipkart Application to Kubernetes

Image used:

`docker.io/devopsbyrushi/octsecurebankcicdpipleline:v1`

### Step 1: Create the Deployment

``` bash
kubectl apply -f - <<'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: flipkart-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: flipkart-app
  template:
    metadata:
      labels:
        app: flipkart-app
    spec:
      containers:
        - name: octsecurebankcicdpipeline
          image: docker.io/devopsbyrushi/octsecurebankcicdpipleline:v1
          imagePullPolicy: Always
          ports:
            - name: http
              containerPort: 8080
EOF
```

### Step 2: Verify the Deployment and Pods

``` bash
kubectl get deployments
kubectl get pods -l app=flipkart-app -o wide
kubectl rollout status deployment/flipkart-app
```

Expected result: two application replicas become ready.

**Important:** The application must actually listen on container port
`8080`. Declaring `containerPort: 8080` does not change the
application's listening port.

## 6. Expose the Application Using a LoadBalancer

The public Service listens on port `80` and forwards traffic to
container port `8080`.

### Step 1: Create the Service

``` bash
kubectl expose deployment flipkart-app \
  --name=flipkart-service \
  --type=LoadBalancer \
  --port=80 \
  --target-port=8080
```

If the Service already exists, do not create it again; continue to
verification.

### Step 2: Check the Service and endpoints

``` bash
kubectl get svc flipkart-service
kubectl get endpoints flipkart-service
```

### Step 3: Wait for the External IP

``` bash
kubectl get svc flipkart-service --watch
```

Press `Ctrl+C` after the External IP appears.

### Step 4: Open the application in Chrome

Use the current External IP shown by the Service:

``` text
http://EXTERNAL_IP
```

The IP shown in our lab was `35.254.166.70`; yours may differ. Do not
add `:8080` to the browser URL because the public Service uses port
`80`.

## 7. Demonstrate Pod Self-Healing and Scaling

### Step 1: List application Pods

``` bash
kubectl get pods -l app=flipkart-app
```

### Step 2: Delete one Pod

Replace `POD_NAME` with the actual Pod name:

``` bash
kubectl delete pod POD_NAME
```

### Step 3: Watch Kubernetes recreate it

``` bash
kubectl get pods -l app=flipkart-app --watch
```

Press `Ctrl+C` after observing the replacement.

### Step 4: Scale to four replicas

``` bash
kubectl scale deployment flipkart-app --replicas=4
kubectl get pods -l app=flipkart-app
```

### Step 5: Scale back to two replicas

``` bash
kubectl scale deployment flipkart-app --replicas=2
kubectl get deployments
kubectl get pods -l app=flipkart-app
```

## 8. Configure GCP Authentication for Jenkins

These steps configure Jenkins to authenticate with GCP and deploy to
GKE.

### Step 1: Create a service account

Run in Google Cloud Shell:

``` bash
gcloud config set project august2027
gcloud iam service-accounts create jenkins-gke-deployer \
  --display-name="Jenkins GKE Deployer"
```

If this service account already exists, do not create it again.

### Step 2: Grant IAM permissions

``` bash
gcloud projects add-iam-policy-binding august2027 \
  --member="serviceAccount:jenkins-gke-deployer@august2027.iam.gserviceaccount.com" \
  --role="roles/container.developer"
```

``` bash
gcloud projects add-iam-policy-binding august2027 \
  --member="serviceAccount:jenkins-gke-deployer@august2027.iam.gserviceaccount.com" \
  --role="roles/container.clusterViewer"
```

Additional Kubernetes RBAC permissions may be required depending on the
cluster configuration.

### Step 3: Create the service account key

``` bash
gcloud iam service-accounts keys create jenkins-gke-key.json \
  --iam-account="jenkins-gke-deployer@august2027.iam.gserviceaccount.com"
```

Download `jenkins-gke-key.json` from Cloud Shell to your computer.

**Security warning:** This JSON file contains a private credential.
Never commit it to GitHub, include it in a Docker image, or share its
contents.

### Step 4: Add the key to Jenkins Credentials

In Jenkins, navigate to **Manage Jenkins → Credentials → the appropriate
credentials store → Add Credentials**.

Set the fields:

  Field         Value
  ------------- ---------------------------------
  Kind          `Secret file`
  File          Upload `jenkins-gke-key.json`
  ID            `gcp-jenkins-key`
  Description   `GCP GKE Deployment Credential`

Save the credential. The pipeline will access it securely through
Jenkins; do not use a placeholder path such as
`/path/to/jenkins-gke-key.json` in the terminal.

## 9. Verify Jenkins Prerequisites

Connect to the Jenkins VM using **Compute Engine → VM instances →
jenkins → SSH**.

These commands were run successfully in our lab:

``` bash
gcloud --version
kubectl version --client
gke-gcloud-auth-plugin --version
```

All three tools were already installed on the Jenkins server. They must
be available on the Jenkins agent that actually executes the pipeline;
installing them only in Cloud Shell is not sufficient.

## 10. Existing Jenkins CI/CD Pipeline

The Jenkins job is `cicd-pipeline2`.

GitHub repository:

`https://github.com/devopsbyrushi/rushiDevOpsTraining-FlipkartWarDemo.git`

The original pipeline performs:

1.  GitHub checkout.
2.  Maven validation.
3.  Maven installation/build.
4.  Copy the WAR artifact to Tomcat.
5.  Build the Docker image.
6.  Log in to Docker Hub using Jenkins credentials.
7.  Push the Docker image to Docker Hub.

We added:

8.  Deploy the image to GKE.
9.  Verify the Deployment, Pods, Service, and deployed image.

The image tag in this lab is
`devopsbyrushi/octsecurebankcicdpipleline:v1`. The image repository and
tag must match in the Docker build, push, and deployment stages.

## 11. Complete Updated Jenkins Pipeline Script

Open **Jenkins → `cicd-pipeline2` → Configure → Pipeline → Script** and
replace the existing script with this version.

``` groovy
node {
    stage('GitHub Code Checkout') {
        git credentialsId: 'rushi-github-credentials',
            url: 'https://github.com/devopsbyrushi/rushiDevOpsTraining-FlipkartWarDemo.git'
    }

    stage('Maven Build - Validate') {
        sh 'sudo mvn validate'
    }

    stage('Maven Build - Install') {
        sh 'sudo mvn install'
    }

    stage('Copy Artifact to Tomcat Webapps') {
        sh '''
            set -e
            sudo cp "$WORKSPACE/target/flipkart-clone.war" \
                /opt/tomcat10/webapps/
        '''
    }

    stage('Create Docker Image') {
        sh '''
            set -e
            docker build \
                -t devopsbyrushi/octsecurebankcicdpipleline:v1 .
        '''
    }

    stage('Docker Login') {
        withCredentials([
            string(
                credentialsId: 'dockerhub-passwd',
                variable: 'rushidockerhubpassword'
            )
        ]) {
            sh '''
                set +x
                echo "$rushidockerhubpassword" |
                    docker login -u devopsbyrushi --password-stdin
            '''
        }
    }

    stage('Push Image to Docker Hub') {
        sh '''
            set -e
            docker push devopsbyrushi/octsecurebankcicdpipleline:v1
        '''
    }

    stage('Deploy to GKE') {
        withCredentials([
            file(
                credentialsId: 'gcp-jenkins-key',
                variable: 'GCP_KEY'
            )
        ]) {
            sh '''
                set -eu

                export USE_GKE_GCLOUD_AUTH_PLUGIN=True

                gcloud auth activate-service-account \
                    --key-file="$GCP_KEY"

                gcloud config set project august2027

                gcloud container clusters get-credentials rushi-gke \
                    --project=august2027 \
                    --zone=us-central1-a

                kubectl set image deployment/flipkart-app \
                    octsecurebankcicdpipeline=docker.io/devopsbyrushi/octsecurebankcicdpipleline:v1

                kubectl rollout restart deployment/flipkart-app

                kubectl rollout status deployment/flipkart-app \
                    --timeout=300s
            '''
        }
    }

    stage('Verify GKE Deployment') {
        sh '''
            set -eu

            echo "===== Deployment Status ====="
            kubectl get deployment flipkart-app

            echo "===== Application Pods ====="
            kubectl get pods -l app=flipkart-app -o wide

            echo "===== LoadBalancer Service ====="
            kubectl get service flipkart-service

            echo "===== Deployed Docker Image ====="
            kubectl get deployment flipkart-app \
                -o jsonpath='{.spec.template.spec.containers[0].image}{"\\n"}'
        '''
    }
}
```

### Pipeline prerequisites and notes

-   Deployment name: `flipkart-app`.
-   Container name: `octsecurebankcicdpipeline`.
-   GCP credential ID: `gcp-jenkins-key`.
-   Docker Hub credential ID: `dockerhub-passwd`.
-   GitHub credential ID: `rushi-github-credentials`.
-   The Jenkins agent must have permission to run Docker and required
    `sudo` commands.
-   The GCP service account must have sufficient IAM and Kubernetes
    permissions.
-   The WAR path assumes Maven creates `target/flipkart-clone.war`.
-   A suitable Dockerfile must exist in the checked-out repository.

This example reuses tag `v1` to match the existing lab. When reusing a
tag, the Deployment should use `imagePullPolicy: Always`. For
production, prefer a unique image tag for every build.

## 12. Run the Jenkins Pipeline

1.  Click **Save** in the Jenkins pipeline configuration.
2.  Open `cicd-pipeline2` and click **Build Now**.
3.  Open the latest build and select **Console Output**.
4.  Verify that all existing stages succeed, followed by `Deploy to GKE`
    and `Verify GKE Deployment`.

Expected final status:

``` text
Finished: SUCCESS
```

In our lab, Jenkins build `#33` completed successfully after the GKE
deployment and verification stages were added. The Console Output
displayed the LoadBalancer Service and deployed Docker image.

A successful Jenkins build does not guarantee that every historical Pod
has disappeared. Confirm that the current Deployment has two ready
replicas.

## 13. Verify the Application in GKE

Run these commands in Google Cloud Shell.

### Check the Deployment and Pods

``` bash
kubectl get deployment flipkart-app
kubectl get pods -l app=flipkart-app -o wide
```

Expected result: two ready application Pods.

### Check the deployed image

``` bash
kubectl get deployment flipkart-app \
  -o jsonpath='{.spec.template.spec.containers[0].image}{"\n"}'
```

### Check the LoadBalancer Service

``` bash
kubectl get svc flipkart-service
```

Open the current External IP in Chrome, for example
`http://35.254.166.70`. Use the actual IP reported by your Service if it
differs.

### Check rollout status

``` bash
kubectl rollout status deployment/flipkart-app
```

### Troubleshoot a Pod in Error

``` bash
kubectl describe pod POD_NAME
kubectl logs POD_NAME
kubectl get deployment flipkart-app
kubectl get replicasets -l app=flipkart-app
kubectl get pods -l app=flipkart-app
```

Replace `POD_NAME` with the actual Pod name. Confirm the current
Deployment has two ready replicas before considering the deployment
complete.

## 14. Delete Application Resources Without Deleting the Cluster

If you want to remove the application but keep the GKE cluster, run
these commands in Cloud Shell.

``` bash
kubectl delete service flipkart-service
kubectl delete deployment flipkart-app
kubectl delete pod nginx-pod --ignore-not-found
```

Verify:

``` bash
kubectl get pods
kubectl get deployments
kubectl get services
```

The default Kubernetes Service named `kubernetes` normally remains.
Deleting the LoadBalancer Service removes its public endpoint.

You do not need to delete application resources separately if you are
going to delete the entire cluster.

## 15. Destroy the GKE Cluster

**Warning:** Deleting the cluster destroys the GKE cluster and its
managed workloads. Make sure you no longer need the application or
cluster before proceeding.

### Step 1: Confirm the project

``` bash
gcloud config set project august2027
```

### Step 2: Verify the cluster

``` bash
gcloud container clusters list --project=august2027
```

Confirm that the cluster is `rushi-gke` in zone `us-central1-a`.

### Step 3: Delete the cluster

``` bash
gcloud container clusters delete rushi-gke \
  --project=august2027 \
  --zone=us-central1-a
```

Confirm the deletion when prompted and wait for completion.

### Step 4: Verify deletion

``` bash
gcloud container clusters list --project=august2027
```

The cluster `rushi-gke` should no longer appear.

### Step 5: Review remaining disks

``` bash
gcloud compute disks list --project=august2027
```

### Step 6: Review reserved IP addresses

``` bash
gcloud compute addresses list --project=august2027
```

Review remaining resources before deleting them; they may belong to
other workloads.

**Important:** Deleting the GKE cluster does not automatically delete
the Jenkins VM, Jenkins credentials, GCP service account, or Docker Hub
repository. Review those resources separately if they were created only
for this lab.

## 16. Important Kubernetes Commands -- Quick Revision

  -------------------------------------------------------------------------------------------
  Task                                Command
  ----------------------------------- -------------------------------------------------------
  Check nodes                         `kubectl get nodes`

  Check Pods                          `kubectl get pods`

  Check Deployments                   `kubectl get deployments`

  Check Services                      `kubectl get services`

  Describe a Pod                      `kubectl describe pod POD_NAME`

  View Pod logs                       `kubectl logs POD_NAME`

  Check rollout status                `kubectl rollout status deployment/flipkart-app`

  Scale to four replicas              `kubectl scale deployment flipkart-app --replicas=4`

  Scale to two replicas               `kubectl scale deployment flipkart-app --replicas=2`

  Check cluster information           `kubectl cluster-info`

  List GKE clusters                   `gcloud container clusters list --project=august2027`
  -------------------------------------------------------------------------------------------

## Lab Conclusion

In this lab, we:

1.  Verified Google Cloud CLI, `kubectl`, and the GKE authentication
    plugin on the Jenkins server.
2.  Created a GKE cluster with two nodes.
3.  Created and managed a standalone NGINX Pod.
4.  Deployed the Flipkart application using a Kubernetes Deployment with
    two replicas.
5.  Exposed the application through a LoadBalancer and accessed it using
    an External IP in Chrome.
6.  Demonstrated Kubernetes self-healing and scaling.
7.  Configured a GCP service account and stored its key securely in
    Jenkins Credentials.
8.  Updated the existing Jenkins pipeline to deploy the Docker Hub image
    to GKE.
9.  Verified the Deployment, Pods, Service, and deployed image.
10. Learned how to delete application resources and destroy the GKE
    cluster.

**Final CI/CD workflow:**

GitHub Checkout → Maven Build → Copy WAR to Tomcat → Docker Image Build
→ Docker Hub Push → Deploy to GKE → Verify Deployment → Access
Application through LoadBalancer.

This completes the hands-on GKE and Jenkins CI/CD integration lab.
