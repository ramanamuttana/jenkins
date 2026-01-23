# Jenkins Pipeline for Petclinic

This repository contains a Jenkins pipeline (see the `jenkinsfile`) that builds, tests, analyzes, and packages the Petclinic sample application, and then builds a Docker image.

Reference: [jenkinsfile](https://github.com/ramanamuttana/jenkins/blob/69bfc79541b31bbf37d94386f4a5ac04013e9eab/jenkinsfile)

## Overview

The declarative Jenkins pipeline performs the following high-level steps:

- Checkout source from the Petclinic Git repository (branch: `main`)
- Compile code with Maven
- Run unit tests (including an additional JaCoCo test run that excludes certain packages)
- Run SonarQube analysis via `sonar-scanner`
- Run OWASP Dependency Check and publish the report
- Build the Maven artifact (`mvn clean install`)
- Verify Docker access and build a Docker image

## Pipeline stages

1. Git Checkout
2. Code Compile (`mvn clean compile`)
3. Unit Tests (`mvn test`)
4. Test with JaCoCo (`mvn test -Djacoco.excludes=org/hibernate/proxy/**`)
5. SonarQube Analysis (via configured SonarQube server and `sonar-scanner`)
6. OWASP Dependency Check (publish `dependency-check-report.xml`)
7. Artifact Build (`mvn clean install`)
8. Check Docker (verify docker socket and permissions)
9. Docker Build (build image using `docker build -t image1 .`)

## Prerequisites (Jenkins configuration)

Make sure your Jenkins master/agent(s) have the following configured:

- Tools (configured in Jenkins -> Global Tool Configuration)
  - JDK with the label `jdk17`
  - Maven with the label `maven3`
  - SonarScanner installation labeled `sonar-scanner`
  - (Optional) Docker installation entry (toolName used in pipeline is `docker`)

- Jenkins plugins
  - Pipeline
  - Pipeline: Groovy
  - SonarQube Scanner for Jenkins
  - OWASP Dependency-Check Plugin (dependency-check)
  - Docker Pipeline
  - Any Git plugin you use (usually Git plugin)

- SonarQube server configured in Jenkins Global Configuration with the name `sonar-server` (used in `withSonarQubeEnv('sonar-server')`)

- OWASP Dependency-Check installation configured in Jenkins with the name `DP-Check` (used as `odcInstallation`)

- Credentials / IDs
  - A credentials entry for Docker registry used in `withDockerRegistry(credentialsId: '16bda58e-7358-4', ...)` — replace with your own credentials ID
  - If your SonarQube server requires authentication you may also need to configure a token/credential and set it up in Jenkins/Sonar configuration

## Environment variables used in the pipeline

- `SCANNER_HOME` — set from the installed tool `sonar-scanner`:
  - The pipeline sets `SCANNER_HOME = tool 'sonar-scanner'` and then runs `$SCANNER_HOME/bin/sonar-scanner ...`

## Important configuration details / notes

- Git checkout: the pipeline currently pulls from `https://github.com/jaiswaladi246/Petclinic.git` (branch `main`). Update this URL to point to your repo if required.
- SonarQube:
  - The pipeline calls sonar-scanner with:
    - `-Dsonar.projectName=Petclinic`
    - `-Dsonar.projectKey=Petclinic`
    - `-Dsonar.java.binaries=.`
    - `-Dsonar.host.url=http://172.17.0.3:9000`
  - Update `sonar.host.url`, `sonar.projectKey`, and `sonar.projectName` as needed for your environment.
- JaCoCo:
  - The pipeline runs a second test invocation with `-Djacoco.excludes=org/hibernate/proxy/**`. Adjust excludes as needed.
- OWASP Dependency Check:
  - The pipeline runs `dependencyCheck` and publishes results with `dependencyCheckPublisher` looking for `**/dependency-check-report.xml`. Ensure this path matches where the plugin writes reports in your job workspace.
- Docker:
  - The pipeline checks for Docker access by running `docker ps` and verifying `/var/run/docker.sock`. Ensure the Jenkins agent has Docker installed and the executing user can access the Docker socket or configure a dedicated Docker agent.
  - The Docker build command tags the image as `image1`. Update the tag to your desired repository/name and push step if needed.
- Credentials and IDs in the pipeline (example placeholders):
  - `sonar-server` — SonarQube server config name in Jenkins
  - `DP-Check` — OWASP Dependency-Check installation name
  - `16bda58e-7358-4` — Docker registry credentials ID (replace with your own)

## How to use

1. Create a new Pipeline job in Jenkins.
2. Add / configure the required global tools and plugin settings listed above.
3. In the Pipeline job:
   - Option A: Use "Pipeline script from SCM" and point to the repo that contains this `jenkinsfile`.
   - Option B: Paste the pipeline script content from the [jenkinsfile](https://github.com/ramanamuttana/jenkins/blob/69bfc79541b31bbf37d94386f4a5ac04013e9eab/jenkinsfile) into the job's Pipeline script field.
4. Make sure the agent you run on has Java, Maven, Docker (if building images) and access to execute the Sonar scanner and dependency-check tool if those stages are required.
5. Run the job and review console output for each stage. Reports from SonarQube and OWASP Dependency-Check will be available per their plugin/configuration.

## Customization tips

- Update the Git repository URL and branch if you're working with a fork or different source.
- Replace IP `172.17.0.3:9000` with your SonarQube server's URL or a configured server in Jenkins.
- Change Docker image name/tag and add a push step to publish to a registry (e.g., Docker Hub or a private registry).
- Add post-build actions, artifact archives, or promotion steps as your delivery process requires.

## Troubleshooting

- Sonar analysis fails: verify `SCANNER_HOME` points to a valid sonar-scanner installation and that `sonar-server` is configured in Jenkins with correct URL and authentication.
- OWASP Dependency-Check report missing: check plugin logs, ensure DP-Check is installed and that the report path `**/dependency-check-report.xml` matches the generated report path.
- Docker permission errors: ensure the Jenkins agent user is in the `docker` group or use a dedicated Docker agent/container with proper access.
- Maven build/test failures: run the same Maven commands locally to reproduce issues: `mvn clean compile`, `mvn test`, and `mvn clean install`.

## License

This repository contains CI/CD configuration and should follow the licensing of the source application (Petclinic). Add a LICENSE file as appropriate for your project.
