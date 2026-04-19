## Step 11: Create the Jenkins Pipeline Job

  Before creating the job, make sure these repo changes are already in place:

  1. `Jenkinsfile`
     - Uses your GitHub repo URL
     - Uses your Docker Hub image
     - Uses your EKS API endpoint
     - Removes the email `post { ... }` block
     - Uses this Docker registry syntax:
     ```groovy
     withDockerRegistry(credentialsId: 'docker-cred', url: '')

  2. deployment-service.yaml
      - Uses the same Docker image pushed by Jenkins:

     image: <dockerhub-username>/boardgame:latest
  3. Dockerfile
      - Uses a current Java 17 runtime image:

     FROM eclipse-temurin:17-jre-alpine

     EXPOSE 8080

     ENV APP_HOME=/usr/src/app

     COPY target/*.jar $APP_HOME/app.jar

     WORKDIR $APP_HOME

     CMD ["java", "-jar", "app.jar"]
  4. pom.xml
      - distributionManagement points to your Nexus server

  ### Jenkins prerequisites

  Make sure Jenkins already has these tools configured in Manage Jenkins > Tools:

  - jdk17
  - maven3
  - sonar-scanner

  Make sure Jenkins already has these credentials in Manage Jenkins > Credentials > System > Global credentials:

  - git-cred
  - sonar-token
  - docker-cred
  - k8-cred

  Make sure Jenkins already has:

  1. A SonarQube server named sonar in Manage Jenkins > System
  2. A managed Maven settings file with ID global-settings

  ### Create the pipeline

  1. Open Jenkins.
  2. Click New Item.
  3. Enter BoardGame.
  4. Select Pipeline.
  5. Click OK.

  In the Pipeline section:

  1. Set Definition to Pipeline script from SCM
  2. Set SCM to Git
  3. Set Repository URL to your repo URL
  4. Select credential git-cred
  5. Set branch to */main
  6. Set Script Path to Jenkinsfile
  7. Click Save

  ———

  ## EKS Access Setup for Jenkins

  The Kubernetes deployment stage requires Jenkins to authenticate to EKS with AWS CLI.

  ### 1. Install AWS CLI on the Jenkins server

  Run on the Jenkins EC2 instance:

  sudo apt update
  sudo apt install -y unzip curl
  curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
  unzip -q awscliv2.zip
  sudo ./aws/install
  aws --version

  If needed:

  sudo ln -sf /usr/local/bin/aws /usr/bin/aws

  ### 2. Attach an IAM role to the Jenkins EC2 instance

  Attach an instance role to the Jenkins EC2 server.

  The role must be allowed to call:

  {
    "Version": "2012-10-17",
    "Statement": [
      {
        "Effect": "Allow",
        "Action": "eks:DescribeCluster",
        "Resource": "arn:aws:eks:<region>:<account-id>:cluster/<cluster-name>"
      }
    ]
  }

  ### 3. Grant that IAM role access to the EKS cluster

  In AWS Console:

  1. Open EKS
  2. Open your cluster
  3. Open the Access tab
  4. If needed, enable authentication mode API and ConfigMap
  5. Create or update an access entry for the Jenkins IAM role
  6. Associate AmazonEKSClusterAdminPolicy
  7. Set access scope to Cluster

  ### 4. Verify AWS access as the Jenkins user

  Run on the Jenkins server:

  sudo -u jenkins aws sts get-caller-identity

  ### 5. Generate kubeconfig for Jenkins

  sudo -u jenkins aws eks update-kubeconfig \
    --region <region> \
    --name <cluster-name> \
    --kubeconfig /tmp/devops-cluster-kubeconfig

  ### 6. Verify Kubernetes access

  sudo -u jenkins KUBECONFIG=/tmp/devops-cluster-kubeconfig kubectl get ns

  If the application namespace does not exist, create it:

  sudo -u jenkins KUBECONFIG=/tmp/devops-cluster-kubeconfig kubectl create namespace webapps

  ### 7. Upload kubeconfig to Jenkins

  Update Jenkins credential k8-cred with the kubeconfig file generated above.

  Use a Secret file credential.

  ———

  ## Run the Pipeline

  1. Open the BoardGame job
  2. Click Build Now
  3. Open Console Output

  Expected stage flow:

  1. Git Checkout
  2. Compile
  3. Test
  4. File System Scan
  5. SonarQube Analysis
  6. Quality Gate
  7. Build
  8. Publish To Nexus
  9. Build Docker Image
  10. Docker Image Scan
  11. Push Docker Image
  12. Deploy To Kubernetes
  13. Verify Deployment

  ———

  ## Common Fixes

  ### SonarQube says Not authorized

  Create a new token in SonarQube and update Jenkins credential sonar-token as Secret text.

  ### Jenkinsfile fails with Invalid parameter "toolName"

  Use:

  withDockerRegistry(credentialsId: 'docker-cred', url: '')

  Do not use toolName in withDockerRegistry.

  ### Docker build fails with openjdk:17-alpine: not found

  Use:

  FROM eclipse-temurin:17-jre-alpine

  ### Kubernetes deploy fails with exec: executable aws not found

  Install AWS CLI on the Jenkins server.

  ### Kubernetes deploy fails with Forbidden

  The Jenkins EC2 IAM role is missing EKS cluster access. Add an EKS access entry and associate AmazonEKSClusterAdminPolicy.

  ———

  ## Verify the Deployment

  Run:

  kubectl get pods -n webapps
  kubectl get svc -n webapps
  kubectl rollout status deployment/boardgame-deployment -n webapps

  Also confirm:

  1. The artifact is published to Nexus
  2. The Docker image is pushed to Docker Hub
  3. The Kubernetes service is created
  4. The application is reachable through the service endpoint

  ———

  ## Quick Re-Deploy Checklist

  Before rerunning later, verify:

  - AWS CLI is still installed on the Jenkins server
  - The Jenkins EC2 instance still has the correct IAM role
  - The EKS access entry still exists
  - k8-cred is still valid
  - sonar-token still works
  - The image in Jenkinsfile and deployment-service.yaml matches
  - pom.xml still points to the correct Nexus URL
