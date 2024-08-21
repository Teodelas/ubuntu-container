## Jenkins Pipeline Structure

The Jenkinsfile represents a declarative pipeline, which defines a continuous integration/continuous deployment (CI/CD) pipeline in Jenkins. Below are the environment variables that are defined for this Jenkins job.

```
environment {
   REPOSITORY = 'registry-server:5000'
   PCC_CONSOLE_URL = "10.160.154.170:8083"
   CONTAINER_NAME = "ubuntu"
}
```
1. `REPOSITORY`: This is the locally hosted Docker repository that is setup.
2. `PCC_CONSOLE_URL`: The URL of Prisma Cloud Compute Console
3. `CONTAINER_NAME`: Name of the Container


## Stages

### Build Stage

```
stage('Build') {
   steps {
      withCredentials([usernamePassword(credentialsId: 'docker_registry_creds', passwordVariable: 'REGISTRY_PASS', usernameVariable: 'REGISTRY_USER')]) {                
         sh ''' 
         docker login -u $REGISTRY_USER -p $REGISTRY_PASS $REPOSITORY
         echo "Building the Docker image..."
         docker build -t $REPOSITORY/$CONTAINER_NAME:$BUILD_NUMBER .
         docker image ls
         '''
       }
   }
}
```

The Build stage is responsible for building the Docker image. It consists of the following steps:

1. Login to the Docker registry
2. Build the Docker image: This step builds a Docker image using the Docker CLI. The image is tagged with the provided `IMAGE_NAME` and `DOCKER_TAG` environment variables.

### Image Scaning Stage

```
stage('Container Scan') {
   steps {
      prismaCloudScanImage ca: '', cert: '', dockerAddress: 'unix:///var/run/docker.sock', ignoreImageBuildTime: true, image: "$REPOSITORY/$CONTAINER_NAME:$BUILD_NUMBER", key: '', logLevel: 'debug', podmanPath: '', project: '', resultsFile: 'prisma-cloud-scan-results.json'
      prismaCloudPublish resultsFilePattern: 'prisma-cloud-scan-results.json'
   }
}        
```
The Container Scan stage is responsible for scanning the Docker image. Below are things to note:
1. It uses Jenkins Prisma Cloud plugin to perform the scan
2. Once the scan is done, it publishes the results in Jenkins Console output and also ships the results to Prisma Cloud Compute Console.
3. This stage will pass/fail depending on the Failure threshold set in Prisma Cloud. For example, if the fail threshold is set to Medium in Prisma CLoud, then this build will fail if there are any Medium severity vulnerabilities found during the scan.

### Push Image Stage

```
stage('Push Image') {
   steps {
         sh ''' 
         echo "Image push into registry"
         docker push $REPOSITORY/$CONTAINER_NAME:$BUILD_NUMBER
         '''
   }
}
```         

1. In this stage, if the build passed in the previous stage, the Docker image is pushed to the specified repository

### Container Sandbox Scan Stage

```
stage('Container Sandbox Scan') {
   steps {
      withCredentials([usernamePassword(credentialsId: 'ssh_creds', passwordVariable: 'SSH_PASS', usernameVariable: 'SSH_USER')]) {
         sh '''
          mkdir -p ~/.ssh/
          ssh-keyscan -t rsa,dsa 10.160.154.170 >> ~/.ssh/known_hosts
          sshpass -p $SSH_PASS ssh $SSH_USER@10.160.154.170 'bash -s' <<EOF         
          sudo chmod +x /home/sysadmin/apps/sandbox-scan.sh
          sudo PCC_CONSOLE_URL=$PCC_CONSOLE_URL token=$token CONTAINER_NAME=$CONTAINER_NAME TAG=$BUILD_NUMBER /home/sysadmin/apps/sandbox-scan.sh
          exit
          EOF
         '''
      }
   }
} 
```

1. In this stage, the Sandbox scanning is performed on the Docker image using twistcli.
2. Docker image from the previous steps is pulled using `docker pull`
3. Then the `sandbox-scan.sh` is executed. Below are the contents of the `sandbox-scan.sh`
    ```
    #!/bin/bash

    REPOSITORY="registry-server:5000"
    docker login $REPOSITORY -u <username> -p <password>
    docker pull $REPOSITORY/$CONTAINER_NAME:$TAG
    twistcli sandbox --address https://$PCC_CONSOLE_URL --user <username> --password <password> --analysis-duration 1m $REPOSITORY/$CONTAINER_NAME:$TAG | sed 's/\x1b\[[0-9;]*m//g'
    ```
