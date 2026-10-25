# SonarQube → Jenkins CI → GKE Deployment → Domain Hosting Runbook

**Project:** `august2027`\
**GKE cluster:** `rushi-gke`\
**Cluster location:** `us-central1-a` (zonal)\
**Kubernetes namespace:** `default`\
**Deployment:** `flipkart-app`\
**Service:** `flipkart-service`\
**Application image:**
`docker.io/devopsbyrushi/octsecurebankcicdpipleline:v1`\
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

1. Developer pushes application changes to GitHub.
2. **Phase 1:** Jenkins checks out the repository, runs Maven tests and SonarQube analysis,
   checks the Quality Gate, builds the Docker image, and pushes it to Docker Hub.
3. After the GKE cluster and its initial Deployment/Service exist, **Phase 2** can run the
   same CI stages and then update the running GKE Deployment to the new image tag.
4. A Kubernetes `LoadBalancer` Service exposes the Pods through an external IP.
5. DNS records point `zetaev.com` and `www.zetaev.com` to the GKE external IP.
6. The application is currently accessed over **HTTP**. HTTPS has not yet been configured.

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

  Docker Hub image                    `docker.io/devopsbyrushi/octsecurebankcicdpipleline:v1`

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

## 5. Phase 1 — Corrected pipeline based on your existing Jenkinsfile

This section follows your existing pipeline and repository:

- Git repository: `https://github.com/devopsbyrushi/rushiDevOpsTraining-FlipkartWarDemo.git`
- Git credential ID: `rushi-github-credentials`
- Docker Hub credential ID: `dockerhub-passwd` (Secret text)
- Docker Hub username: `devopsbyrushi`
- WAR artifact: `target/flipkart-clone.war`
- Tomcat webapps directory: `/opt/tomcat10/webapps/`
- Docker Hub repository spelling in your script: `devopsbyrushi/octsecurebankcicdpipleline`

**Repository spelling note:** Your script spells the Docker Hub repository `octsecurebankcicdpipleline`
(`...cicd + pipleline...`). This is preserved below to match your actual script. Confirm that this exact
repository exists in Docker Hub. If your real repository is spelled `octsecurebankcicdpipeline`,
change the image name consistently in both Phase 1 and Phase 2 before running.

### 5.1 Why the old pipeline needed correction

Your original code uses a Scripted Pipeline (`node { ... }`), so its stages should contain commands
directly; it should not use Declarative `steps {}` blocks. The original Docker login/push section also
had mismatched braces and a missing `{` after the push stage name. The corrected script below uses
valid Scripted Pipeline structure.

It also avoids hard-coding `/var/lib/jenkins/workspace/cicd-pipeline`, because the actual job name
may be `cicd-pipeline2` or another name. `$WORKSPACE` is used instead.

### 5.2 Jenkins agent prerequisites

The Jenkins agent executing this pipeline must have:

```bash
java -version
mvn -version
git --version
docker --version
```

The agent must be able to access the Docker daemon. The Tomcat copy stage assumes that Tomcat 10 is
installed on the same machine as the Jenkins agent and that `/opt/tomcat10/webapps/` is its correct
deployment directory. If Tomcat is on a different VM, this local `cp` command will not deploy to it;
use a separate authenticated deployment method instead.

Configure these Jenkins items:

1. **Git credential:** ID `rushi-github-credentials`, with access to the GitHub repository.
2. **Docker Hub credential:** ID `dockerhub-passwd`, type **Secret text**, containing a Docker Hub access token.
3. **SonarQube server:** Under **Manage Jenkins → System → SonarQube servers**, configure a server named
   exactly `SonarQube`, its reachable URL, and the SonarQube token credential.
4. Install/verify the **SonarQube Scanner for Jenkins** plugin.
5. Configure the webhook in Section 6 if you want the Quality Gate to block the image push.

### Maven SonarScanner fix for this Jenkins job

The console error `No plugin found for prefix 'sonar'` occurs while Maven tries to resolve the shorthand
`sonar:sonar` prefix. This is not a SonarQube server-startup error. The pipeline below avoids prefix
resolution by invoking the Sonar Maven plugin with its fully qualified coordinates and a pinned version.
The Jenkins agent still needs outbound access to Maven Central and the checked-out project must contain
a valid `pom.xml`.

### 5.3 Complete Phase 1 Scripted Pipeline — GitHub to Docker Hub

Replace the existing job script with this complete script. It keeps your existing stages, corrects the
syntax and workspace path, adds SonarQube analysis/Quality Gate using the already configured SonarQube
server, and publishes an immutable Jenkins build tag plus `v1`.

If you want to match the original behavior exactly without SonarQube analysis, remove the two clearly
labelled `SonarQube analysis` and `Quality Gate` stages only after deciding that code analysis is not
part of this pipeline.

```groovy
node {
    def imageName = 'devopsbyrushi/octsecurebankcicdpipleline'
    def imageTag = "build-${env.BUILD_NUMBER}"

    stage('GitHub code checkout') {
        git credentialsId: 'rushi-github-credentials',
            url: 'https://github.com/devopsbyrushi/rushiDevOpsTraining-FlipkartWarDemo.git'
    }

    stage('Maven validate') {
        sh 'mvn -B validate'
    }

    stage('Maven install and tests') {
        // "install" runs the earlier Maven lifecycle phases as well.
        sh 'mvn -B install'
        sh """
            set -eu
            test -f "\$WORKSPACE/target/flipkart-clone.war" || {
              echo "ERROR: target/flipkart-clone.war was not created."
              echo "Check the Maven build output and the WAR artifact name."
              exit 1
            }
        """
    }

    stage('Copy WAR to local Tomcat webapps') {
        // This assumes Tomcat 10 is installed on this same Jenkins agent.
        sh """
            set -eu
            sudo test -d /opt/tomcat10/webapps || {
              echo "ERROR: /opt/tomcat10/webapps does not exist on this agent."
              echo "Confirm Tomcat's installation path or use a remote deployment method."
              exit 1
            }
            sudo install -m 0644 \
              "\$WORKSPACE/target/flipkart-clone.war" \
              /opt/tomcat10/webapps/flipkart-clone.war
        """
    }

    stage('SonarQube analysis') {
        withSonarQubeEnv('SonarQube') {
            sh 'mvn -B org.sonarsource.scanner.maven:sonar-maven-plugin:5.2.0.4988:sonar'
        }
    }

    stage('SonarQube Quality Gate') {
        timeout(time: 10, unit: 'MINUTES') {
            waitForQualityGate abortPipeline: true
        }
    }

    stage('Build Docker image') {
        sh """
            set -eu
            docker build \
              -t ${imageName}:${imageTag} \
              -t ${imageName}:v1 \
              .
        """
    }

    stage('Docker Hub login and push') {
        withCredentials([string(
            credentialsId: 'dockerhub-passwd',
            variable: 'rushidockerhubpassword'
        )]) {
            sh """
                set -eu
                set +x
                printf '%s' "\$rushidockerhubpassword" |
                  docker login --username devopsbyrushi --password-stdin
                docker push ${imageName}:${imageTag}
                docker push ${imageName}:v1
                docker logout
            """
        }
    }

    stage('Verify pushed image tags') {
        echo "Pushed image: ${imageName}:${imageTag}"
        echo "Compatibility tag: ${imageName}:v1"
    }
}
```

### 5.4 Important notes about this pipeline

- **No `sudo` for Maven:** Maven runs as the Jenkins user so the workspace and `target/` files do not
  become root-owned. The only `sudo` commands are for writing to the protected Tomcat directory.
- **Tomcat stage:** This copies the WAR to Tomcat on the same machine. It does not copy the WAR into
  the Docker image, and it does not deploy to GKE. The Docker build uses the repository root as its
  build context, so confirm that the repository's `Dockerfile` is designed for this context.
- **SonarQube analysis:** `withSonarQubeEnv('SonarQube')` requires the Jenkins plugin and a server
  configuration with that exact name. `mvn sonar:sonar` assumes the project can be analyzed by Maven.
- **Quality Gate:** This stage requires the SonarQube webhook described in Section 6. Configure it
  before running the pipeline; otherwise the stage may time out.
- **Build tags:** Each run pushes `build-<Jenkins build number>` for traceability and also updates `v1`.
  Use the build-specific tag for GKE deployments and rollback. `v1` is mutable.
- **Credentials:** `dockerhub-passwd` must be a Jenkins **Secret text** credential holding a Docker Hub
  access token. Never hard-code the token into the Jenkinsfile.
- **Repository access:** The Git credential ID must exactly match `rushi-github-credentials`.

### 5.5 Configure and run Phase 1

1. Open **Jenkins → your pipeline job → Configure**.
2. Select **Pipeline script** and replace the old code completely. Do not wrap this Scripted Pipeline
   inside another `pipeline {}` or add Declarative `steps {}`.
3. Save and select **Build Now**.
4. Confirm the stages complete in order:

   `GitHub code checkout → Maven validate → Maven install and tests → Copy WAR to local Tomcat webapps → SonarQube analysis → SonarQube Quality Gate → Build Docker image → Docker Hub login and push → Verify pushed image tags`

5. Confirm Docker Hub contains both `build-<BUILD_NUMBER>` and `v1` tags for the exact repository name.
6. Confirm the WAR exists at `/opt/tomcat10/webapps/flipkart-clone.war` on the Jenkins/Tomcat host.

### 5.6 Common Phase 1 errors

- **`No such DSL method 'steps'`**: the old job mixed Declarative and Scripted Pipeline syntax. Replace the whole job script with the Scripted Pipeline above.
- **Git checkout authentication error**: verify `rushi-github-credentials` and repository access.
- **`mvn: command not found`**: install Maven on the Jenkins agent or configure the Jenkins Maven tool and use it consistently.
- **Maven permission denied / root-owned workspace**: avoid `sudo mvn`; correct workspace ownership if earlier runs created root-owned files.
- **WAR not found**: check Maven output and the artifact name. The script expects `target/flipkart-clone.war`.
- **Tomcat directory not found**: verify Tomcat is installed on the same agent and that `/opt/tomcat10/webapps/` is correct.
- **`withSonarQubeEnv` not found**: install/enable the SonarQube Scanner for Jenkins plugin and configure the server name as `SonarQube`.
- **`No plugin found for prefix 'sonar'`**: the short `sonar:sonar` goal could not resolve the plugin prefix from Maven metadata. The updated scripts use the fully qualified plugin coordinates `org.sonarsource.scanner.maven:sonar-maven-plugin:5.2.0.4988:sonar`. Ensure the Jenkins agent can reach Maven Central and that the project root contains `pom.xml`.
- **Quality Gate timeout**: verify the webhook in Section 6 and that SonarQube can reach Jenkins.
- **Docker permission denied**: verify Docker is running and the Jenkins agent user can use the Docker daemon.
- **Docker Hub push denied**: verify the `dockerhub-passwd` Secret text credential, token validity, repository spelling, and push permissions.

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

### 6.1 Quality Gate stages used in this runbook

The Phase 1 and Phase 2 scripts use **Scripted Pipeline** syntax (`node { ... }`).
The matching stage pattern is:

```groovy
stage('SonarQube analysis') {
    withSonarQubeEnv('SonarQube') {
        sh 'mvn -B org.sonarsource.scanner.maven:sonar-maven-plugin:5.2.0.4988:sonar'
    }
}

stage('SonarQube Quality Gate') {
    timeout(time: 10, unit: 'MINUTES') {
        waitForQualityGate abortPipeline: true
    }
}
```

Do not add Declarative `steps {}` blocks inside these Scripted Pipeline stages.
The server configured in Jenkins must be named exactly `SonarQube`, and the
webhook above must be reachable from the SonarQube server.

------------------------------------------------------------------------

## 7. Verify the Docker image pushed by Jenkins

The corrected Phase 1 pipeline pushes two tags:

```text
devopsbyrushi/octsecurebankcicdpipleline:build-<JENKINS_BUILD_NUMBER>
devopsbyrushi/octsecurebankcicdpipleline:v1
```

Replace `<JENKINS_BUILD_NUMBER>` with the successful Jenkins build number. Verify both tags in the
Docker Hub repository page. If the actual repository is spelled differently, update the Jenkinsfile
and the GKE deployment image consistently.

For a local pull test on a machine with Docker:

```bash
docker pull devopsbyrushi/octsecurebankcicdpipleline:v1
docker image inspect devopsbyrushi/octsecurebankcicdpipleline:v1
```

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
`docker.io/devopsbyrushi/octsecurebankcicdpipleline:v1`.

### 9.2 Set the image and pull policy

For the image shown in this project:

``` bash
kubectl set image deployment/flipkart-app \
  octsecurebankcicdpipeline=docker.io/devopsbyrushi/octsecurebankcicdpipleline:v1
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


### 9.4 Phase 2 — Full CI/CD Jenkinsfile with GKE deployment

Use this **second** Jenkinsfile only after all of the following are true:

- The GKE cluster exists and is `RUNNING`.
- The Kubernetes deployment and container already exist.
- The application image has been successfully pushed to Docker Hub by Phase 1.
- Jenkins has the `gcloud` CLI and `kubectl` installed on the agent that runs
  the pipeline.
- Jenkins has a secure Google Cloud authentication credential and permission
  to access the cluster and update the deployment.
- The SonarQube server, Docker Hub credential, and webhook are already working.

This is a complete replacement for the Phase 1 script; do not paste both
scripts into the same Jenkins job. It follows your existing Scripted Pipeline
style, repeats the Maven/WAR/Tomcat, SonarQube, and Docker Hub stages, then
deploys the newly built image to GKE. The Tomcat copy assumes Tomcat is on the
Jenkins agent host.

#### 9.4.1 Configure Google Cloud credentials in Jenkins

For this runbook, the script below uses a Google Cloud service-account JSON key
stored in Jenkins as a **Secret file** credential:

- Go to **Manage Jenkins → Credentials**.
- Add a credential of type **Secret file**.
- ID: `gcp-gke-deployer-key`
- Upload the JSON key for a dedicated service account with only the permissions
  needed to access the intended cluster and update workloads.

Do not commit the JSON key to Git, put it in the Jenkinsfile, or paste it into
the terminal output. Prefer Workload Identity Federation or another keyless
authentication method where your Jenkins environment supports it. If using a
service-account key, protect and rotate it according to your organization's
security policy.

The Google Cloud service account must have the required IAM permissions for
the intended project/cluster, and Kubernetes authorization must allow it to
read and update the target deployment. Merely creating a service-account key
does not automatically grant Kubernetes workload permissions.

#### 9.4.2 Verify names before using the script

Run these commands in **Google Cloud Shell** after connecting to the cluster:

```bash
gcloud config set project august2027
gcloud container clusters get-credentials rushi-gke \
  --zone us-central1-a \
  --project august2027

kubectl get deployments -n default
kubectl get deployment flipkart-app -n default \
  -o jsonpath='{.spec.template.spec.containers[*].name}{"\n"}'
kubectl get service flipkart-service -n default
```

The values below follow the resource names recorded earlier in this runbook.
If your live output differs, update the corresponding Jenkinsfile values first.

#### 9.4.3 Complete Phase 2 Scripted Pipeline — existing workflow plus GKE deployment

This version keeps your original `node { ... }` style and the same Git, Docker Hub, Maven, WAR,
and Tomcat stages. It then updates the existing GKE Deployment after the image push. It is a
complete replacement for the Phase 1 script, not an extra block to append to it.

The image name below preserves the repository spelling in your original script. Change it only
if you confirm the actual Docker Hub repository uses a different spelling.

```groovy
node {
    def imageName = 'devopsbyrushi/octsecurebankcicdpipleline'
    def imageTag = "build-${env.BUILD_NUMBER}"

    stage('GitHub code checkout') {
        git credentialsId: 'rushi-github-credentials',
            url: 'https://github.com/devopsbyrushi/rushiDevOpsTraining-FlipkartWarDemo.git'
    }

    stage('Maven validate') {
        sh 'mvn -B validate'
    }

    stage('Maven install and tests') {
        sh 'mvn -B install'
        sh """
            set -eu
            test -f "\$WORKSPACE/target/flipkart-clone.war"
        """
    }

    stage('Copy WAR to local Tomcat webapps') {
        // Retained from the current workflow; this assumes Tomcat is on this same Jenkins agent.
        sh """
            set -eu
            sudo test -d /opt/tomcat10/webapps
            sudo install -m 0644 \
              "\$WORKSPACE/target/flipkart-clone.war" \
              /opt/tomcat10/webapps/flipkart-clone.war
        """
    }

    stage('SonarQube analysis') {
        withSonarQubeEnv('SonarQube') {
            sh 'mvn -B org.sonarsource.scanner.maven:sonar-maven-plugin:5.2.0.4988:sonar'
        }
    }

    stage('SonarQube Quality Gate') {
        timeout(time: 10, unit: 'MINUTES') {
            waitForQualityGate abortPipeline: true
        }
    }

    stage('Build Docker image') {
        sh """
            set -eu
            docker build \
              -t ${imageName}:${imageTag} \
              -t ${imageName}:v1 \
              .
        """
    }

    stage('Docker Hub login and push') {
        withCredentials([string(
            credentialsId: 'dockerhub-passwd',
            variable: 'rushidockerhubpassword'
        )]) {
            sh """
                set -eu
                set +x
                printf '%s' "\$rushidockerhubpassword" |
                  docker login --username devopsbyrushi --password-stdin
                docker push ${imageName}:${imageTag}
                docker push ${imageName}:v1
                docker logout
            """
        }
    }

    stage('Deploy to GKE') {
        withCredentials([file(
            credentialsId: 'gcp-gke-deployer-key',
            variable: 'GCP_KEY_FILE'
        )]) {
            sh """
                set -eu

                command -v gcloud >/dev/null || {
                  echo "ERROR: gcloud CLI is not installed on this Jenkins agent."
                  exit 1
                }
                command -v kubectl >/dev/null || {
                  echo "ERROR: kubectl is not installed on this Jenkins agent."
                  exit 1
                }

                gcloud auth activate-service-account \
                  --key-file="\$GCP_KEY_FILE" \
                  --project=august2027

                gcloud container clusters get-credentials rushi-gke \
                  --zone us-central1-a \
                  --project august2027

                kubectl get deployment flipkart-app --namespace default

                kubectl set image \
                  deployment/flipkart-app \
                  octsecurebankcicdpipeline=${imageName}:${imageTag} \
                  --namespace default

                kubectl rollout status deployment/flipkart-app \
                  --namespace default \
                  --timeout=5m

                kubectl get deployment flipkart-app \
                  --namespace default \
                  -o jsonpath='{.spec.template.spec.containers[*].image}{"\\n"}'
                kubectl get pods --namespace default -o wide
                kubectl get service flipkart-service --namespace default
            """
        }
    }
}
```

**Before running Phase 2, verify the actual Kubernetes container name.** The script assumes the
container inside `deployment/flipkart-app` is named `octsecurebankcicdpipeline`. Run this in
Cloud Shell:

```bash
kubectl get deployment flipkart-app -n default \
  -o jsonpath='{.spec.template.spec.containers[*].name}{"\n"}'
```

If the returned name is different, replace `octsecurebankcicdpipeline` in the `kubectl set image`
command with the exact returned container name. This container name is not necessarily the same as
the Docker Hub repository name.

#### 9.4.4 Validate Phase 2

1. Replace the Phase 1 pipeline with the Phase 2 script only after the GKE
   prerequisites above are complete.
2. Confirm the Jenkins credential IDs exactly match `dockerhub-credentials`
   and `gcp-gke-deployer-key`.
3. Run **Build Now** and verify that the `Deploy to GKE` stage completes.
4. Verify the rollout in Cloud Shell:

```bash
gcloud container clusters get-credentials rushi-gke \
  --zone us-central1-a \
  --project august2027

kubectl rollout status deployment/flipkart-app -n default
kubectl get deployment flipkart-app -n default \
  -o jsonpath='{.spec.template.spec.containers[*].image}{"\n"}'
kubectl get pods -n default -o wide
```

The image should end with the Jenkins build number that just succeeded, not
a hard-coded `v1` tag.

**Important:** If the deployment uses a different container name, namespace,
project, cluster location, or deployment name, update the environment values
before running the Phase 2 script. The deployment stage updates an existing
Deployment; it does not create the GKE cluster or create the initial
Kubernetes Deployment/Service.


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
  octsecurebankcicdpipeline=docker.io/devopsbyrushi/octsecurebankcicdpipleline:YOUR_NEW_TAG

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
