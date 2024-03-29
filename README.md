# Gateway Deployment Project

### Project Overview

The project consists of several microservices, the overall objective of the project is to successfully build, deploy and  automate the project onto a cloud platform. 

#### Schema of microservices and their dependencies
![alt text](https://github.com/DarylMcBride/img/blob/master/ciprojectdiagr.png)

The most significant challenge of this task will be ensuring image creation of each of the microservices adheres to the project schema.   

Prerequisites
* clone in project -git clone https://github.com/DarylMcBride/CloudGatewayDeployment.git

Creating Kubernetes Cluster

```
gcloud container clusters create [CLUSTER-NAME] --zone [COMPUTE-ZONE]
gcloud container clusters get-credentials [CLUSTER-NAME] --zone [COMPUTE-ZONE]
```

Required ports
* Clients - 8080
* Services - 3000
* nginx - 80
* Jenkins - 8080
                                                             
#### Project Walkthrough
![alt text](https://github.com/DarylMcBride/img/blob/master/project-steps2.png)


Components
* 1 gateway API
* 2 client API's
* 10 service API's
* Mongo database

##### 1. Implementing project with Docker
  
  For the initial implementation for the project I created Dockerfiles for each of the microservices in order to
  test that they were working correctly.
  
```
FROM node:10
COPY . .
RUN npm install
ENTRYPOINT ["/usr/local/bin/npm", "run", "serve"]

                                                                            Dockerfile for a service
```

##### 2. Uploading Docker images

  In order to test that my deployment conformed to the schema I created an inital docker-compose.yaml file directed towards 
  the Dockerfiles I created in step one. 
  
```
version: '3.3'
services:
        secret-service:
                build: ./secret-service
                image: ${DOCKER_LOGIN}/secret-service:v${BUILD_NUMBER}
                container_name: secret-service
                depends_on:
                - mongo-service
                
                                                              Snippet example of docker-compose.yaml
```
  After ensuring all of the micro-services were working as intended, I changed the image tag of each microservice to point
  towards its own dockerhub repo and then pushed the images. This allowed me to fully test that the applications worked
  correctly and allowed me to easily convert the deployment into Kubernetes clusters.
  
  the ${DOCKER_LOGIN} environment variable must be changed to your docker-hub account within a .env file in order to allow
  you to login to your account.
  
  ```
  cd CloudGatewayDeploy/ci-project-dist/
  docker login
  docker-compose build -t ./
  docker-compose push
  ```
  The above command will trigger a push of the images to the appropriate docker-hub account, this command requires you to
  input your docker-hub account details in order to gain permission to push the images.
  
##### 3. Implementing Kubernetes
  
  Each of the microservices require a deployment.yaml in order to create kubernetes deployments and services. Each of
  the microservices are deployed as a microservice, with the gateway service deployed as a LoadBalancer in order to allow
  the user to access a public domain.
  
```
 spec:
      containers:
      - name: account-service
        image: docker.io/${DOCKER_LOGIN}/account-service
        ports:
        - containerPort: 3000    
        
                                                              Snippet example of account-service.yaml
```
Once all environment variables have been checked to be correct simply navigate to the kubectl-deploy folder and apply the changes.

```
cd ~/CloudGatewayProject/ci-project-dist/kubectl-deploy/
kubectl apply -f ./
```

The activation link environment variable must be changed to the ip address given to your gateway load balancer. This must be changed and reapplied within the authenctication-service-deploy.yaml. See below for the steps to achieve this.

```
 cd ~/CloudGatewayProject/ci-project-dist/kubectl-deploy/
 vim authentication-service-deploy.yaml
```

```
 env:
        - name: ACTIVATION_LINK      
          value: http://35.246.7.186/authentication/api/activate 
```
Once this step has been achieved simply reapply the changes.

```
kubectl apply -f ./authentication-service-deploy.yaml
```
The project has now been deployed successfully within a Kubernetes cluster.
	
##### 4. Create/Deploy Jenkins

  Jenkins can be deployed on the kubernetes cluster to give it access to the project. The major difficulty in this task is 
  giving jenkins the correct permissions to access and run the project.
  This step requires you to find which vm instance jenkins is running on and ssh into it. It is neccessary to ensure that 
  both docker and docker-compose are installed, you then have to log into docker-hub on the instance to allow jenkins 
  permission to download the images and also create a new user group for jenkins.

```
          spec:
                serviceAccountName: jenkins-service-account
                securityContext:
                        fsGroup: 1000
                containers:
                - name: jenkins
                  securityContext:
                          runAsUser: 1000
                  image: docker.io/${DOCKER_LOGIN}/jenkins
                  
                                                         Creating and deploying jenkins instance      
```

  
##### 5. Add each microservice into Jenkins

Each of the microservices can then be individually built using the command below on jenkins.

```
export DOCKER_LOGIN=[docker account]
cd ci-project-dist
docker-compose build
docker-compose push
kubectl --record deployment.apps/authentication-client set image deployment.v1.apps/
authentication-client authentication-client=docker.io/${DOCKER_LOGIN}/
authentication-client:v${BUILD_NUMBER}

                                                              Building an image via jenkins
```

##### 6. Create web-hooks for each microservice

Implementing web hooks on jenkins allow the jenkins job to scan git hub for any updates which occur in the projects repository. If an update occurs jenkins will scan through each microservice and update the build appropriately.


![alt text](https://github.com/DarylMcBride/img/blob/master/CI.png)

