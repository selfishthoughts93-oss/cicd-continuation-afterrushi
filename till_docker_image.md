# SonarQube + Jenkins + GKE CI/CD Runbook

**GCP project:** `august2027`\
**GKE cluster:** `rushi-gke`\
**Zone:** `us-central1-a`\
**Jenkins job:** `cicd-pipeline2`\
**Last verified:** 24 October 2026

> **Terminology:** This is **GKE (Google Kubernetes Engine)** on Google
> Cloud, not Amazon EKS. The commands below are for GCP.

## 1. Environment and successful result

  ----------------------------------------------------------------------------------------------------------------
  Component                           Details observed
  ----------------------------------- ----------------------------------------------------------------------------
  Jenkins VM                          `34.31.133.187:8080`

  SonarQube VM                        `104.197.125.252:9000`

  Git repository                      `https://github.com/devopsbyrushi/rushiDevOpsTraining-FlipkartWarDemo.git`

  Git credential ID                   `rushi-github-credentials`

  Docker Hub image repository         `devopsbyrushi/octsecurebankcicdpipleline`

  Docker Hub credential ID            `dockerhub-passwd` (Secret text)

  SonarQube server name configured in `SonarQube`
  Jenkins                             

  GKE node pool                       `default-pool`

  Kubernetes Deployment               `flipkart-app`

  Kubernetes Service                  `flipkart-service`

  Initial image deployed              `docker.io/devopsbyrushi/octsecurebankcicdpipleline:v1`

  Application URL that worked         `http://35.253.60.152/`
  ----------------------------------------------------------------------------------------------------------------

Verified during this project: - SonarQube analysis succeeded and the
Quality Gate was `OK`. - Docker image was built and pushed to Docker
Hub. - GKE cluster `rushi-gke` had two `Ready` nodes. - Deployment
`flipkart-app` had two ready Pods. - The LoadBalancer IP `35.253.60.152`
returned `HTTP/1.1 200`, and the Flipkart Clone page loaded in Chrome.

Public IPs can change when resources are recreated. Always verify
current addresses before using them.

------------------------------------------------------------------------

## 2. Install Docker and SonarQube with PostgreSQL on a new Ubuntu server

These are **new-server instructions**. The existing SonarQube server is
already running at `http://104.197.125.252:9000`; do not reinstall it if
it is healthy.

### 2.1 Install Docker

Run on the **SonarQube VM**:

``` bash
sudo apt-get update
sudo apt-get install -y ca-certificates curl
```

Install Docker Engine and the Docker Compose plugin using Docker's
official instructions for your Ubuntu release. Then verify:

``` bash
docker --version
docker compose version
sudo systemctl enable --now docker
sudo systemctl status docker --no-pager
```

### 2.2 Configure the kernel setting required by Elasticsearch

``` bash
echo 'vm.max_map_count=524288' | sudo tee /etc/sysctl.d/99-sonarqube.conf
sudo sysctl --system
sysctl vm.max_map_count
```

Confirm the value is at least `524288`, and check the requirements for
the specific SonarQube version.

### 2.3 Create a Compose file

``` bash
sudo mkdir -p /opt/sonarqube
cd /opt/sonarqube
sudo nano compose.yaml
```

Example `compose.yaml`:

``` yaml
services:
  db:
    image: postgres:16
    container_name: sonarqube-db
    restart: unless-stopped
    environment:
      POSTGRES_USER: sonar
      POSTGRES_PASSWORD: REPLACE_WITH_STRONG_PASSWORD
      POSTGRES_DB: sonar
    volumes:
      - postgresql_data:/var/lib/postgresql/data
    networks:
      - sonarnet
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U sonar -d sonar"]
      interval: 10s
      timeout: 5s
      retries: 10

  sonarqube:
    image: sonarqube:community
    container_name: sonarqube
    restart: unless-stopped
    depends_on:
      db:
        condition: service_healthy
    ports:
      - "9000:9000"
    environment:
      SONAR_JDBC_URL: jdbc:postgresql://db:5432/sonar
      SONAR_JDBC_USERNAME: sonar
      SONAR_JDBC_PASSWORD: REPLACE_WITH_STRONG_PASSWORD
    volumes:
      - sonarqube_data:/opt/sonarqube/data
      - sonarqube_extensions:/opt/sonarqube/extensions
      - sonarqube_logs:/opt/sonarqube/logs
    networks:
      - sonarnet

networks:
  sonarnet:

volumes:
  postgresql_data:
  sonarqube_data:
  sonarqube_extensions:
  sonarqube_logs:
```

Replace both password placeholders with the **same strong password**.
Use image versions supported by the selected SonarQube release; pin and
test versions before a long-lived deployment.

Start and inspect the containers:

``` bash
cd /opt/sonarqube
sudo docker compose up -d
sudo docker compose ps
sudo docker compose logs --tail=100 sonarqube
sudo docker compose logs --tail=100 db
```

Open `http://SONARQUBE_PUBLIC_IP:9000`, sign in, and change the initial
administrator password. Keep PostgreSQL private; do not expose port
`5432` publicly.

### 2.4 Network and firewall

-   Allow TCP `9000` only from trusted source IPs, including Jenkins
    where required.
-   Ensure the VM has sufficient memory, CPU, and disk for the selected
    SonarQube version.
-   For production use, prefer HTTPS and restricted network access
    rather than exposing the UI openly over HTTP.

------------------------------------------------------------------------

## 3. Connect SonarQube to Jenkins

### 3.1 Install the Jenkins plugin

In Jenkins at `http://34.31.133.187:8080`:

1.  Open **Manage Jenkins → Plugins**.
2.  Search for **SonarQube Scanner**.
3.  Install it and restart Jenkins if requested.

### 3.2 Create a SonarQube token

In SonarQube at `http://104.197.125.252:9000`:

1.  Sign in with an account authorized to run analysis.
2.  Open the account/security token area (menu names vary by version).
3.  Generate a token such as `jenkins-cicd`.
4.  Copy it and store it securely. Do not put it in a Jenkinsfile or Git
    repository.

### 3.3 Add the token to Jenkins Credentials

In Jenkins:

1.  Open **Manage Jenkins → Credentials** and select the appropriate
    global domain.
2.  Click **Add Credentials**.
3.  Kind: **Secret text**.
4.  Secret: paste the SonarQube token.
5.  ID: `sonarqube-token` (recommended example; use the same ID in
    server configuration).
6.  Save.

### 3.4 Configure the SonarQube server

1.  Open **Manage Jenkins → System**.
2.  Find **SonarQube servers** and add a server.
3.  Set:
    -   **Name:** `SonarQube`
    -   **Server URL:** `http://104.197.125.252:9000`
    -   **Server authentication token:** select `sonarqube-token`
4.  Enable environment injection if the option is shown, then save.

The server name must match `withSonarQubeEnv('SonarQube')` in the
pipeline.

### 3.5 Scanner tool note

If the pipeline runs `mvn sonar:sonar`, a separately installed
SonarScanner CLI is normally unnecessary. If using `sonar-scanner`
directly, configure it under **Manage Jenkins → Tools** and reference
the configured installation name.

------------------------------------------------------------------------

## 4. Create the SonarQube webhook

The webhook is required for Jenkins' `waitForQualityGate()` step to
receive the Quality Gate result.

In SonarQube:

1.  Open **Administration → Configuration → Webhooks** (wording may
    vary).
2.  Add a webhook.
3.  Name it `Jenkins`.
4.  URL:

``` text
http://34.31.133.187:8080/sonarqube-webhook/
```

5.  Keep the final slash and save.

The SonarQube VM must be able to reach Jenkins on TCP `8080`, and the
Jenkins network/firewall must permit that connection. For production,
prefer HTTPS and restrict traffic to trusted hosts. If the webhook
fails, inspect its delivery status in SonarQube and Jenkins logs.

------------------------------------------------------------------------

## 5. Jenkins prerequisites and fixes from this project

### 5.1 Verify tools on the Jenkins VM

``` bash
java -version
mvn -version
git --version
docker --version
sudo systemctl status jenkins --no-pager
sudo systemctl status docker --no-pager
```

Use a Java version compatible with your Jenkins and scanner versions.
The Java used to run Maven is separate from the Java/Tomcat runtime
inside the application image.

### 5.2 Docker socket permission error

The pipeline previously failed with
`permission denied while trying to connect to the Docker API`. The
Jenkins service account was added to the Docker group:

``` bash
sudo usermod -aG docker jenkins
sudo systemctl restart jenkins
sudo -u jenkins -H docker ps
```

Restarting Jenkins applies the group membership (and interrupts active
builds). Docker group access is effectively root-level access, so grant
it only to trusted accounts.

### 5.3 Maven workspace ownership error

The pipeline previously failed with `Permission denied` while writing
`target/maven-archiver/pom.properties`. Running Maven with `sudo` can
leave root-owned files in the Jenkins workspace. Avoid `sudo mvn`.

For this job, the workspace repair used was:

``` bash
sudo chown -R jenkins:jenkins /var/lib/jenkins/workspace/cicd-pipeline2
sudo rm -rf /var/lib/jenkins/workspace/cicd-pipeline2/target
```

Run Maven as the Jenkins user. Do not use `chmod -R 777` as a
workaround.

### 5.4 Credentials reference

  -----------------------------------------------------------------------
  Credential ID                       Type / purpose
  ----------------------------------- -----------------------------------
  `rushi-github-credentials`          GitHub credentials for checkout

  `dockerhub-passwd`                  Secret text containing Docker Hub
                                      password/token

  `sonarqube-token`                   Secret text SonarQube token
                                      (recommended ID; confirm in
                                      Jenkins)
  -----------------------------------------------------------------------

Never print or commit secret values.

------------------------------------------------------------------------

## 6. Jenkins pipeline to build, scan, gate, and push Docker images

Job: `cicd-pipeline2`. Back up the current working script before
replacing it. This script keeps the repository and Docker Hub repository
from the existing setup and adds a unique tag per build.

Paste into **Configure → Pipeline → Script** after verifying the
credential IDs, branch, and SonarQube server name:

``` groovy
node {
    def imageRepo = 'devopsbyrushi/octsecurebankcicdpipleline'
    def imageTag = "build-${env.BUILD_NUMBER}"

    stage('GitHub code checkout') {
        git credentialsId: 'rushi-github-credentials',
            url: 'https://github.com/devopsbyrushi/rushiDevOpsTraining-FlipkartWarDemo.git',
            branch: 'master'
    }

    stage('Maven validate') {
        sh 'mvn -B validate'
    }

    stage('Maven build and package') {
        sh 'mvn -B clean install'
        sh 'test -f target/flipkart-clone.war'
    }

    stage('SonarQube Analysis') {
        withSonarQubeEnv('SonarQube') {
            sh '''
                mvn -B sonar:sonar \
                  -Dsonar.projectKey=flipkart-war-demo \
                  -Dsonar.projectName="Flipkart War Demo"
            '''
        }
    }

    stage('SonarQube Quality Gate') {
        timeout(time: 5, unit: 'MINUTES') {
            def qualityGate = waitForQualityGate()
            if (qualityGate.status != 'OK') {
                error "SonarQube Quality Gate failed: ${qualityGate.status}"
            }
        }
    }

    stage('Build Docker image') {
        sh "docker build -t ${imageRepo}:${imageTag} -t ${imageRepo}:v1 ."
    }

    stage('Docker Hub login') {
        withCredentials([string(credentialsId: 'dockerhub-passwd',
                                variable: 'DOCKERHUB_PASSWORD')]) {
            sh '''
                set +x
                echo "$DOCKERHUB_PASSWORD" | docker login \
                  -u devopsbyrushi --password-stdin
            '''
        }
    }

    stage('Push Docker image') {
        sh "docker push ${imageRepo}:${imageTag}"
        sh "docker push ${imageRepo}:v1"
    }

    stage('Record image tag') {
        writeFile file: 'image-tag.txt',
                  text: "${imageRepo}:${imageTag}\\n"
        archiveArtifacts artifacts: 'image-tag.txt', fingerprint: true
        echo "Pushed image: ${imageRepo}:${imageTag}"
    }
}
```

Important notes:

-   This builds tags such as `build-14` and also updates the convenience
    tag `v1`. Deploy the unique build tag for traceability and rollback.
-   The Dockerfile must copy `target/flipkart-clone.war` to Tomcat's
    `ROOT.war`, as the working build did.
-   The old VM Tomcat copy stage is intentionally omitted; Kubernetes
    does not require copying the WAR to `/opt/tomcat10/webapps/`.
-   `waitForQualityGate()` needs the SonarQube Jenkins plugin and the
    webhook configured above.
-   If Maven says `No plugin found for prefix 'sonar'`, inspect Maven
    connectivity and scanner configuration. Do not ignore the failed
    analysis.
-   The observed repository branch was `master`; change the pipeline if
    the branch changes.
-   This script **does not yet deploy to GKE automatically**. Section 8
    describes the manual image update.

### 6.1 Test with Build Now

1.  Save the pipeline.
2.  Click **Build Now**.
3.  Open **Console Output**.
4.  Verify checkout, Maven, SonarQube analysis, Quality Gate (`OK`),
    Docker build, login, and image push all succeed.
5.  Open SonarQube and verify the project analysis.

Fix any failing stage before moving on.

------------------------------------------------------------------------

## 7. Create or connect to GKE from Google Cloud Shell

### 7.1 Select the GCP project and enable APIs

Run in **Google Cloud Shell**:

``` bash
gcloud config set project august2027
gcloud services enable container.googleapis.com
gcloud services enable compute.googleapis.com
```

### 7.2 Create the cluster only if it does not already exist

The existing cluster is `rushi-gke` in `us-central1-a` with two nodes
observed. Check first:

``` bash
gcloud container clusters list --project=august2027
```

Only if the cluster does not exist, a new standard cluster can be
created with:

``` bash
gcloud container clusters create rushi-gke \
  --project=august2027 \
  --zone=us-central1-a \
  --num-nodes=2 \
  --machine-type=e2-medium
```

Cluster creation and worker nodes are billable. Check quota, pricing,
and sizing before creating resources.

### 7.3 Get cluster credentials

``` bash
gcloud container clusters get-credentials rushi-gke \
  --zone=us-central1-a \
  --project=august2027
kubectl get nodes
kubectl get namespaces
```

The two nodes were observed in `Ready` state.

------------------------------------------------------------------------

## 8. Deploy the Docker image to Pods and expose it in Chrome

### 8.1 Check whether the Deployment already exists

``` bash
kubectl get deployment flipkart-app
```

If it does not exist, create it:

``` bash
kubectl create deployment flipkart-app \
  --image=docker.io/devopsbyrushi/octsecurebankcicdpipleline:v1 \
  --replicas=2
```

Do not create a duplicate if the Deployment already exists.

### 8.2 Expose the app with a LoadBalancer Service

Check first:

``` bash
kubectl get service flipkart-service
```

If it does not exist, create it:

``` bash
kubectl expose deployment flipkart-app \
  --name=flipkart-service \
  --type=LoadBalancer \
  --port=80 \
  --target-port=8080
```

This maps external TCP port `80` to Tomcat container port `8080`.

### 8.3 Wait for the external IP

``` bash
kubectl get service flipkart-service --watch
```

Wait until `EXTERNAL-IP` is an IP rather than `<pending>`, then press
`Ctrl+C`.

The observed IP was `35.253.60.152`. The application loaded successfully
at:

``` text
http://35.253.60.152/
```

### 8.4 Verify rollout and connectivity

``` bash
kubectl rollout status deployment/flipkart-app
kubectl get pods -o wide
kubectl get service flipkart-service
kubectl describe service flipkart-service
kubectl get endpoints flipkart-service
curl -I --connect-timeout 10 http://35.253.60.152/
```

Use the current Service IP if it differs. In this project, `curl`
returned `HTTP/1.1 200` and Chrome displayed the Flipkart Clone page.

### 8.5 Update Kubernetes to the newest image from Jenkins

After a successful Jenkins build, use the unique image tag printed in
the Jenkins build output, for example `build-14`.

First find the actual container name in the Deployment:

``` bash
kubectl get deployment flipkart-app \
  -o jsonpath='{.spec.template.spec.containers[*].name}{"\n"}'
```

Then update the image, replacing `BUILD_NUMBER` with the actual build
number and using the container name returned above:

``` bash
kubectl set image deployment/flipkart-app \
  octsecurebankcicdpipleline=docker.io/devopsbyrushi/octsecurebankcicdpipleline:build-BUILD_NUMBER
```

Monitor:

``` bash
kubectl rollout status deployment/flipkart-app
kubectl get pods -o wide
kubectl describe deployment flipkart-app
```

If the container name differs, substitute the real name. If the Docker
Hub repository is private, configure an image-pull secret; do not embed
credentials in the image URL.

### 8.6 Current workflow vs. future automation

Currently verified: Jenkins → SonarQube analysis and Quality Gate →
Docker Hub push. GKE deployment is performed manually with
`kubectl set image`.

To automate deployment from Jenkins later, configure a dedicated GCP
identity with least-privilege permissions, securely store credentials,
install/configure `gcloud` and `kubectl` on the Jenkins agent, and add a
stage that updates the Deployment and waits for rollout success. Do not
grant broad Owner/Editor access just to make the pipeline work.

------------------------------------------------------------------------

## 9. Stop the app and reduce GKE costs

GKE does not have one universal `stop cluster` command like a VM. You
can scale workloads down, scale a node pool down, or delete the cluster.

### Option A: stop the application Pods

``` bash
kubectl scale deployment/flipkart-app --replicas=0
kubectl get pods
```

The LoadBalancer can continue to incur charges. To remove its resources
too:

``` bash
kubectl delete service flipkart-service
```

The public IP may stop working and may not be reused.

To start the app again:

``` bash
kubectl scale deployment/flipkart-app --replicas=2
kubectl expose deployment flipkart-app \
  --name=flipkart-service \
  --type=LoadBalancer \
  --port=80 \
  --target-port=8080
kubectl get service flipkart-service --watch
```

The recreated Service may receive a different IP.

### Option B: scale the worker node pool to zero

Inspect the cluster and node pools first:

``` bash
gcloud container clusters describe rushi-gke \
  --project=august2027 \
  --zone=us-central1-a
gcloud container node-pools list \
  --cluster=rushi-gke \
  --project=august2027 \
  --zone=us-central1-a
```

For the observed `default-pool`, the resize command is:

``` bash
gcloud container clusters resize rushi-gke \
  --project=august2027 \
  --zone=us-central1-a \
  --node-pool=default-pool \
  --num-nodes=0
```

To restore two nodes:

``` bash
gcloud container clusters resize rushi-gke \
  --project=august2027 \
  --zone=us-central1-a \
  --node-pool=default-pool \
  --num-nodes=2
```

Scaling to zero is not deleting the cluster; control-plane and other
resources may still incur charges. Confirm that your cluster/node-pool
configuration supports this operation.

------------------------------------------------------------------------

## 10. Delete the GKE cluster completely

> **Destructive operation:** Deleting the cluster removes the Kubernetes
> workloads and cluster resources. Make sure you no longer need the
> application or any data stored only in the cluster. This does not
> delete the Jenkins VM, SonarQube VM, or Docker Hub image.

### 10.1 Verify project and cluster

``` bash
gcloud config get-value project
gcloud container clusters list --project=august2027
```

Confirm the intended cluster is `rushi-gke` in `us-central1-a`.

### 10.2 Delete the cluster

``` bash
gcloud container clusters delete rushi-gke \
  --project=august2027 \
  --zone=us-central1-a
```

Read the confirmation prompt and proceed only if you intend to delete
this cluster.

### 10.3 Verify deletion

``` bash
gcloud container clusters list --project=august2027
```

Confirm `rushi-gke` no longer appears. Check the GCP Console for
separately reserved IP addresses or other resources you created. Review
billing after deletion; independently provisioned resources can remain
billable.

------------------------------------------------------------------------

## 11. Troubleshooting

### SonarQube analysis fails

-   Verify `http://104.197.125.252:9000` is reachable from Jenkins.
-   Confirm the configured Jenkins server name is exactly `SonarQube`.
-   Confirm the token credential ID and token validity.
-   Inspect the Maven scanner error.
-   Verify the webhook URL ends with `/sonarqube-webhook/`.

### Maven permission denied

Do not run `sudo mvn`. Ensure the job workspace is owned by
`jenkins:jenkins` and remove stale root-owned `target` files.

### Docker permission denied

Check Docker is running, the `jenkins` user has the required access, and
Jenkins was restarted after group membership changes:

``` bash
sudo systemctl status docker --no-pager
sudo -u jenkins -H docker ps
```

### Pod cannot start or pull the image

``` bash
kubectl get pods
kubectl describe pod POD_NAME
kubectl logs POD_NAME
```

Check the image tag, Docker Hub visibility, pull permissions, and Pod
events.

### External IP pending or browser timeout

``` bash
kubectl get service flipkart-service
kubectl describe service flipkart-service
kubectl get endpoints flipkart-service
kubectl get pods -o wide
```

Check the current IP, Service target port, endpoints, load-balancer
provisioning, and GKE firewall/network configuration.

------------------------------------------------------------------------

## 12. Completion checklist

-   [x] SonarQube running on a separate VM and accessible in Chrome.
-   [x] Jenkins configured to use SonarQube.
-   [x] SonarQube analysis successful and Quality Gate `OK`.
-   [x] Jenkins built and pushed a Docker image to Docker Hub.
-   [x] GKE cluster `rushi-gke` created in project `august2027`.
-   [x] Two GKE worker nodes reached `Ready`.
-   [x] Deployment `flipkart-app` ran two ready Pods.
-   [x] Service `flipkart-service` received a public external IP.
-   [x] Application returned HTTP 200 and loaded in Chrome at
    `http://35.253.60.152/`.
-   [ ] Jenkins automatically deploys each successful image to GKE
    (future enhancement).
-   [ ] HTTPS/domain configured (optional production hardening).

## 13. Security reminders

-   The observed endpoints use HTTP; configure HTTPS for production.
-   Never commit SonarQube tokens, Docker Hub secrets, or GCP
    service-account keys to Git.
-   Use unique image tags for traceability and rollback.
-   Back up the SonarQube database and persistent data before upgrades.
-   Verify the active project and resource name before deleting
    anything.
-   Check Google Cloud billing after creating clusters, node pools, and
    LoadBalancer services.
