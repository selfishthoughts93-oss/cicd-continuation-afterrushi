# Jenkins + SonarQube CI Pipeline --- Installation and Configuration Notes

**Last updated:** 24 October 2026\
**Status:** Successful end-to-end pipeline build confirmed in Jenkins
Build #13.

## 1. Objective

Integrate SonarQube running on a separate server with Jenkins, analyze
the application during the CI pipeline, enforce the SonarQube Quality
Gate, and continue with the existing Tomcat artifact copy, Docker image
build, and Docker Hub push.

The next planned phase is deployment to Amazon EKS Kubernetes pods. EKS
has **not** been configured as part of these notes.

## 2. Current architecture

``` text
GitHub repository
    |
    v
Jenkins server
  1. Git checkout
  2. Maven validate
  3. Maven install
  4. SonarQube analysis
  5. Wait for Quality Gate
  6. Copy WAR to Tomcat webapps
  7. Build Docker image
  8. Docker Hub login
  9. Push Docker image
    |
    | HTTP on TCP 9000 for analysis
    v
SonarQube server
    |
    | Webhook to Jenkins /sonarqube-webhook/
    v
Jenkins receives Quality Gate result
```

## 3. Servers and application details

  ----------------------------------------------------------------------------------------------------------------
  Item                                Value
  ----------------------------------- ----------------------------------------------------------------------------
  Jenkins URL                         `http://34.31.133.187:8080`

  Jenkins job                         `cicd-pipeline2`

  SonarQube URL                       `http://104.197.125.252:9000`

  SonarQube server name in Jenkins    `SonarQube`

  GitHub repository                   `https://github.com/devopsbyrushi/rushiDevOpsTraining-FlipkartWarDemo.git`

  GitHub Jenkins credential ID        `rushi-github-credentials`

  SonarQube project key               `flipkart-war-demo`

  SonarQube project name              `Flipkart-War-Demo`

  Docker Hub image                    `devopsbyrushi/octsecurebankcicdpipleline:v1`

  Docker Hub credential ID            `dockerhub-passwd`

  Docker Hub username                 `devopsbyrushi`

  WAR artifact                        `target/flipkart-clone.war`

  Tomcat deployment directory         `/opt/tomcat10/webapps/`

  Jenkins workspace observed          `/var/lib/jenkins/workspace/cicd-pipeline2`
  ----------------------------------------------------------------------------------------------------------------

**Cloud project note:** The SSH-in-browser URL shown in the session
included project `august2027` and VM zone `us-central1-a`. Confirm the
intended Google Cloud project and zone in the Cloud Console before using
these values in other infrastructure documentation.

## 4. SonarQube server setup

SonarQube was installed using Docker with PostgreSQL as its database.
The dashboard is accessible at:

`http://104.197.125.252:9000`

The SonarQube server and Jenkins run on separate servers.

### SonarQube settings configured for Jenkins

1.  A SonarQube user token was generated from **My Account → Security**.
2.  The token was stored in Jenkins Credentials as a **Secret text**
    credential with ID:
    -   `sonarqube-token`
3.  Jenkins SonarQube server configuration was added under **Manage
    Jenkins → System → SonarQube servers**:
    -   Name: `SonarQube`
    -   URL: `http://104.197.125.252:9000`
    -   Authentication token: the saved SonarQube token
4.  The SonarQube Jenkins plugin was installed.
5.  A SonarQube webhook was configured to send Quality Gate results to
    Jenkins:

`http://34.31.133.187:8080/sonarqube-webhook/`

The webhook address assumes Jenkins is reachable at `34.31.133.187:8080`
from the SonarQube server. Confirm the URL and network rules if webhook
delivery stops working.

## 5. Jenkins prerequisites and configuration

### Plugins and credentials

-   Install the **SonarQube Scanner for Jenkins** plugin.
-   Configure the SonarQube server named exactly `SonarQube`.
-   Keep the GitHub credential ID `rushi-github-credentials`.
-   Keep the Docker Hub secret-text credential ID `dockerhub-passwd`.
-   Do not put SonarQube or Docker Hub tokens directly in the
    Jenkinsfile.

### Docker permissions

The Docker build initially failed with:

``` text
permission denied while trying to connect to the Docker API
at unix:///var/run/docker.sock
```

Docker was running. Jenkins needed permission to access the Docker
socket.

Commands used on the **Jenkins server**:

``` bash
sudo systemctl status docker
sudo usermod -aG docker jenkins
sudo systemctl restart jenkins
sudo systemctl status jenkins
sudo -u jenkins -H docker ps
```

The `docker ps` command succeeded under the Jenkins account and
displayed the container table. An empty table is normal when no
containers are running.

**Security note:** Membership in the `docker` group effectively grants
root-level control of the host. Grant it only to trusted accounts.

### Workspace permission repair

Maven previously failed with a permission error because commands had
been run with `sudo`, which could leave root-owned files in the Jenkins
workspace.

The workspace was repaired using:

``` bash
sudo chown -R jenkins:jenkins /var/lib/jenkins/workspace/cicd-pipeline2
sudo rm -rf /var/lib/jenkins/workspace/cicd-pipeline2/target
```

The `target` directory contains generated build output and was removed
so Maven could recreate it with the correct ownership. The GitHub source
code was not removed.

Maven stages should run without `sudo`:

``` groovy
sh 'mvn validate'
sh 'mvn clean install'
```

The pipeline still uses `sudo cp` for copying the WAR into the Tomcat
webapps directory. Jenkins must have the appropriate sudo permission for
this operation.

## 6. Final Jenkins Pipeline script

The following is the pipeline version used after the SonarQube and
Docker permissions fixes. It preserves the existing build and publish
stages.

``` groovy
node {
    stage('github code checkout') {
        git credentialsId: 'rushi-github-credentials',
            url: 'https://github.com/devopsbyrushi/rushiDevOpsTraining-FlipkartWarDemo.git'
    }

    stage('maven build phases-mvn validate') {
        sh 'mvn validate'
    }

    stage('mvn build phase -mvn install') {
        sh 'mvn clean install'
    }

    stage('SonarQube Analysis') {
        withSonarQubeEnv('SonarQube') {
            sh '''
                mvn org.sonarsource.scanner.maven:sonar-maven-plugin:5.3.0.6276:sonar \
                  -Dsonar.projectKey=flipkart-war-demo \
                  -Dsonar.projectName=Flipkart-War-Demo
            '''
        }
    }

    stage('SonarQube Quality Gate') {
        timeout(time: 5, unit: 'MINUTES') {
            def qualityGate = waitForQualityGate()

            if (qualityGate.status != 'OK') {
                error "Quality Gate failed: ${qualityGate.status}"
            }
        }
    }

    stage('copying artifact to tomcat webapps') {
        sh '''
            sudo cp "$WORKSPACE/target/flipkart-clone.war" \
                /opt/tomcat10/webapps/
        '''
    }

    stage('creating an docker image') {
        sh 'docker build -t devopsbyrushi/octsecurebankcicdpipleline:v1 .'
    }

    stage('dockerlogin with rushi cred') {
        withCredentials([
            string(
                credentialsId: 'dockerhub-passwd',
                variable: 'rushidockerhubpassword'
            )
        ]) {
            sh '''
                echo "$rushidockerhubpassword" |
                docker login -u devopsbyrushi --password-stdin
            '''
        }
    }

    stage('pushing mylatestimages to dockerhub') {
        sh 'docker push devopsbyrushi/octsecurebankcicdpipleline:v1'
    }
}
```

### Notes about this pipeline

-   The SonarScanner Maven plugin is invoked by its fully qualified
    Maven coordinates to avoid the earlier
    `No plugin found for prefix 'sonar'` error.
-   Scanner version `5.3.0.6276` must be compatible with the Java
    runtime used by Maven on the Jenkins server. If Java compatibility
    errors appear, select a scanner version compatible with the
    installed Java version.
-   The Quality Gate stage waits up to five minutes for the result. It
    requires the Jenkins SonarQube plugin and a working webhook.
-   A Quality Gate result other than `OK` stops later stages.
-   The Tomcat copy, Docker build, Docker login, and Docker push remain
    after the Quality Gate.
-   The image tag is currently fixed at `v1`; subsequent builds
    overwrite that tag in the registry. Consider adding unique build
    tags in a future improvement.

## 7. Troubleshooting history and resolutions

### Issue A: Maven could not find the `sonar` plugin prefix

Error:

``` text
No plugin found for prefix 'sonar' in the current project
```

Resolution: Invoke the Maven plugin using its fully qualified
coordinates:

``` bash
mvn org.sonarsource.scanner.maven:sonar-maven-plugin:5.3.0.6276:sonar
```

### Issue B: Maven WAR packaging permission denied

Error:

``` text
Error assembling WAR:
target/maven-archiver/pom.properties (Permission denied)
```

Cause: Build output could have been created by earlier `sudo mvn`
commands, leaving root-owned files.

Resolution: - Change workspace ownership to `jenkins:jenkins`. - Remove
the generated `target` directory. - Run Maven as the Jenkins user
without `sudo`.

### Issue C: Docker socket permission denied

Error:

``` text
permission denied while trying to connect to the Docker API
at unix:///var/run/docker.sock
```

Resolution: - Add the Jenkins service user to the `docker` group. -
Restart Jenkins to refresh group membership. - Verify with
`sudo -u jenkins -H docker ps`.

### Issue D: Quality Gate result

The Jenkins Console Output confirmed:

``` text
SonarQube task status is 'SUCCESS'
Quality gate is 'OK'
```

This confirms that the analysis report was processed and the Quality
Gate passed for that build.

## 8. Confirmed successful result

On 24 October 2026, Jenkins Build **#13** completed successfully.

The Stage View showed all nine stages in green:

1.  GitHub code checkout
2.  Maven validate
3.  Maven install
4.  SonarQube Analysis
5.  SonarQube Quality Gate
6.  Copy artifact to Tomcat webapps
7.  Create Docker image
8.  Docker Hub login
9.  Push image to Docker Hub

The displayed full run time was approximately one minute and three
seconds.

This confirms that the current CI flow---including SonarQube analysis
and Quality Gate, WAR copy, Docker build, and Docker Hub push---was
successful for that build.

## 9. How to verify after future changes

### Jenkins

Open:

`http://34.31.133.187:8080/job/cicd-pipeline2/`

Check the latest build's Stage View and Console Output.

### SonarQube

Open:

`http://104.197.125.252:9000/projects`

Open project `Flipkart-War-Demo` (project key `flipkart-war-demo`) and
review the latest analysis and Quality Gate status.

### Docker Hub

Check the repository:

`https://hub.docker.com/r/devopsbyrushi/octsecurebankcicdpipleline`

Confirm that the `v1` tag exists and corresponds to the latest build.

## 10. Next phase: Amazon EKS (not yet completed)

The next planned task is to extend the pipeline to deploy the
successfully built image to Amazon EKS.

Before implementing that phase, decide and document: - AWS account and
region - EKS cluster name - How Jenkins will authenticate to AWS and
Kubernetes - Whether the pipeline will deploy using `kubectl` or another
deployment method - Kubernetes namespace, Deployment, Service, and
ingress/load-balancer approach - Image-tag strategy and rollback plan

Do not treat EKS deployment as completed yet; the confirmed success so
far is the nine-stage Jenkins CI pipeline through Docker Hub push.
