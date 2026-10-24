# SonarQube → Jenkins CI → GKE Deployment → Domain Hosting Runbook

**Project:** `august2027`  
**GKE cluster:** `rushi-gke`  
**Cluster location:** `us-central1-a` (zonal)  
**Kubernetes namespace:** `default`  
**Deployment:** `flipkart-app`  
**Service:** `flipkart-service`  
**Application image:** `docker.io/devopsbyrushi/octsecurebankcicdpipeline:v1`  
**Application container port:** `8080`  
**Public HTTP port:** `80`  
**Domain:** `zetaev.com`  
**Current GKE external IP:** `35.253.60.152`  
**Current access URL:** `http://zetaev.com`

> This runbook documents the workflow demonstrated in the project. The GKE cluster is Google Kubernetes Engine (GKE), not Amazon EKS. Commands below assume the same project and resource names shown in the terminal screenshots. Verify names before running destructive commands.

---

## 1. Architecture and workflow

1. Developer pushes application changes to GitHub.
2. Jenkins checks out the repository and builds the application.
3. Maven packages the application and SonarQube analyses the code.
4. Jenkins builds and pushes a Docker image to Docker Hub.
5. GKE pulls the image and runs the application in Kubernetes Pods.
6. A Kubernetes `LoadBalancer` Service exposes the Pods through an external IP.
7. DNS records point `zetaev.com` and `www.zetaev.com` to the GKE external IP.
8. The application is currently accessed over **HTTP**. HTTPS has not yet been configured.

---

## 2. Values to confirm before running commands

| Setting | Value used in this project |
|---|---|
| Google Cloud project | `august2027` |
| GKE cluster | `rushi-gke` |
| GKE zone | `us-central1-a` |
| Jenkins public IP | `34.31.133.187` (shown in earlier project context; verify if changed) |
| SonarQube server public IP | `104.197.125.252` (shown in earlier project context; verify if changed) |
| GitHub repository | `https://github.com/devopsbyrushi/Banking-3Tier-Warfile.git` was used in earlier project work; verify the Jenkins job's actual repository |
| Docker Hub image | `docker.io/devopsbyrushi/octsecurebankcicdpipeline:v1` |
| GKE service external IP | `35.253.60.152` |
| Domain | `zetaev.com` |

Do not copy a public IP from this document blindly if a VM or LoadBalancer has been recreated. Check the live value first.

---

## 3. SonarQube on a new Ubuntu server

### 3.1 Prerequisites

- A fresh Ubuntu server with a reachable public IP.
- SSH access and `sudo` privileges.
- Allow TCP `9000` to access the SonarQube web interface during setup.
- Use a supported SonarQube release and a compatible PostgreSQL version. Avoid exposing PostgreSQL (`5432`) publicly; keep it on localhost or a private network.

The project used a Docker-based SonarQube deployment with PostgreSQL. Exact versions, database credentials, and Compose file contents were not captured in the available screenshots. The following is a safe reference pattern, not a claim that these exact values were used previously.

### 3.2 Install Docker and the Compose plugin

Run on the **SonarQube server**:

```bash
sudo apt-get update
sudo apt-get install -y ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg \
  -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" \
  | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt-get update
sudo apt-get install -y docker-ce docker-ce-cli containerd.io \
  docker-buildx-plugin docker-compose-plugin

sudo systemctl enable --now docker
sudo docker version
sudo docker compose version
```

### 3.3 Prepare persistent directories and PostgreSQL credentials

Use strong, unique passwords and do not commit credentials to Git:

```bash
sudo mkdir -p /opt/sonarqube
sudo chown "$USER":"$USER" /opt/sonarqube
cd /opt/sonarqube
```

Create `.env`:

```bash
cat > .env <<'EOF'
POSTGRES_USER=sonar
POSTGRES_PASSWORD=REPLACE_WITH_A_LONG_RANDOM_PASSWORD
POSTGRES_DB=sonarqube
EOF

chmod 600 .env
```

Replace the example password before starting the services.

### 3.4 Create `compose.yaml`

```yaml
services:
  db:
    image: postgres:15
    container_name: sonarqube-db
    restart: unless-stopped
    env_file:
      - .env
    environment:
      POSTGRES_USER: ${POSTGRES_USER}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
      POSTGRES_DB: ${POSTGRES_DB}
    volumes:
      - postgres_data:/var/lib/postgresql/data
    networks:
      - sonar-net
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER} -d ${POSTGRES_DB}"]
      interval: 10s
      timeout: 5s
      retries: 10

  sonarqube:
    image: sonarqube:lts-community
    container_name: sonarqube
    restart: unless-stopped
    depends_on:
      db:
        condition: service_healthy
    environment:
      SONAR_JDBC_URL: jdbc:postgresql://db:5432/${POSTGRES_DB}
      SONAR_JDBC_USERNAME: ${POSTGRES_USER}
      SONAR_JDBC_PASSWORD: ${POSTGRES_PASSWORD}
    ports:
      - "9000:9000"
    volumes:
      - sonarqube_data:/opt/sonarqube/data
      - sonarqube_extensions:/opt/sonarqube/extensions
      - sonarqube_logs:/opt/sonarqube/logs
    networks:
      - sonar-net

networks:
  sonar-net:

volumes:
  postgres_data:
  sonarqube_data:
  sonarqube_extensions:
  sonarqube_logs:
```

Start and verify:

```bash
cd /opt/sonarqube
sudo docker compose up -d
sudo docker compose ps
sudo docker compose logs -f sonarqube
```

Wait for the logs to indicate that SonarQube has started. Then open:

```text
http://SONARQUBE_SERVER_PUBLIC_IP:9000
```

On a new installation, sign in with the initial credentials shown by the SonarQube version/documentation and immediately change the password. Do not reuse the initial password.

**Linux prerequisites:** SonarQube may require kernel and file-descriptor limits depending on the selected version and host. Review the requirements for the exact version and confirm adequate memory before production use.

### 3.5 Troubleshooting SonarQube

```bash
sudo docker compose ps
sudo docker compose logs --tail=200 sonarqube
sudo docker compose logs --tail=200 db
free -h
df -h
```

If the web page does not open, check the VM firewall and Google Cloud VPC firewall rules for TCP `9000`. Restrict access to trusted IP addresses instead of opening it to everyone. Do not open PostgreSQL port `5432` to the internet.

---

## 4. Connect SonarQube to Jenkins

### 4.1 Create a SonarQube token

1. Sign in to SonarQube.
2. Open **My Account → Security** (the exact menu may vary by version).
3. Generate a user token for Jenkins integration.
4. Copy it once and store it securely.

### 4.2 Add the token to Jenkins Credentials

In Jenkins:

1. Open **Manage Jenkins → Credentials**.
2. Select the appropriate global or folder credential store.
3. Choose **Add Credentials**.
4. Kind: **Secret text**.
5. Secret: paste the SonarQube token.
6. ID: `sonarqube-token`.
7. Description: `SonarQube token for pipeline`.
8. Save.

Use the exact credential ID in the Jenkinsfile. Never paste the token directly into the Jenkinsfile or Git repository.

### 4.3 Configure the SonarQube server in Jenkins

1. Install/verify the **SonarQube Scanner for Jenkins** plugin.
2. Go to **Manage Jenkins → System**.
3. Find **SonarQube servers** and add a server.
4. Name: `SonarQube`.
5. Server URL: `http://SONARQUBE_SERVER_PUBLIC_IP:9000` (use the actual reachable IP or private DNS).
6. Authentication token: select the Jenkins credential `sonarqube-token`.
7. Save.

If the Jenkins and SonarQube VMs are on the same VPC, prefer a private IP for server-to-server traffic and allow TCP `9000` only from Jenkins. Confirm connectivity from the Jenkins VM:

```bash
curl -I --connect-timeout 5 http://SONARQUBE_SERVER_PRIVATE_OR_PUBLIC_IP:9000
```

A successful HTTP response confirms basic connectivity, not token validity.

### 4.4 Install/verify build tools

The Jenkins agent that executes the pipeline needs a compatible Java runtime, Maven (if not using the Maven Wrapper), Git, and Docker access. Check:

```bash
java -version
mvn -version
git --version
docker --version
```

If a command is missing, install/configure it on the **actual Jenkins agent** running the job, not automatically on the SonarQube server.

---

## 5. Configure the pipeline and test a build

### 5.1 Jenkins pipeline configuration

1. Open the relevant Jenkins pipeline job.
2. Confirm the correct GitHub repository and branch.
3. Confirm the job uses the expected `Jenkinsfile` or configured pipeline script.
4. Ensure the pipeline uses the configured SonarQube server name `SonarQube` and credential ID `sonarqube-token`.
5. Ensure Docker Hub credentials are stored in Jenkins Credentials, not hard-coded in the script.

The exact previously working Jenkinsfile was not included in the screenshots available for this runbook. Do not replace a working pipeline with a generic script without first backing up the current script and checking the repository's build structure.

### 5.2 Maven and SonarQube analysis

For a Maven project with a valid `pom.xml`, a typical analysis command is:

```bash
mvn clean verify sonar:sonar \
  -Dsonar.host.url=http://SONARQUBE_SERVER_PUBLIC_IP:9000 \
  -Dsonar.token="$SONAR_TOKEN"
```

Use the Jenkins SonarQube integration (`withSonarQubeEnv('SonarQube')`) when configured. Define the token through Jenkins Credentials; do not print it to logs. Some project structures require a project key or additional scanner configuration. Use the actual project's configuration.

### 5.3 Run the test build

1. Save the Jenkins configuration.
2. Select **Build Now**.
3. Open **Console Output**.
4. Confirm checkout, Maven tests/package, SonarQube analysis, Docker build, and Docker push stages finish successfully.
5. In SonarQube, open the project and verify the new analysis appears.

If the build fails, use the first meaningful error in Console Output rather than changing multiple settings at once.

---

## 6. Create a SonarQube webhook for Jenkins

A webhook is needed when the pipeline uses Jenkins' `waitForQualityGate` step.

1. In SonarQube, open **Administration → Configuration → Webhooks** (menu wording varies by version).
2. Add a webhook.
3. Name: `Jenkins`.
4. URL:

```text
http://JENKINS_SERVER_REACHABLE_ADDRESS:8080/sonarqube-webhook/
```

5. Save and use SonarQube's delivery/test details, if available, to confirm delivery.

Use the Jenkins address reachable **from the SonarQube server**. The trailing slash in `/sonarqube-webhook/` matters. Configure firewalls so SonarQube can reach Jenkins on TCP `8080` (or the actual Jenkins HTTP port). Restrict source IPs where feasible.

### 6.1 Optional Quality Gate stage

If the Jenkins pipeline has configured SonarQube integration and a webhook, a Maven pipeline can use:

```groovy
stage('SonarQube Analysis') {
    steps {
        withSonarQubeEnv('SonarQube') {
            sh 'mvn clean verify sonar:sonar'
        }
    }
}

stage('Quality Gate') {
    steps {
        timeout(time: 10, unit: 'MINUTES') {
            waitForQualityGate abortPipeline: true
        }
    }
}
```

This is a pattern, not a complete replacement Jenkinsfile. It assumes the SonarQube server is named `SonarQube` in Jenkins and the project uses Maven. Place the stages in the correct order for your current pipeline.

---

## 7. Verify the Docker image pushed by Jenkins

The image used during the GKE deployment shown in Cloud Shell was:

```text
docker.io/devopsbyrushi/octsecurebankcicdpipeline:v1
```

In Jenkins, verify the Docker push stage completed. In Docker Hub, verify that the repository and tag exist.

**Important:** A reusable tag such as `v1` can be overwritten. For reliable release tracking, prefer a unique tag such as the Jenkins build number or Git commit SHA. The Kubernetes deployment must reference the exact tag pushed by the successful build.

---

## 8. Create/access GKE using Google Cloud Shell

The cluster was created with `gcloud`, not Terraform, in the session captured by the screenshots. Google Kubernetes Engine is **GKE**, not Amazon EKS.

### 8.1 Set the project and inspect the cluster

Run in **Google Cloud Shell**:

```bash
gcloud config set project august2027
gcloud container clusters list
```

The captured cluster details were:

- Name: `rushi-gke`
- Location: `us-central1-a`
- Status: `RUNNING`
- Two nodes were shown as `Ready`.

### 8.2 Get cluster credentials

```bash
gcloud container clusters get-credentials rushi-gke \
  --zone us-central1-a \
  --project august2027
```

Verify access:

```bash
kubectl get nodes
kubectl get namespaces
```

Do not create a second cluster if `rushi-gke` already exists and is healthy.

---

## 9. Deploy the Docker image to Kubernetes Pods

### 9.1 Check the existing deployment

```bash
kubectl get deployments
kubectl get pods -o wide
kubectl get deployment flipkart-app -o yaml
```

The existing deployment was named `flipkart-app`, with two replicas. Its container image was shown as `docker.io/devopsbyrushi/octsecurebankcicdpipeline:v1`.

### 9.2 Set the image and pull policy

For the image shown in this project:

```bash
kubectl set image deployment/flipkart-app \
  octsecurebankcicdpipeline=docker.io/devopsbyrushi/octsecurebankcicdpipeline:v1
```

The container name was verified in the terminal as `octsecurebankcicdpipeline`.

To ensure Kubernetes checks Docker Hub for the tag whenever it starts a container:

```bash
kubectl patch deployment flipkart-app --type='json' \
  -p='[{"op":"replace","path":"/spec/template/spec/containers/0/imagePullPolicy","value":"Always"}]'
```

If this patch reports that the field is missing, use a merge patch:

```bash
kubectl patch deployment flipkart-app --type='merge' \
  -p '{"spec":{"template":{"spec":{"containers":[{"name":"octsecurebankcicdpipeline","imagePullPolicy":"Always"}]}}}}'
```

Wait for rollout:

```bash
kubectl rollout status deployment/flipkart-app
kubectl get pods -o wide
kubectl describe deployment flipkart-app
```

**Best practice:** For each new Jenkins build, publish a unique image tag and update the deployment to that tag. `imagePullPolicy: Always` helps with mutable tags, but unique tags make rollbacks and audits safer.

### 9.3 Check application logs

```bash
kubectl logs deployment/flipkart-app --all-containers=true --tail=100
```

The captured application logs showed Apache Tomcat starting on port `8080` and deploying `ROOT.war`. Confirm that remains true for the current image.

---

## 10. Expose the application using a LoadBalancer Service

The following service already existed in the captured session:

- Service: `flipkart-service`
- Type: `LoadBalancer`
- Cluster IP: `34.118.225.207`
- External IP: `35.253.60.152`
- Service port: `80`
- Target port: `8080`

Inspect it:

```bash
kubectl get svc flipkart-service
kubectl describe svc flipkart-service
kubectl get endpoints flipkart-service
kubectl get pods -o wide
```

If the service is missing, it can be created using:

```bash
kubectl expose deployment flipkart-app \
  --name=flipkart-service \
  --type=LoadBalancer \
  --port=80 \
  --target-port=8080
```

Do not run that command if the service already exists; inspect and reuse the existing service instead.

Wait for an external IP:

```bash
kubectl get svc flipkart-service --watch
```

Stop watching with `Ctrl+C` once the IP is assigned.

### 10.1 Test the external IP

```bash
curl -v --connect-timeout 10 http://35.253.60.152/
```

The captured terminal showed an HTTP `200` response from the external IP, with HTML content for the Flipkart demo. That is evidence the application was responding at that time.

---

## 11. Connect the domain using Hostinger DNS

Although GoDaddy was initially mentioned, the actual DNS-management screenshots show the domain `zetaev.com` managed in **Hostinger hPanel**. Make DNS changes at the provider whose nameservers are authoritative for the domain. Do not change nameservers merely to follow this runbook.

### 11.1 DNS records

In Hostinger, open **Domains → zetaev.com → DNS / Nameservers**.

Set or verify:

| Type | Name | Target / Value | TTL |
|---|---|---|---|
| A | `@` | `35.253.60.152` | `300` or provider default |
| CNAME | `www` | `zetaev.com` | `300` or provider default |

The screenshot showed an existing `ALIAS @` pointing to `zetaev.com.cdn.hstgr.net` and a `CNAME www` pointing to `www.zetaev.com.cdn.hstgr.net`. The working change was to point the apex and `www` to the GKE IP. Remove or replace conflicting web-hosting records for `@` and `www`, but **preserve mail-related MX, SPF, DKIM, DMARC, and autodiscover records**.

If the DNS interface does not allow an `A` record alongside the existing apex `ALIAS`, remove/replace the conflicting apex web record with the A record as supported by the provider.

### 11.2 Verify DNS resolution

Run in Cloud Shell:

```bash
dig +short zetaev.com
dig +short www.zetaev.com
```

The screenshots eventually showed both names resolving to:

```text
35.253.60.152
```

DNS propagation/caching may take time. Check the authoritative nameservers if results differ.

### 11.3 Open the site in Chrome

Use:

```text
http://zetaev.com
```

or:

```text
http://www.zetaev.com
```

**Important:** The project was successfully opened after using `http://zetaev.com`. Do not use `https://zetaev.com` yet: HTTPS has not been configured in this workflow. A browser may upgrade to HTTPS automatically; type the full `http://` URL explicitly while testing.

The existing website content hosted through Hostinger may be replaced when the domain points to GKE. Back up any existing website files before deleting them. DNS changes redirect traffic; they do not themselves delete old files.

---

## 12. Confirm the complete deployment

Run:

```bash
kubectl get deployment flipkart-app
kubectl get pods -o wide
kubectl get svc flipkart-service
kubectl get endpoints flipkart-service
curl -I --connect-timeout 10 http://35.253.60.152/
curl -I --connect-timeout 10 http://zetaev.com/
```

Expected checks:

- Deployment desired and available replicas match.
- Pods are `Running` and `Ready`.
- The service has an external IP.
- Endpoints list the application Pods on port `8080`.
- HTTP requests reach the application.
- The domain resolves to the same external IP.

If a check fails, inspect:

```bash
kubectl describe deployment flipkart-app
kubectl describe svc flipkart-service
kubectl get events --sort-by=.lastTimestamp
kubectl logs deployment/flipkart-app --all-containers=true --tail=100
```

---

## 13. Stop the cluster to reduce cost

GKE billing can include cluster-management charges, node VM costs, load balancer costs, and persistent storage. Merely closing Cloud Shell or the browser does not stop resources.

### 13.1 Scale application Pods down

This leaves the cluster running but reduces application replicas:

```bash
kubectl scale deployment flipkart-app --replicas=0
```

Restore them later with:

```bash
kubectl scale deployment flipkart-app --replicas=2
```

The LoadBalancer may still incur charges while the service exists.

### 13.2 Resize the node pool to zero (if supported for your configuration)

First inspect node pools:

```bash
gcloud container node-pools list \
  --cluster rushi-gke \
  --zone us-central1-a \
  --project august2027
```

For a standard GKE cluster, identify the actual node-pool name, then resize it to zero if the pool/configuration permits:

```bash
gcloud container clusters resize rushi-gke \
  --node-pool=YOUR_NODE_POOL_NAME \
  --num-nodes=0 \
  --zone=us-central1-a \
  --project=august2027
```

This can make workloads unavailable and may not remove all cluster or external-resource costs. Do not run until you have confirmed the node-pool name and intend to take the application offline.

### 13.3 Deleting the LoadBalancer Service

If you are intentionally taking the app offline and no longer need its public endpoint:

```bash
kubectl delete service flipkart-service
```

This normally asks GKE to release the Service's cloud load balancer and its ephemeral external IP. Wait for cleanup and verify in Google Cloud Console. If you later recreate the service, the IP can change and DNS must be updated.

---

## 14. Delete the GKE cluster completely

**Destructive action:** This deletes the GKE cluster and its workloads. Back up anything you need first. Confirm the project and cluster name before running the command.

### 14.1 Confirm target

```bash
gcloud config get-value project
gcloud container clusters list --project august2027
```

Expected target:

```text
Project: august2027
Cluster: rushi-gke
Zone: us-central1-a
```

### 14.2 Delete the cluster

```bash
gcloud container clusters delete rushi-gke \
  --zone=us-central1-a \
  --project=august2027
```

Review the confirmation prompt carefully before accepting.

### 14.3 Verify deletion

```bash
gcloud container clusters list --project august2027
```

Also inspect **Google Cloud Console → Network services → Load balancing** and **VPC network → IP addresses** for any leftover resources. Do not delete resources that other workloads use. If the service used an ephemeral IP, it will generally be released during cleanup; if a static IP was reserved separately, it may need separate release.

**Domain reminder:** After the cluster or LoadBalancer is deleted, `zetaev.com` will continue resolving to the old IP until DNS is changed. Remove or update the A record when the application is intentionally retired.

---

## 15. Important operational and security notes

- **GKE is not EKS.** The project uses Google Kubernetes Engine and `gcloud`, not Amazon EKS.
- Keep Jenkins, SonarQube, Docker Hub, and database secrets in credential stores, not Git.
- Do not expose PostgreSQL to the internet.
- Restrict Jenkins and SonarQube administrative ports to trusted IPs or private networking.
- The application endpoint currently uses HTTP. Do not send real credentials or sensitive data over it. Configure HTTPS before production use.
- Prefer immutable image tags (build number or Git SHA) instead of overwriting `v1`.
- Back up Jenkins configuration, SonarQube data/database, and any required application data before deleting servers or clusters.
- Stopping or deleting GKE does not automatically delete the separate Jenkins and SonarQube VMs.

---

## 16. Quick reference commands

```bash
# Select project
gcloud config set project august2027

# Cluster access
gcloud container clusters get-credentials rushi-gke \
  --zone us-central1-a --project august2027

# Workload status
kubectl get nodes
kubectl get deployments
kubectl get pods -o wide
kubectl get svc flipkart-service
kubectl get endpoints flipkart-service

# Application logs
kubectl logs deployment/flipkart-app --all-containers=true --tail=100

# Update the running image (replace with the actual tag pushed by Jenkins)
kubectl set image deployment/flipkart-app \
  octsecurebankcicdpipeline=docker.io/devopsbyrushi/octsecurebankcicdpipeline:YOUR_NEW_TAG

# Check domain DNS
dig +short zetaev.com
dig +short www.zetaev.com

# Test HTTP
curl -I --connect-timeout 10 http://35.253.60.152/
curl -I --connect-timeout 10 http://zetaev.com/

# Delete the cluster (destructive; verify target first)
gcloud container clusters delete rushi-gke \
  --zone=us-central1-a --project=august2027
```

**Current successful milestone:** SonarQube/Jenkins integration and pipeline work were discussed; GKE Pods were running; the LoadBalancer IP was `35.253.60.152`; DNS was changed to that IP; and the application was accessible using `http://zetaev.com`. Verify all resources and credentials against the live environment before repeating any setup or cleanup.
