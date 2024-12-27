![Alt text](
# CI/CD for Docker and Kubernetes Using Jenkins

## Project Overview

In this project, we have:
1. **Jenkins** as the CI/CD tool for automating the process.
2. **Docker** for containerization of the application.
3. **Kubernetes** (using Kops) for managing containerized applications.
4. **Helm** for packaging Kubernetes applications.
5. **SonarQube** for static code analysis.
6. **Nexus** for artifact storage.
7. **DockerHub** as the container registry.

## Pre-Requisites
Before starting the setup, ensure you have the following:
- A **Jenkins** server up and running.
- A **DockerHub** account and access credentials.
- A **Kubernetes** cluster set up using **Kops**.
- **Helm** installed on your Kubernetes VM.
- A **SonarQube** server set up for code quality analysis.
- A **Nexus Repository** setup to store artifacts.

## Steps

### 1. **Continuous Integration Setup**
- Set up **Jenkins**, **SonarQube**, and **Nexus** as part of your Continuous Integration (CI) setup.

### 2. **DockerHub Account (Containerization Project)**
- Create a **DockerHub** account for storing and retrieving container images.
- Store DockerHub credentials securely in Jenkins (refer to Step 3).

### 3. **Store DockerHub Credentials in Jenkins**
- Add your DockerHub credentials (username and password) in Jenkins using **Credentials Manager**.

### 4. **Setup Docker Engine in Jenkins**
- Install **Docker** engine on the Jenkins server.
- Ensure Jenkins can interact with Docker (use Docker's built-in pipeline functionality).

### 5. **Install Plugins in Jenkins**
Install the following plugins on Jenkins:
- **Docker Pipeline Plugin**: To enable Jenkins to interact with Docker images.
- **Docker Plugin**: To allow Jenkins to control Docker operations like building and pushing images.
- **Pipeline Utility Steps Plugin**: Useful for handling various utilities within pipeline scripts.

### 6. **Create Kubernetes Cluster with Kops**
- Use **Kops** to create a Kubernetes cluster in your AWS environment. Follow the steps to create the cluster and ensure it is up and running.
- Configure your `kubectl` to interact with the newly created Kubernetes cluster.

### 7. **Install Helm in Kops VM**
- Install **Helm** on the Kops-managed VM. Helm will help you manage Kubernetes applications with Helm charts.

### 8. **Create Helm Charts**
- Define Kubernetes deployment resources using **Helm charts**. These charts will manage the Kubernetes resources like deployments, services, and namespaces.
  
### 9. **Test Helm Charts in Kubernetes Cluster**
- Deploy and test the Helm charts in your Kubernetes cluster. Ensure the application deploys correctly in a **test namespace** before proceeding with production.

### 10. **Add Kops VM as Jenkins Slave**
- Add the Kops VM as a Jenkins **slave** to run deployment-related tasks in the pipeline (especially Kubernetes-related jobs).

### 11. **Create Declarative Jenkins Pipeline Code**
- The Jenkins pipeline code defines the various stages of the CI/CD process. This includes building the project, running tests, building a Docker image, deploying the image to Kubernetes, and more.

### 12. **Update Git Repository with Required Files**
- **Helm Charts**: Store Helm charts in a Git repository.
- **Dockerfile**: Define a `Dockerfile` for containerizing your application.
- **Jenkinsfile**: Store the Jenkins pipeline code in the Git repository.

### 13. **Create Jenkins Job for the Pipeline**
- In Jenkins, create a **Pipeline** job, link it to your Git repository, and ensure the Jenkinsfile is used to configure the pipeline.

### 14. **Run and Test the Jenkins Job**
- Run the pipeline job in Jenkins and verify that the entire CI/CD flow works, from building and testing the app to containerizing it and deploying it to Kubernetes.

---

## Dockerfile

```dockerfile
FROM tomcat:8-jre11

# Remove any default web apps
RUN rm -rf /usr/local/tomcat/webapps/*

# Copy the WAR file to the Tomcat webapps directory
COPY target/vprofile-v2.war /usr/local/tomcat/webapps/ROOT.war

# Expose port 8080
EXPOSE 8080

# Command to run the Tomcat server
CMD ["catalina.sh", "run"]
```

This `Dockerfile` builds an image based on **Tomcat 8** with JDK 11. It copies the WAR file into the Tomcat webapp directory and exposes port `8080`.

---

## Jenkinsfile (Pipeline Code)

This is the declarative Jenkins pipeline file that defines the steps for building, testing, containerizing, and deploying the application.

```groovy
pipeline {
    agent any

    environment {
        registry = "imranvisualpath/vproappdock"
        registryCredential = 'dockerhub'
    }

    stages {
        
        // Stage 1: Build the Application
        stage('BUILD') {
            steps {
                sh 'mvn clean install -DskipTests'
            }
            post {
                success {
                    echo 'Archiving artifacts...'
                    archiveArtifacts artifacts: '**/target/*.war'
                }
            }
        }

        // Stage 2: Run Unit Tests
        stage('UNIT TEST') {
            steps {
                sh 'mvn test'
            }
        }

        // Stage 3: Run Integration Tests
        stage('INTEGRATION TEST') {
            steps {
                sh 'mvn verify -DskipUnitTests'
            }
        }

        // Stage 4: Code Analysis with Checkstyle
        stage ('CODE ANALYSIS WITH CHECKSTYLE') {
            steps {
                sh 'mvn checkstyle:checkstyle'
            }
            post {
                success {
                    echo 'Generated Checkstyle Analysis Result'
                }
            }
        }

        // Stage 5: Build Docker Image
        stage('Building image') {
            steps {
                script {
                    dockerImage = docker.build registry + ":$BUILD_NUMBER"
                }
            }
        }
        
        // Stage 6: Deploy Docker Image to DockerHub
        stage('Deploy Image') {
            steps {
                script {
                    docker.withRegistry( '', registryCredential ) {
                        dockerImage.push("$BUILD_NUMBER")
                        dockerImage.push('latest')
                    }
                }
            }
        }

        // Stage 7: Clean up Unused Docker Image
        stage('Remove Unused docker image') {
            steps {
                sh "docker rmi $registry:$BUILD_NUMBER"
            }
        }

        // Stage 8: SonarQube Code Quality Analysis
        stage('CODE ANALYSIS with SONARQUBE') {
            environment {
                scannerHome = tool 'mysonarscanner4'
            }

            steps {
                withSonarQubeEnv('sonar-pro') {
                    sh '''${scannerHome}/bin/sonar-scanner -Dsonar.projectKey=vprofile \
                       -Dsonar.projectName=vprofile-repo \
                       -Dsonar.projectVersion=1.0 \
                       -Dsonar.sources=src/ \
                       -Dsonar.java.binaries=target/test-classes/com/visualpathit/account/controllerTest/ \
                       -Dsonar.junit.reportsPath=target/surefire-reports/ \
                       -Dsonar.jacoco.reportsPath=target/jacoco.exec \
                       -Dsonar.java.checkstyle.reportPaths=target/checkstyle-result.xml'''
                }

                timeout(time: 10, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        // Stage 9: Deploy to Kubernetes using Helm
        stage('Kubernetes Deploy') {
            agent { label 'KOPS' }
            steps {
                sh "helm upgrade --install --force vprofile-stack helm/vprofilecharts --set appimage=${registry}:${BUILD_NUMBER} --namespace prod"
            }
        }
    }
}
```

### Breakdown of the Jenkinsfile:
- **Build**: This stage compiles the application with Maven.
- **Unit Test**: Runs unit tests using Maven.
- **Integration Test**: Runs integration tests after unit tests.
- **Checkstyle**: Runs code analysis using Checkstyle.
- **Docker Image Build**: Builds a Docker image and tags it with the build number.
- **Push Docker Image**: Pushes the Docker image to DockerHub.
- **Remove Unused Docker Image**: Cleans up the local Docker image to save space.
- **SonarQube Analysis**: Performs static code analysis using SonarQube.
- **Kubernetes Deploy**: Deploys the application to Kubernetes using Helm.

---

## Conclusion

This CI/CD pipeline leverages Jenkins, Docker, Kubernetes (via Kops), and Helm to automate the entire software delivery process, from code commit to deployment. By following the steps outlined above, you can achieve continuous integration and continuous deployment for your applications using Docker containers in a Kubernetes environment.
