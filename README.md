# Udacity Project 4: 
Project 4: Refactor Udagram App into Microservices and Deploy

## Important:
Please refer to the pdf file: **project4_rubrics.pdf** in the parent directory for the project screenshots and meeting the rubrics.

## Prerequisites:
1. Install Docker in your local computer.
2. Register a Docker Hub account in https://hub.docker.com .
3. Ensure Docker service is started in your computer and login to Docker Hub in the Terminal.
4. Recommended to create an AWS IAM account with installation privileges (DO NOT use the AWS ROOT account ).
5. With the AWS IAM account, ensure the AWS_ACCESS_KEY_ID and AWS_SECRET_ACCESS_KEY values can be found in the file ~/.aws/credentials, 
   which is needed for the docker image jsleung1/udacity-restapi-feed to run properly when issue AWS request to getSignedUrl.
   Also export the AWS_ACCESS_KEY_ID and AWS_SECRET_ACCESS_KEY values in .bash_profile, which is needed by the terminal
   when execute Terraform commands to create the infrastructure and KubeOne to install Kubernetes cluster 
   and create worker nodes.
6. Ensure a new SSH key is generated and add it to the ssh-agent before running ```terraform plan``` .
   - ```ssh-keygen -t rsa -b 4096 -C "your_email@example.com"```
   - ```ssh-add ~/.ssh/id_rsa```

## Setup Instructions:

#### 1. Build the docker images and ensure the images are running properly in the local system before installing Kubernetes.

  - Export the following environmental variables in .bash_profile.  These environmental variables will be read into docker-compose.yaml
    when execute *docker-compose up*.
    ```
    export AWS_BUCKET=
    export AWS_PROFILE=
    export AWS_REGION=
    export JWT_SECRET=

    export POSTGRESS_DATABASE=
    export POSTGRESS_DB=
    export POSTGRESS_HOST=
    export POSTGRESS_PASSWORD=
    export POSTGRESS_USERNAME=
    export URL=
    ```
  - Build the individual docker images:
    ```
    - cd udacity-c3-restapi-user/
    - docker build -t jsleung1/udacity-restapi-user .

    - cd udacity-c3-restapi-feed/
    - docker build -t jsleung1/udacity-restapi-feed .

    - cd udacity-c3-frontend/
    - docker build -t jsleung1/udacity-frontend .

    - cd udacity-c3-deployment/docker
    - docker build -t jsleung1/reverseproxy .
    ```
  - Run the docker containers and verify the application behavior:
    ```
    - cd udacity-c3-deployment/docker
    - docker-compose up
    ```
    This will run the above four docker images as defined in **udacity-c3-deployment/docker/docker-compose.yaml** .
    From the terminal, ensure the four docker images are started without any errors.
    Verify the behavior of the Udagram by going to http://localhost:8100 in the Browser:
    
    - User is able to login to Udagram.
    - User is able to view the images of the feed.
    - User is able to upload images in Udagram.

#### 2. Instructions for installation of Kubernetes cluster:

  - Follow the installation steps in https://github.com/kubermatic/kubeone/blob/master/docs/quickstart-aws.md:
    - Created terraform.tfvars (refer to source: **terraform/aws/terraform.tfvars**)
      - Specify the correct AWS region ("us-east-1")
      - Specify the name of my Kubernetes cluster ("udacitykubeone")
      - ssh_public_key_file: point to the ssh key ("~/.ssh/id_rsa.pub")
    - execute: ```kubeone install config.yaml -t tf.json```
    - After the Kubernetes cluster is created, in .bash_profile, export KUBECONFIG=~/udacity_project4/terraform/aws/udacitykubeone-kubeconfig . 
      This is required so kubectl will automatically refer to this config when execute any commands related to kubectl. 
    - The source related to creation of Kubernetes cluster can be found in the source folder **terraform/aws** :
      - **terraform.tfvars**: Required by terraform to create the infrastructure.
      - **config.yaml**: Install Kubernetes and create the Kubernetes cluster.
      - **udacitykubeone-kubeconfig**: The full config including the sensitive data such as certificate authority data of my Kubernetes cluster ("udacitykubeone").
      - **udacitykubeone-kubeconfig-bare-travis**: The config of my Kubernetes cluster excluding the sensitive data, required by Travis CI to update the Kubernetes pods after    the images were successfully built by Travis CI.  The sensitive data in the config are replaced by Travis environment variables whose values are defined in Travis Settings.

#### 3. Instructions for creation of Kubernetes pods in Kubernetes cluster:

  - Ensure the Kubernetes cluster are running at least 3 replicas:
    - Execute ```kubectl scale -n kube-system machinedeployment/udacitykubeone-pool1 --replicas=3```
  - Refer to the folder: **udacity-c3-deployment/k8s**.
  - Apply the following files using ```kubectl apply -f <file_name.yaml>```  for the configuration of the Kubernetes cluster:
    - **aws-secret.yaml**: Contains the credentials to access AWS.  In the file, under credentials, fill in the value by convert the .aws/credentials file to base64 string.
    - **env-secret.yaml**: Contains authenticaion details to the DB:  The Db login user and password are encoded in base64 string. 
    - **env-configmap.yaml**: Contains non-senstivie configuration values, such as AWS_BUCKET, AWS_REGION, and DB information such as POSTGRESS_DB, POSTGRESS_HOST.
  - Apply the following files using ```kubectl apply -f <file_name.yaml>``` for the creation of the Kubernetes pods:
    - **backend-feed-deployment.yaml** - for the deployment of docker image jsleung1/udacity-restapi-feed in the Kubernetes pods.
    - **backend-user-deployment.yaml** - for the deployment of docker image jsleung1/udacity-restapi-user in the Kubernetes pods.
    - **reverseproxy-deployment.yaml** - for the deployment of docker image jsleung1/reverseproxy in the Kubernetes pods.
    - **frontend-deployment.yaml** - for the deployment of docker image jsleung1/udacity-frontend in the Kubernetes pods.
  - Apply the following files using ```kubectl apply -f <file_name.yaml>``` for the creation of services in the Kubernetes cluster:
    - **backend-user-service.yaml** - for the creation of backend-user service in the Kubernetes cluster.
    - **backend-feed-service.yaml** - for the creation of backend-feed service in the Kubernetes cluster.
    - **reverseproxy-service.yaml** - for the creation of reverseproxy service in the Kubernetes cluster.
    - **frontend-service.yaml** - for the creation of frontend service in the Kubernetes cluster.
  - Execute ```kubectl get pods``` and ensure all the pods have **STATUS = running**.
  - At this point, we want to verify the behavior of the Udagram application after succcessfully created the pods in the Kubernetes cluster:
    - Execute ```kubectl port-forward service/reverseproxy 8080:8080```
    - Execute ```kubectl port-forward service/frontend 8100:8100```
    - Verify the behavior of the Udagram by going to http://localhost:8100 in the Browser:
      - User is able to login to Udagram.
      - User is able to view the images of the feed.
      - User is able to upload images in Udagram.

#### 4. Instructions for integrating Travis CI with Github repository

  - Go to https://github.com/jsleung1/udacity_project4 , go to Settings, and under Apps and integrations, select Travis CI and grant Travis CI access to the repository jsleung1/udacity_project4.
  - Go to https://travis-ci.com.  Since you are already logged in to GitHub, Travis CI will display a list of repository from Github including udacity_project4.  Ensure udacity_project4 is activated in Travis CI.
  - In Travis CI, select the repository jsleung1/udacity_project4. Go to more options, settings.
    - In the settings, under Environment Variables, create the following:
    ```
      - DOCKER_USERNAME - Docker hub login username.
      - DOCKER_PASSWORD - Docker hub password.
      - KUBE_CA_CERT - copy the certificate-authority-data defined in the file in the file terraform/aws/udacitykubeone-kubeconfig.
      - KUBE_ENDPOINT - copy the server URL path defined in the file in the file terraform/aws/udacitykubeone-kubeconfig.
      - KUBE_ADMIN_CERT - copy the client-certificate-data defined in the file terraform/aws/udacitykubeone-kubeconfig.
      - KUBE_ADMIN_KEY - copy the client-key-data defined in the file terraform/aws/udacitykubeone-kubeconfig.
    ``` 
  - Create a "bare" kubeconfig file that strip out the above sensitive information related to udacitykubeone-kubeconfig.
    - Under the parent source directory, this file is placed in **terraform/aws/udacitykubeone-kubeconfig-bare-travis**.
  - In the parent source folder, create **.travis.yml** file with the following instructions:
    - Before the build process start, it install docker-compose and kubectl.  Also, it will start Docker and login to my Docker hub account.
    - For the build process, it run docker-compose with the latest source code changes.
    - After the docker images were successfully built, it will push the images to my Docker hub, by TRAVIS_BUILD_ID, and also tag/push the Docker images as latest.
    - From the environmental variables in Travis settings, it will fill the stripped udacitykubeone-kubeconfig-bare-travis such that Travis can access our Kubernetes cluster.
    - Travis CI will update the four images in the Kubernetes pod using rolling-update by running ```kubectl set image```. (For details, please refer to my .travis.yml).