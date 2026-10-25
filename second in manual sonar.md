# SonarQube → Jenkins CI → GKE Deployment → Domain Hosting Runbook

**Project:** `august2027`\
**GKE cluster:** `rushi-gke`\
**Cluster location:** `us-central1-a` (zonal)\
**Kubernetes namespace:** `default`\
**Deployment:** `flipkart-app`\
**Service:** `flipkart-service`\
**Application image:**
`docker.io/devopsbyrushi/octsecurebankcicdpipeline:v1`\
**Application container port:** `8080`\
**Public HTTP port:** `80`\
**Domain:** `zetaev.com`\
**Current GKE external IP:** `35.253.60.152`\
**Current access URL:** `http://zetaev.com`

> This runbook documents the workflow demonstrated in the project. The
> GKE cluster is Google Kubernetes Engine (GKE), not Amazon EKS.
> Commands below assume the same project and resource names shown in the
> terminal screenshots. Verify names before running destructive
> commands.

------------------------------------------------------------------------

## 1. Architecture and workflow

1.  Developer pushes application changes to GitHub.
2.  Jenkins checks out the repository and builds the application.
3.  Maven packages the application and SonarQube analyses the code.
4.  Jenkins builds and pushes a Docker image to Docker Hub.
5.  GKE pulls the image and runs the application in Kubernetes Pods.
6.  A Kubernetes `LoadBalancer` Service exposes the Pods through an
    external IP.
7.  DNS records point `zetaev.com` and `www.zetaev.com` to the GKE
    external IP.
8.  The application is currently accessed over **HTTP**. HTTPS has not
    yet been configured.

------------------------------------------------------------------------

## 2. Values to confirm before running commands

  --------------------------------------------------------------------------------------------------
  Setting                             Value used in this project
  ----------------------------------- --------------------------------------------------------------
  Google Cloud project                `august2027`

  GKE cluster                         `rushi-gke`

  GKE zone                            `us-central1-a`

  Jenkins public IP                   `34.31.133.187` (shown in earlier project context; verify if
                                      changed)

  SonarQube server public IP          `34.58.51.3` (shown in the successful browser screenshot;
                                      verify if changed)

  GitHub repository                   `https://github.com/devopsbyrushi/Banking-3Tier-Warfile.git`
                                      was used in earlier project work; verify the Jenkins job's
                                      actual repository

  Docker Hub image                    `docker.io/devopsbyrushi/octsecurebankcicdpipeline:v1`

  GKE service external IP             `35.253.60.152`

  Domain                              `zetaev.com`
  --------------------------------------------------------------------------------------------------

Do not copy a public IP from this document blindly if a VM or
LoadBalancer has been recreated. Check the live value first.

------------------------------------------------------------------------

## 3. Manual SonarQube Community Build and PostgreSQL installation (Ubuntu)

This section replaces the former Docker/Compose procedure. The
successful installation shown in the project used **Ubuntu 24.04**,
**PostgreSQL 16.15**, **OpenJDK 21.0.12**, and **SonarQube Community
Build 26.10.0.133078**. SonarQube was installed directly under
`/opt/sonarqube`, configured to use PostgreSQL on `127.0.0.1:5432`, and
started by `systemd`.

> **Important:** Run these steps on the **SonarQube VM**. Do not run the
> PostgreSQL commands on Jenkins or Cloud Shell. The commands below are
> for a fresh/manual setup. If PostgreSQL and the database already
> exist, inspect them before repeating the database-creation commands.

### 3.1 Confirm the OS, Java, disk, and existing PostgreSQL

``` bash
lsb_release -a
java -version
df -h /
sudo systemctl is-active postgresql
sudo -u postgres psql -l
```

The successful environment showed Ubuntu 24.04, Java 21, PostgreSQL
16.15, an active PostgreSQL service, and a database named `sonarqube`
owned by role `sonar`.

### 3.2 Install prerequisites and PostgreSQL

``` bash
sudo apt update
sudo apt install -y postgresql postgresql-contrib openjdk-21-jdk curl wget unzip nano
sudo systemctl enable --now postgresql
sudo systemctl is-active postgresql
java -version
psql --version
nano --version
```

If `java -version` does not report Java 21, stop and resolve the Java
installation before continuing. The SonarQube build used in this runbook
was started successfully with Java 21.

### 3.3 Create the PostgreSQL role and database

Open the PostgreSQL shell:

``` bash
sudo -u postgres psql
```

At the `postgres=#` prompt, run the following SQL. Replace
`REPLACE_WITH_A_STRONG_PASSWORD` with a strong password and keep it
available securely for the matching SonarQube configuration:

``` sql
CREATE USER sonar WITH ENCRYPTED PASSWORD 'REPLACE_WITH_A_STRONG_PASSWORD';
CREATE DATABASE sonarqube OWNER sonar;
\l
\q
```

If the role or database already exists, do **not** run the `CREATE`
statements again. Inspect the existing objects instead. To change the
existing role password, use:

``` bash
sudo -u postgres psql
```

Then run:

``` sql
ALTER USER sonar WITH ENCRYPTED PASSWORD 'REPLACE_WITH_A_NEW_STRONG_PASSWORD';
\q
```

If you change the password, update `sonar.jdbc.password` in
`/opt/sonarqube/conf/sonar.properties` to the same value before
restarting SonarQube.

Verify that the application role can connect. Enter the same password
configured above when prompted:

``` bash
psql -h 127.0.0.1 -U sonar -d sonarqube \
  -c "SELECT current_database(), current_user;"
```

Expected result: `sonarqube` as the database and `sonar` as the user.
Keep PostgreSQL private; do **not** open port `5432` to the internet.

### 3.4 Download the Linux Community Build directly on Ubuntu

The first download URL used in the session returned an HTML page rather
than a ZIP archive. Do **not** use that modal-form URL. The working
direct binary URL was:

``` bash
cd /tmp
sudo rm -f /tmp/sonarqube.zip
wget -O /tmp/sonarqube.zip \
  "https://binaries.sonarsource.com/Distribution/sonarqube/sonarqube-26.10.0.133078.zip"
```

Verify that the file is a valid archive before extracting:

``` bash
ls -lh /tmp/sonarqube.zip
unzip -t /tmp/sonarqube.zip | tail -5
```

The successful download was approximately 906 MB and `unzip -t` reported
no errors. If the archive test fails, do not extract it; download it
again from the direct binary URL above.

### 3.5 Create the service account, then prepare `/opt/sonarqube`

**Important:** Create the Linux `sonar` account before running `chown` or `sudo -u sonar`. Otherwise, commands fail with `invalid user: 'sonar:sonar'` or `unknown user sonar`.

Create the group first, then create the user. **Do not skip the group-creation command.** The earlier failed command attempted to create the user with `--gid sonar` before the `sonar` group existed, which causes `useradd: group 'sonar' does not exist`.

Run these commands **in this exact order**, one at a time:

```bash
# 1. Create the system group only if it does not already exist.
if ! getent group sonar >/dev/null 2>&1; then
  sudo groupadd --system sonar
fi

# 2. Create the system user only if it does not already exist.
if ! id sonar >/dev/null 2>&1; then
  sudo useradd --system --gid sonar --home-dir /opt/sonarqube --shell /usr/sbin/nologin sonar
fi

# 3. Verify that both the user and group now exist.
getent group sonar
id sonar
```

Expected: `getent group sonar` prints the `sonar` group, and `id sonar` prints the `sonar` user and group. Do not continue to `chown` or `sudo -u sonar` unless both checks succeed. If the group command fails, capture its full error before proceeding.

**If you already extracted the ZIP and `/opt/sonarqube` exists, do not extract or move it again.** Continue with the ownership and permission commands below. Otherwise, check the paths and extract the verified archive:

```bash
ls -ld /opt/sonarqube /opt/sonarqube-26.10.0.133078 2>/dev/null || true
unzip -t /tmp/sonarqube.zip | tail -5
```

If `/opt/sonarqube-26.10.0.133078` is already present, inspect it before proceeding; do not extract over it blindly. For a new extraction, run:

```bash
sudo systemctl stop sonarqube 2>/dev/null || true

# Preserve an existing installation rather than deleting it.
if [ -d /opt/sonarqube ]; then
  sudo mv /opt/sonarqube "/opt/sonarqube-backup-$(date +%Y%m%d-%H%M%S)"
fi

sudo unzip -q /tmp/sonarqube.zip -d /opt/
sudo mv /opt/sonarqube-26.10.0.133078 /opt/sonarqube
```

Then apply ownership and permissions (run these whether the directory was already extracted or was extracted just now):

```bash
sudo chown -R sonar:sonar /opt/sonarqube
sudo chmod -R u+rwX,go-rwx /opt/sonarqube
sudo chmod -R u+rX /opt/sonarqube/bin
sudo chmod 750 /opt/sonarqube/bin/linux-x86-64/sonar.sh
sudo -u sonar test -x /opt/sonarqube/bin/linux-x86-64/sonar.sh \
  && echo "SonarQube startup script is executable"
```

Expected final output: `SonarQube startup script is executable`. If `sudo` itself reports an audit-plugin initialization error, stop and resolve that `sudo` issue before continuing; do not try to work around it by changing system permissions.

### 3.6 Configure SonarQube to use PostgreSQL

Make sure Nano is installed before opening the file (Step 3.2 installs it). If `nano: command not found` appears, install it now:

``` bash
sudo apt-get update
sudo apt-get install -y nano
```

Edit the configuration file:

``` bash
sudo nano /opt/sonarqube/conf/sonar.properties
```

Ensure these settings are present and uncommented. Use the same database
password you set in section 3.3:

``` properties
sonar.jdbc.username=sonar
sonar.jdbc.password=REPLACE_WITH_THE_SAME_DATABASE_PASSWORD
sonar.jdbc.url=jdbc:postgresql://127.0.0.1:5432/sonarqube
sonar.web.host=0.0.0.0
sonar.web.port=9000
```

Save in Nano with `Ctrl+O`, press `Enter`, then exit with `Ctrl+X`.
Ensure the file remains readable by the `sonar` account:

``` bash
sudo chown sonar:sonar /opt/sonarqube/conf/sonar.properties
sudo chmod 640 /opt/sonarqube/conf/sonar.properties
sudo grep -E '^(sonar.jdbc.username|sonar.jdbc.url|sonar.web.host|sonar.web.port)=' \
  /opt/sonarqube/conf/sonar.properties
```

Do not paste a database password into chat or commit it to Git. The
password line is intentionally omitted from the final verification
output.

### 3.7 Create the systemd service

Create the service file:

``` bash
sudo nano /etc/systemd/system/sonarqube.service
```

Use this service definition:

``` ini
[Unit]
Description=SonarQube Community Build
After=network.target postgresql.service
Wants=postgresql.service

[Service]
Type=forking
User=sonar
Group=sonar
ExecStart=/opt/sonarqube/bin/linux-x86-64/sonar.sh start
ExecStop=/opt/sonarqube/bin/linux-x86-64/sonar.sh stop
Restart=on-failure
RestartSec=10
LimitNOFILE=131072
LimitNPROC=8192
TimeoutStartSec=300
TimeoutStopSec=300

[Install]
WantedBy=multi-user.target
```

Save the file, then enable and start the service:

``` bash
sudo systemctl daemon-reload
sudo systemctl enable sonarqube
sudo systemctl start sonarqube
sudo systemctl status sonarqube --no-pager -l
```

The successful session showed `sonarqube.service` as `enabled` and
`active (running)`, with the SonarQube Java processes running. The first
startup can take several minutes while the database is initialized and
indexes are created.

### 3.8 Verify service, port, and local HTTP response

Run on the SonarQube VM:

``` bash
sudo systemctl is-active sonarqube
sudo ss -lntp | grep ':9000'
curl -I http://127.0.0.1:9000
```

Successful results in the project were: - `systemctl is-active` returned
`active`. - Port `9000` was listening. - `curl -I` returned
`HTTP/1.1 200`.

If startup is not successful, inspect the service and logs before
changing configuration:

``` bash
sudo journalctl -u sonarqube -n 100 --no-pager
sudo tail -n 100 /opt/sonarqube/logs/sonar.log
sudo tail -n 100 /opt/sonarqube/logs/web.log
sudo tail -n 100 /opt/sonarqube/logs/es.log
```

### 3.9 Open the SonarQube web interface

The browser successfully opened the SonarQube project-creation page at:

``` text
http://34.58.51.3:9000
```

Use the VM's **current** external IP if it has changed. In Google Cloud,
allow inbound TCP `9000` only from your own public IP (or preferably
access through private networking/VPN). Do not expose PostgreSQL port
`5432`.

On a fresh SonarQube database, the initial credentials are usually
`admin` / `admin`, followed by a forced password change. If this
database has already been initialized, use the administrator password
already configured.

### 3.10 Troubleshooting the errors seen during manual setup

**`useradd: group 'sonar' does not exist`, `chown: invalid user: 'sonar:sonar'`, or `sudo: unknown user sonar`**

The group/user creation order was incorrect or the account was not created. Return to Step 3.5 and run the group creation command first, then the user creation command, then verify with both `getent group sonar` and `id sonar`. Only repeat ownership commands after both checks succeed. Do not use `id sonar ... || sudo useradd --gid sonar ...` by itself because that does not create the group.

**`sudo: nano: command not found`**

Nano is not installed. Run `sudo apt-get update` followed by `sudo apt-get install -y nano`, then retry the editor command.

**`sudo: error initializing audit plugin sudoers_audit`**

This is a `sudo` configuration/plugin problem, not a SonarQube ZIP problem. Do not edit `/etc/sudoers` or remove audit settings based on a guess. If it persists across normal `sudo` commands, preserve the full error output and use the VM's authorized administrator/root access or your organization's Ubuntu support process to inspect the sudo configuration. Continue only when `sudo` works normally.

### 3.11 Security and maintenance notes

-   The database password appeared in terminal screenshots during setup.
    Treat it as exposed and rotate it to a new strong password; update
    `sonar.jdbc.password` to match, then restart SonarQube.
-   Do not run SonarQube as `root`; this installation uses the dedicated
    `sonar` account.
-   Keep PostgreSQL bound to localhost/private networking and do not
    create a public firewall rule for `5432`.
-   Use the SonarQube version's official requirements when resizing or
    upgrading. Back up the PostgreSQL database and SonarQube
    configuration/data before upgrades.
-   This runbook records the tested version shown in the session. For a
    future version, verify its Java and database compatibility before
    changing the binary URL.

------------------------------------------------------------------------

## 4. Connect SonarQube to Jenkins

### 4.1 Create a SonarQube token

1.  Sign in to SonarQube.
2.  Open **My Account → Security** (the exact menu may vary by version).
3.  Generate a user token for Jenkins integration.
4.  Copy it once and store it securely.

### 4.2 Add the token to Jenkins Credentials

In Jenkins:

1.  Open **Manage Jenkins → Credentials**.
2.  Select the appropriate global or folder credential store.
3.  Choose **Add Credentials**.
4.  Kind: **Secret text**.
5.  Secret: paste the SonarQube token.
6.  ID: `sonarqube-token`.
7.  Description: `SonarQube token for pipeline`.
8.  Save.

Use the exact credential ID in the Jenkinsfile. Never paste the token
directly into the Jenkinsfile or Git repository.

### 4.3 Configure the SonarQube server in Jenkins

1.  Install/verify the **SonarQube Scanner for Jenkins** plugin.
2.  Go to **Manage Jenkins → System**.
3.  Find **SonarQube servers** and add a server.
4.  Name: `SonarQube`.
5.  Server URL: `http://SONARQUBE_SERVER_PUBLIC_IP:9000` (use the actual
    reachable IP or private DNS).
6.  Authentication token: select the Jenkins credential
    `sonarqube-token`.
7.  Save.

If the Jenkins and SonarQube VMs are on the same VPC, prefer a private
IP for server-to-server traffic and allow TCP `9000` only from Jenkins.
Confirm connectivity from the Jenkins VM:

``` bash
curl -I --connect-timeout 5 http://SONARQUBE_SERVER_PRIVATE_OR_PUBLIC_IP:9000
```

A successful HTTP response confirms basic connectivity, not token
validity.

### 4.4 Install/verify build tools

The Jenkins agent that executes the pipeline needs a compatible Java
runtime, Maven (if not using the Maven Wrapper), Git, and Docker access.
Check:

``` bash
java -version
mvn -version
git --version
docker --version
```

If a command is missing, install/configure it on the **actual Jenkins
agent** running the job, not automatically on the SonarQube server.

------------------------------------------------------------------------

## 5. Configure the pipeline and test a build

### 5.1 Jenkins pipeline configuration

1.  Open the relevant Jenkins pipeline job.
2.  Confirm the correct GitHub repository and branch.
3.  Confirm the job uses the expected `Jenkinsfile` or configured
    pipeline script.
4.  Ensure the pipeline uses the configured SonarQube server name
    `SonarQube` and credential ID `sonarqube-token`.
5.  Ensure Docker Hub credentials are stored in Jenkins Credentials, not
    hard-coded in the script.

The exact previously working Jenkinsfile was not included in the
screenshots available for this runbook. Do not replace a working
pipeline with a generic script without first backing up the current
script and checking the repository's build structure.

### 5.2 Maven and SonarQube analysis

For a Maven project with a valid `pom.xml`, a typical analysis command
is:

``` bash
mvn clean verify sonar:sonar \
  -Dsonar.host.url=http://SONARQUBE_SERVER_PUBLIC_IP:9000 \
  -Dsonar.token="$SONAR_TOKEN"
```

Use the Jenkins SonarQube integration (`withSonarQubeEnv('SonarQube')`)
when configured. Define the token through Jenkins Credentials; do not
print it to logs. Some project structures require a project key or
additional scanner configuration. Use the actual project's
configuration.

### 5.3 Run the test build

1.  Save the Jenkins configuration.
2.  Select **Build Now**.
3.  Open **Console Output**.
4.  Confirm checkout, Maven tests/package, SonarQube analysis, Docker
    build, and Docker push stages finish successfully.
5.  In SonarQube, open the project and verify the new analysis appears.

If the build fails, use the first meaningful error in Console Output
rather than changing multiple settings at once.

------------------------------------------------------------------------

## 6. Create a SonarQube webhook for Jenkins

A webhook is needed when the pipeline uses Jenkins' `waitForQualityGate`
step.

1.  In SonarQube, open **Administration → Configuration → Webhooks**
    (menu wording varies by version).
2.  Add a webhook.
3.  Name: `Jenkins`.
4.  URL:

``` text
http://JENKINS_SERVER_REACHABLE_ADDRESS:8080/sonarqube-webhook/
```

5.  Save and use SonarQube's delivery/test details, if available, to
    confirm delivery.

Use the Jenkins address reachable **from the SonarQube server**. The
trailing slash in `/sonarqube-webhook/` matters. Configure firewalls so
SonarQube can reach Jenkins on TCP `8080` (or the actual Jenkins HTTP
port). Restrict source IPs where feasible.

### 6.1 Optional Quality Gate stage

If the Jenkins pipeline has configured SonarQube integration and a
webhook, a Maven pipeline can use:

``` groovy
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

This is a pattern, not a complete replacement Jenkinsfile. It assumes
the SonarQube server is named `SonarQube` in Jenkins and the project
uses Maven. Place the stages in the correct order for your current
pipeline.

------------------------------------------------------------------------

## 7. Verify the Docker image pushed by Jenkins

The image used during the GKE deployment shown in Cloud Shell was:

``` text
docker.io/devopsbyrushi/octsecurebankcicdpipeline:v1
```

In Jenkins, verify the Docker push stage completed. In Docker Hub,
verify that the repository and tag exist.

**Important:** A reusable tag such as `v1` can be overwritten. For
reliable release tracking, prefer a unique tag such as the Jenkins build
number or Git commit SHA. The Kubernetes deployment must reference the
exact tag pushed by the successful build.

------------------------------------------------------------------------

## 8. Create/access GKE using Google Cloud Shell

The cluster was created with `gcloud`, not Terraform, in the session
captured by the screenshots. Google Kubernetes Engine is **GKE**, not
Amazon EKS.

### 8.1 Set the project and inspect the cluster

Run in **Google Cloud Shell**:

``` bash
gcloud config set project august2027
gcloud container clusters list
```

The captured cluster details were:

-   Name: `rushi-gke`
-   Location: `us-central1-a`
-   Status: `RUNNING`
-   Two nodes were shown as `Ready`.

### 8.2 Get cluster credentials

``` bash
gcloud container clusters get-credentials rushi-gke \
  --zone us-central1-a \
  --project august2027
```

Verify access:

``` bash
kubectl get nodes
kubectl get namespaces
```

Do not create a second cluster if `rushi-gke` already exists and is
healthy.

------------------------------------------------------------------------

## 9. Deploy the Docker image to Kubernetes Pods

### 9.1 Check the existing deployment

``` bash
kubectl get deployments
kubectl get pods -o wide
kubectl get deployment flipkart-app -o yaml
```

The existing deployment was named `flipkart-app`, with two replicas. Its
container image was shown as
`docker.io/devopsbyrushi/octsecurebankcicdpipeline:v1`.

### 9.2 Set the image and pull policy

For the image shown in this project:

``` bash
kubectl set image deployment/flipkart-app \
  octsecurebankcicdpipeline=docker.io/devopsbyrushi/octsecurebankcicdpipeline:v1
```

The container name was verified in the terminal as
`octsecurebankcicdpipeline`.

To ensure Kubernetes checks Docker Hub for the tag whenever it starts a
container:

``` bash
kubectl patch deployment flipkart-app --type='json' \
  -p='[{"op":"replace","path":"/spec/template/spec/containers/0/imagePullPolicy","value":"Always"}]'
```

If this patch reports that the field is missing, use a merge patch:

``` bash
kubectl patch deployment flipkart-app --type='merge' \
  -p '{"spec":{"template":{"spec":{"containers":[{"name":"octsecurebankcicdpipeline","imagePullPolicy":"Always"}]}}}}'
```

Wait for rollout:

``` bash
kubectl rollout status deployment/flipkart-app
kubectl get pods -o wide
kubectl describe deployment flipkart-app
```

**Best practice:** For each new Jenkins build, publish a unique image
tag and update the deployment to that tag. `imagePullPolicy: Always`
helps with mutable tags, but unique tags make rollbacks and audits
safer.

### 9.3 Check application logs

``` bash
kubectl logs deployment/flipkart-app --all-containers=true --tail=100
```

The captured application logs showed Apache Tomcat starting on port
`8080` and deploying `ROOT.war`. Confirm that remains true for the
current image.

------------------------------------------------------------------------

## 10. Expose the application using a LoadBalancer Service

The following service already existed in the captured session:

-   Service: `flipkart-service`
-   Type: `LoadBalancer`
-   Cluster IP: `34.118.225.207`
-   External IP: `35.253.60.152`
-   Service port: `80`
-   Target port: `8080`

Inspect it:

``` bash
kubectl get svc flipkart-service
kubectl describe svc flipkart-service
kubectl get endpoints flipkart-service
kubectl get pods -o wide
```

If the service is missing, it can be created using:

``` bash
kubectl expose deployment flipkart-app \
  --name=flipkart-service \
  --type=LoadBalancer \
  --port=80 \
  --target-port=8080
```

Do not run that command if the service already exists; inspect and reuse
the existing service instead.

Wait for an external IP:

``` bash
kubectl get svc flipkart-service --watch
```

Stop watching with `Ctrl+C` once the IP is assigned.

### 10.1 Test the external IP

``` bash
curl -v --connect-timeout 10 http://35.253.60.152/
```

The captured terminal showed an HTTP `200` response from the external
IP, with HTML content for the Flipkart demo. That is evidence the
application was responding at that time.

------------------------------------------------------------------------

## 11. Connect the domain using Hostinger DNS

Although GoDaddy was initially mentioned, the actual DNS-management
screenshots show the domain `zetaev.com` managed in **Hostinger
hPanel**. Make DNS changes at the provider whose nameservers are
authoritative for the domain. Do not change nameservers merely to follow
this runbook.

### 11.1 DNS records

In Hostinger, open **Domains → zetaev.com → DNS / Nameservers**.

Set or verify:

  Type    Name    Target / Value    TTL
  ------- ------- ----------------- ---------------------------
  A       `@`     `35.253.60.152`   `300` or provider default
  CNAME   `www`   `zetaev.com`      `300` or provider default

The screenshot showed an existing `ALIAS @` pointing to
`zetaev.com.cdn.hstgr.net` and a `CNAME www` pointing to
`www.zetaev.com.cdn.hstgr.net`. The working change was to point the apex
and `www` to the GKE IP. Remove or replace conflicting web-hosting
records for `@` and `www`, but **preserve mail-related MX, SPF, DKIM,
DMARC, and autodiscover records**.

If the DNS interface does not allow an `A` record alongside the existing
apex `ALIAS`, remove/replace the conflicting apex web record with the A
record as supported by the provider.

### 11.2 Verify DNS resolution

Run in Cloud Shell:

``` bash
dig +short zetaev.com
dig +short www.zetaev.com
```

The screenshots eventually showed both names resolving to:

``` text
35.253.60.152
```

DNS propagation/caching may take time. Check the authoritative
nameservers if results differ.

### 11.3 Open the site in Chrome

Use:

``` text
http://zetaev.com
```

or:

``` text
http://www.zetaev.com
```

**Important:** The project was successfully opened after using
`http://zetaev.com`. Do not use `https://zetaev.com` yet: HTTPS has not
been configured in this workflow. A browser may upgrade to HTTPS
automatically; type the full `http://` URL explicitly while testing.

The existing website content hosted through Hostinger may be replaced
when the domain points to GKE. Back up any existing website files before
deleting them. DNS changes redirect traffic; they do not themselves
delete old files.

------------------------------------------------------------------------

## 12. Confirm the complete deployment

Run:

``` bash
kubectl get deployment flipkart-app
kubectl get pods -o wide
kubectl get svc flipkart-service
kubectl get endpoints flipkart-service
curl -I --connect-timeout 10 http://35.253.60.152/
curl -I --connect-timeout 10 http://zetaev.com/
```

Expected checks:

-   Deployment desired and available replicas match.
-   Pods are `Running` and `Ready`.
-   The service has an external IP.
-   Endpoints list the application Pods on port `8080`.
-   HTTP requests reach the application.
-   The domain resolves to the same external IP.

If a check fails, inspect:

``` bash
kubectl describe deployment flipkart-app
kubectl describe svc flipkart-service
kubectl get events --sort-by=.lastTimestamp
kubectl logs deployment/flipkart-app --all-containers=true --tail=100
```

------------------------------------------------------------------------

## 13. Stop the cluster to reduce cost

GKE billing can include cluster-management charges, node VM costs, load
balancer costs, and persistent storage. Merely closing Cloud Shell or
the browser does not stop resources.

### 13.1 Scale application Pods down

This leaves the cluster running but reduces application replicas:

``` bash
kubectl scale deployment flipkart-app --replicas=0
```

Restore them later with:

``` bash
kubectl scale deployment flipkart-app --replicas=2
```

The LoadBalancer may still incur charges while the service exists.

### 13.2 Resize the node pool to zero (if supported for your configuration)

First inspect node pools:

``` bash
gcloud container node-pools list \
  --cluster rushi-gke \
  --zone us-central1-a \
  --project august2027
```

For a standard GKE cluster, identify the actual node-pool name, then
resize it to zero if the pool/configuration permits:

``` bash
gcloud container clusters resize rushi-gke \
  --node-pool=YOUR_NODE_POOL_NAME \
  --num-nodes=0 \
  --zone=us-central1-a \
  --project=august2027
```

This can make workloads unavailable and may not remove all cluster or
external-resource costs. Do not run until you have confirmed the
node-pool name and intend to take the application offline.

### 13.3 Deleting the LoadBalancer Service

If you are intentionally taking the app offline and no longer need its
public endpoint:

``` bash
kubectl delete service flipkart-service
```

This normally asks GKE to release the Service's cloud load balancer and
its ephemeral external IP. Wait for cleanup and verify in Google Cloud
Console. If you later recreate the service, the IP can change and DNS
must be updated.

------------------------------------------------------------------------

## 14. Delete the GKE cluster completely

**Destructive action:** This deletes the GKE cluster and its workloads.
Back up anything you need first. Confirm the project and cluster name
before running the command.

### 14.1 Confirm target

``` bash
gcloud config get-value project
gcloud container clusters list --project august2027
```

Expected target:

``` text
Project: august2027
Cluster: rushi-gke
Zone: us-central1-a
```

### 14.2 Delete the cluster

``` bash
gcloud container clusters delete rushi-gke \
  --zone=us-central1-a \
  --project=august2027
```

Review the confirmation prompt carefully before accepting.

### 14.3 Verify deletion

``` bash
gcloud container clusters list --project august2027
```

Also inspect **Google Cloud Console → Network services → Load
balancing** and **VPC network → IP addresses** for any leftover
resources. Do not delete resources that other workloads use. If the
service used an ephemeral IP, it will generally be released during
cleanup; if a static IP was reserved separately, it may need separate
release.

**Domain reminder:** After the cluster or LoadBalancer is deleted,
`zetaev.com` will continue resolving to the old IP until DNS is changed.
Remove or update the A record when the application is intentionally
retired.

------------------------------------------------------------------------

## 15. Important operational and security notes

-   **GKE is not EKS.** The project uses Google Kubernetes Engine and
    `gcloud`, not Amazon EKS.
-   Keep Jenkins, SonarQube, Docker Hub, and database secrets in
    credential stores, not Git.
-   Do not expose PostgreSQL to the internet.
-   Restrict Jenkins and SonarQube administrative ports to trusted IPs
    or private networking.
-   The application endpoint currently uses HTTP. Do not send real
    credentials or sensitive data over it. Configure HTTPS before
    production use.
-   Prefer immutable image tags (build number or Git SHA) instead of
    overwriting `v1`.
-   Back up Jenkins configuration, SonarQube data/database, and any
    required application data before deleting servers or clusters.
-   Stopping or deleting GKE does not automatically delete the separate
    Jenkins and SonarQube VMs.

------------------------------------------------------------------------

## 16. Quick reference commands

``` bash
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

**Current successful milestone:** SonarQube/Jenkins integration and
pipeline work were discussed; GKE Pods were running; the LoadBalancer IP
was `35.253.60.152`; DNS was changed to that IP; and the application was
accessible using `http://zetaev.com`. Verify all resources and
credentials against the live environment before repeating any setup or
cleanup.
