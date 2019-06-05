# Gateway Deployment Project

### Project Overview

The project consists of several microservices, the overall objective of the project is to successfully build, deploy and  automate the project onto a cloud platform. 

#### Schema of microservices and their dependencies
![alt text](https://github.com/DarylMcBride/scripts/blob/master/ciprojectdiagr.png)

The most significant challenge of this task will be ensuring image creation of each of the microservices adheres to the project schema.   

Prerequisites
* clone in project -git clone https://github.com/DarylMcBride/CloudGatewayDeployment.git

Required ports
* Clients - 8080
* Services - 3000
* nginx - 80
* Jenkins - 8080
                                                             
#### Project Walkthrough
![alt text](https://github.com/DarylMcBride/scripts/blob/master/project-steps2.png)


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

  In order to test that my deployment conformed to the schema I created an inital docker-compose.yaml file directed towards     the Dockerfiles I created in step one. 
  
```
version: '3.3'
services:
        secret-service:
                build: ./secret-service
                image: darylmcbride/secret-service:v${BUILD_NUMBER}
                container_name: secret-service
                depends_on:
                - mongo-service
                
                                                              Snippet example of docker-compose.yaml
```
  After ensuring all of the micro-services were working as intended, I changed the image tag of each microservice to point
  towards its own dockerhub repo and then pushed the images. This allowed me to fully test that the applications worked
  correctly and allowed me to easily convert the deployment into Kubernetes clusters.
  
##### 3. Implementing Kubernetes
  
  Each of the microservices require a deployment.yaml in order to create kubernetes deployments and services. Each of
  the microservices are deployed as a microservice, with the gateway service deployed as a LoadBalancer in order to allow
  the user to access a public domain.
  
```
 spec:
      containers:
      - name: account-service
        image: docker.io/darylmcbride/account-service
        ports:
        - containerPort: 3000    
        
                                                              Snippet example of account-service.yaml
```
##### 4. Create/Deploy Jenkins


```
          spec:
                serviceAccountName: jenkins-service-account
                securityContext:
                        fsGroup: 1000
                containers:
                - name: jenkins
                  securityContext:
                          runAsUser: 1000
                  image: docker.io/darylmcbride/jenkins
                  
                                                         Creating and deploying jenkins instance      
```
##### 5. Add each microservice into Jenkins

```
kubectl --record deployment.apps/authentication-client set image deployment.v1.apps/
authentication-client authentication-client=docker.io/darylmcbride/
authentication-client:v${BUILD_NUMBER}

                                                              Building an image via jenkins
```

##### 6. Create web-hooks for each microservice


