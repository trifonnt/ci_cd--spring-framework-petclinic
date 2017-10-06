# Getting started with Docker for Java applications & Setting up CI/CD pipeline

Docker is already became quite famous and more organizations are moving to Docker based application development and deployment.
Here is a quick guide on how to containerize an existing Java web application and setup an end to end deployment pipeline for it
using Jenkins.

I am using the very famous Spring based petstore application for this and it represents a good sample as most of the applications
follow similar architecture.

### Here are the steps

1) Build the Petstore application
2) Run Sonar quality check on this
3) Prepare the Docker image with the webapplication
4) Run the container and execute Integration tests
5) If the tests are successful push the image to docker hub account

All the code is available in this location

The Jenkins pipeline code which can be used for the above steps are 
```
node {
    stage 'checkout'
    git 'https://gitlab.com/RavisankarCts/hello-world.git' 
    
    stage 'build'
    sh 'mvn clean install'
    
    stage('Results - 1') {
         junit '**/target/surefire-reports/TEST-*.xml'
         archive 'target/*.jar'
        }
    
    stage 'bake image'
    docker.withRegistry('https://registry.hub.docker.com','docker-hub-credentials') {
        def image = docker.build("ravisankar/ravisankardevops:${env.BUILD_TAG}",'.')
        
        stage 'test image'
        image.withRun('-p 8888:8888') {springboot ->
        sh 'while ! httping -qc1 http://localhost:8888/info; do sleep 1; done'
        git 'https://github.com/RavisankarCts/petclinicacceptance.git'
        sh 'mvn clean verify'
        }
        
        stage('Results') {
         junit '**/target/surefire-reports/TEST-*.xml'
         archive 'target/*.jar'
        }
        
        stage 'push image'
        image.push()
    }
}
```

The initial steps just checksout the code and run the build. The interesting part starts with this step runs with in Docker context using docker hub credentials
```
step 3 'bake image'
docker.withRegistry('https://registry.hub.docker.com','docker-hub-credentials') 
```

This step builds the docker image. Docker build command takes the your docker hub repository name and the tag name as one argument and your build location is another argument.
```
def image = docker.build("dockerhub registry name":"tag name",'location of docker file'). 
def image = docker.build("ravisankar/ravisankardevops:${env.BUILD_TAG}",'.')
```
This uses the Dockerfile to build the docker image. The contents of the docker file

```
FROM tomcat:8

ADD target/*.war /usr/local/tomcat/webapps
```

Next step is to run the image and run tests on it.

```
stage 'test image'
        image.withRun('-p 8888:8888') { springboot ->
        sh 'while ! httping -qc1 http://localhost:8888/info; do sleep 1; done'
        git 'https://github.com/RavisankarCts/petclinicacceptance.git'
        sh 'mvn clean verify'
}
```		
withRun step helps you to run the docker image you just build and expose the port where this application 
can be exposed. I have another test code base which is built and executed which will run tests on the image
that is running.

Final step is pushing the image to Dockerhub registry or any internal registry setup in your organization.

```
stage('Results') {
         junit '**/target/surefire-reports/TEST-*.xml'
         archive 'target/*.jar'
        }
        
        stage 'push image'
        image.push()
```
