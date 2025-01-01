# Complete CI/CD Pipeline with EKS and Private DockerHub Registry

This project demonstrates a fully functional CI/CD pipeline leveraging Kubernetes, Jenkins, AWS EKS, DockerHub (private registry), Java, Maven, Linux, Docker, and Git. The pipeline automates the process of building, pushing, and deploying Java-based applications into an EKS cluster while ensuring version management and continuous updates.

## Project Overview

### Objectives
- Write Kubernetes manifest files for Deployment and Service configuration.
- Integrate deployment steps into the CI/CD pipeline.
- Automate application image building and pushing to a private DockerHub registry.
- Deploy new application versions to an EKS cluster dynamically.
- Commit version updates to a Git repository automatically.

### Pipeline Configuration
The pipeline consists of the following stages:

1. **CI Step: Increment Version**
2. **CI Step: Build Artifact for Java Maven Application**
3. **CI Step: Build and Push Docker Image to DockerHub**
4. **CD Step: Deploy New Application Version to EKS Cluster**
5. **CD Step: Commit the Version Update**

---

## Prerequisites

### Tools and Dependencies
1. **Jenkins**: CI/CD tool.
2. **AWS EKS**: Kubernetes cluster on AWS.
3. **Docker**: Containerization platform.
4. **Maven**: Build tool for Java applications.
5. **Linux**: Operating system environment.
6. **Git**: Version control system.

### Additional Setup
- Install the `gettext-base` tool in the Jenkins container for using envsubst for substituting environment variables placeholders inside configuration files:
  ```bash
  apt-get install gettext-base
  ```
- Create a DockerHub secret for pulling images:
  ```bash
  kubectl create secret docker-registry my-registry-key \
    --docker-server=docker.io \
    --docker-username=irschad \
    --docker-password=********
  ```
- Ensure `kubectl` and AWS CLI are configured to work with your EKS cluster.

---

## Jenkinsfile
The following `Jenkinsfile` defines the CI/CD pipeline:

```groovy
pipeline {
    agent any
    tools {
        maven 'Maven'
    }
    stages {
        stage('Increment Version') {
            steps {
                script {
                    echo "Incrementing app version"
                    sh 'mvn build-helper:parse-version versions:set \\n                       -DnewVersion=\${parsedVersion.majorVersion}.\${parsedVersion.minorVersion}.\${parsedVersion.nextIncrementalVersion} \\n                       versions:commit'
                    def matcher = readFile('pom.xml') =~ '<version>(.+)</version>'
                    def version = matcher[0][1]
                    env.IMAGE_NAME = "$version-$BUILD_NUMBER"
                }
            }
        }

        stage('Build JAR') {
            steps {
                script {
                    echo 'Building the application...'
                    sh 'mvn clean package'
                }
            }
        }

        stage('Build Image') {
            when {
                expression {
                    BRANCH_NAME == "master"
                }
            }
            steps {
                script {
                    echo "Building the Docker image..."
                    withCredentials([usernamePassword(credentialsId: 'docker-hub-repo', passwordVariable: 'PASS', usernameVariable: 'USER')]) {
                        sh "docker build -t irschad/java-app:${IMAGE_NAME} ."
                        sh 'echo $PASS | docker login -u $USER --password-stdin'
                        sh "docker push irschad/java-app:${IMAGE_NAME}"
                    }
                }
            }
        }

        stage('Deploy') {
            when {
                expression {
                    BRANCH_NAME == "master"
                }
            }
            environment {
                AWS_ACCESS_KEY_ID = credentials('jenkins_aws_access_key_id')
                AWS_SECRET_ACCESS_KEY = credentials('jenkins_aws_secret_access_key')
                APP_NAME = 'java-maven-app'
            }
            steps {
                script {
                    echo 'Deploying the application...'
                    sh 'envsubst < kubernetes/deployment.yaml | kubectl apply -f -'
                    sh 'envsubst < kubernetes/service.yaml | kubectl apply -f -'
                }
            }
        }

        stage('Commit Version Update') {
            when {
                expression {
                    BRANCH_NAME == "master"
                }
            }
            steps {
                script {
                    withCredentials([usernamePassword(credentialsId: 'jenkinspush', passwordVariable: 'PAT', usernameVariable: 'USER')]) {
                        sh 'git remote set-head origin master'
                        sh 'git config --global user.email "jenkins@example.com"'
                        sh 'git config --global user.name "jenkins"'
                        sh 'git add .'
                        sh "git remote set-url origin https://${PAT}@github.com/irschad/complete-ci-cd-eks-dockerhub.git"
                        sh "git commit -m 'ci: version bump'"
                        sh 'git push origin HEAD:master'
                    }
                }
            }
        }
    }
}
```

---

## Kubernetes Configuration

### `deployment.yaml`
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: $APP_NAME
  labels:
    app: $APP_NAME
spec:
  replicas: 1
  selector:
    matchLabels:
      app: $APP_NAME
  template:
    metadata:
      labels:
        app: $APP_NAME
    spec:
      imagePullSecrets:
        - name: my-registry-key
      containers:
        - name: $APP_NAME
          image: irschad/java-app:$IMAGE_NAME
          imagePullPolicy: Always
          ports:
            - containerPort: 8080
```

### `service.yaml`
```yaml
apiVersion: v1
kind: Service
metadata:
  name: $APP_NAME
spec:
  selector:
    app: $APP_NAME
  ports:
    - protocol: TCP
      port: 80
      targetPort: 8080
```

---

## Execution Workflow
1. **Run the Pipeline**:
   Trigger the Jenkins pipeline to start CI/CD automation.
2. **Build Stages**:
   - Increment the application version. The incremented version is used to dynamically generate the image name (e.g., `1.0.1-123`), reflecting the new version.
   - Build the JAR file using Maven.
   - Build and push the Docker image to DockerHub with the dynamically generated name.
3. **Deploy Application**:
   Deploy the new image to the EKS cluster using the dynamic image name.
4. **Verify Deployment**:
   - Check running pods:
     ```bash
     kubectl get pods
     ```
   - Verify the deployment:
     ```bash
     kubectl describe pod <pod-name>
     ```
   - Verify services:
     ```bash
     kubectl get service
     ```

---

## Notes
- **Dynamic Image Names**: Each pipeline run generates a unique image name by incrementing the application version, ensuring proper versioning and avoiding conflicts.
- **Secrets Management**: Ensure secrets are configured securely for DockerHub and AWS credentials.
- **Environment Variables**: Use `envsubst` to substitute dynamic values in Kubernetes YAML files.
- **Ignore Committer Setup**: The email `jenkins@example.com` is configured in Jenkins as part of the Ignore Committer setup to prevent the pipeline from auto-triggering itself.
- **Personal Access Token (PAT)**: Used in the Git commit stage to authenticate Jenkins with the GitHub repository securely. The PAT replaces the need for username/password and ensures secure, script-based interactions. To generate a PAT in GitHub:
  1. Navigate to **Settings** > **Developer Settings** > **Personal Access Tokens** > **Tokens (classic)**.
  2. Click on **Generate new token**.
  3. Select the necessary scopes such as `repo` for repository access.
  4. Use the generated token securely in Jenkins credentials management.

---

