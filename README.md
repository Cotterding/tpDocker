# Rapport TP Docker
## 1-1 Document your database container essentials : commands and Dockerfile
Creation of Dockerfile :

```
FROM postgres:14.1-alpine

ENV POSTGRES_DB=db \
   POSTGRES_USER=usr \
   POSTGRES_PASSWORD=pwd
```
Creation of network : 
```
docker network create app-network
```
Start Adminer :
```
    docker run \
    -p "8090:8080" \
    --net=app-network \
    --name=adminer \
    -d \
    adminer
```
Run adminer & la database : 
```
docker run -p 8888:5000 --name mydatabase —network app-network juliedlg/mydatabase
```
Creation of 2 sql files, in initdb folder: 01-CreateScheme.sql and 02-InsertData.sql  
Add this in Dockerfile :
```
COPY /initdb /docker-entrypoint-initdb.d
```
Rebuild the image : 
```
docker build -t juliedlg/mydatabase .
```
Re-run : 
```
docker run -p 8888:5000 --name mydatabase juliedlg/mydatabase
```
Data persistence : 
```
docker run -p 8888:5000 --name mydatabase -v /my/own/datadir:/var/lib/postgresql/data
juliedlg/mydatabase
```
## 1-2 Why do we need a multistage build? And explain each step of this dockerfile.
Multistage builds are used to optimize Docker images by separating the build environment from the final production image. It reduces the size of the final image and eliminates unnecessary build dependencies.
```FROM maven:3.8.6-amazoncorretto-17 AS myapp-build```
sets the base image to Maven with Amazon Corretto 17. AS creates a name build stage to be referenced to. 

```ENV MYAPP_HOME /opt/myapp```
defines an environment variable MYAPP_HOME, which specifies the application's home directory inside the container.

```WORKDIR $MYAPP_HOME```
sets the working directory in the container to $MYAPP_HOME, which will be “/opt/myapp."

```COPY pom.xml . and COPY src ./src```
copies the project's pom.xml and the src directory into the container.

```RUN mvn package -DskipTests```
runs Maven to build the application. The -DskipTests flag skips the execution of tests. After this command, the application is compiled, and the resulting JAR file is created.

```FROM amazoncorretto:17```
sets the base image to Amazon Corretto 17

```ENV MYAPP_HOME /opt/myapp```
defines the MYAPP_HOME environment variable

```WORKDIR $MYAPP_HOME```
sets the working directory to the application's home directory.

```COPY --from=myapp-build $MYAPP_HOME/target/*.jar $MYAPP_HOME/myapp.jar ```
copies the JAR file generated in the previous build stage from the "myapp-build" stage to the final image's /opt/myapp directory as "myapp.jar." It only includes the necessary artifacts in the final image while excluding build dependencies.

```ENTRYPOINT java -jar myapp.jar ```
specifies the command to execute when the container is started. Here, it runs the Java application by executing the JAR file with the java -jar command.

## 1-3 Document your docker-compose file.
Creation of a docker-compose.yml : 
```
version: '3.7'

services:
    backend:
        build:
          context: ./simple-api-student-main
          dockerfile: Dockerfile
        container_name: mybackend
        networks:
        - app-network
        depends_on:
        - database

    database:
        build:
          context : .
          dockerfile : Dockerfile
        container_name: mydatabase
        networks:
        - app-network

    httpd:
        build:
          context : ./httpserver
          dockerfile : Dockerfile
        container_name: myfrontend
        ports:
        - "8080:80"
        networks:
        - app-network
        depends_on:
        - backend

networks:
    app-network:
```
## 1-4 Document docker-compose most important commands.
```docker compose up```

## 1-5 Document your publication commands and published images in dockerhub.
Login to DockerHub account : 
```docker login```

Tag the image :
```docker tag tp1-httpd juliedlg/tp1-httpd:1.0```

Push the image to dockerhub :
```docker push juliedlg/tp1-httpd:1.0```

You can view the image on the Docker Hub account:
![image](https://github.com/Cotterding/tpDocker/assets/107808913/5451c670-93ac-4cce-8a89-6fe4b9b9717f)

## 2-1 What are testcontainers?
They are java libraries that allow you to run a bunch of docker containers while testing.

## 2-2 Document your Github Actions configurations.
Creation of GitHub repository
```
git init 
git remote add origin https://github.com/Cotterding/tpDocker.git
git push --set-upstream origin main
```
Creation of folder et file : ```.github/workflows et du fichier main.yml```
```
name: CI devops 2023
on:
  #to begin you want to launch this job in main and develop
  push:
    branches: 
      - main
      - tp1
  pull_request:

jobs:
  test-backend: 
    runs-on: ubuntu-22.04
    steps:
     #checkout your github code using actions/checkout@v2.5.0
      - uses: actions/checkout@v2.5.0

     #do the same with another action (actions/setup-java@v3) that enable to setup jdk 17
      - name: Set up JDK 17
        uses: actions/setup-java@v3
        with:
          java-version: '17'
          distribution: 'adopt'
          cache: maven

     #finally build your app with the latest command
      - name: Build and test with Maven
        working-directory : simple-api-student-main
        run: |
          mvn -B verify sonar:sonar -Dsonar.projectKey=Cotterding_tpDocker -Dsonar.organization=cotterding -Dsonar.host.url=https://sonarcloud.io -Dsonar.token=${{ secrets.SONAR_TOKEN }}

    # define job to build and publish docker image
  build-and-push-docker-image:
   needs: test-backend
   # run only when code is compiling and tests are passing
   runs-on: ubuntu-22.04

   # steps to perform in job
   steps:

     - name: Login to DockerHub
       run: docker login -u ${{ secrets.DOCKERHUB_USERNAME }} -p ${{ secrets.DOCKERHUB_TOKEN }}
       
     - name: Checkout code
       uses: actions/checkout@v2.5.0

     - name: Build image and push backend
       uses: docker/build-push-action@v3
       with:
         # relative path to the place where source code with Dockerfile is located
         context: ./simple-api-student-main
         # Note: tags has to be all lower-case
         tags:  ${{secrets.DOCKERHUB_USERNAME}}/tp-devops-simple-api-student-main:latest
         # build on feature branches, push only on main branch
         push: ${{ github.ref == 'refs/heads/main' }}

     - name: Build image and push database
       uses: docker/build-push-action@v3
       with:
          context: .
          tags: ${{ secrets.DOCKERHUB_USERNAME }}/tp-devops-database:latest
          push: ${{ github.ref == 'refs/heads/main' }}

     - name: Build image and push httpd
       uses: docker/build-push-action@v3
       with:
        context: ./httpserver
        tags: ${{ secrets.DOCKERHUB_USERNAME }}/tp-devops-httpd:latest
        push: ${{ github.ref == 'refs/heads/main' }}

    - name : Build image and push frontend
      uses: docker/build-push-action@v3
      with:
        context: ./frontend
        tags: ${{ secrets.DOCKERHUB_USERNAME }}/tp-devops-frontend:latest
        push: ${{ github.ref == 'refs/heads/main'}}
```
The image has been successfully created on DockerHub in the repositories : 
![image](https://github.com/Cotterding/tpDocker/assets/107808913/8c86f8cd-7591-4bf0-81d2-f8704def8908)

## 2-3 Document your quality gate configuration.

Creation of a SonarCloud account, by taking a free trial and creating an organization.  

Retrieving a project key and an organization key, then creating a secret in GitHub : 

![image](https://github.com/Cotterding/tpDocker/assets/107808913/cbc72db6-c4d6-4455-be69-883297b2949b)

Adding 
```
- name: Build and test with Maven
        working-directory : simple-api-student-main
        run: |
          mvn -B verify sonar:sonar -Dsonar.projectKey=Cotterding_tpDocker -Dsonar.organization=cotterding -Dsonar.host.url=https://sonarcloud.io -Dsonar.token=${{ secrets.SONAR_TOKEN }}
```

![image](https://github.com/Cotterding/tpDocker/assets/107808913/a685b78a-751f-4eff-a1cb-2138895bd2a4)

## 3-1 Document your inventory and base commands

Creation of an /ansible/inventories/setup.yml
```
all:
  vars:
    ansible_user: centos
    ansible_ssh_private_key_file: ../id_rsa
  children:
    prod:
      hosts: julie.doligezlelourec.takima.cloud
```
Test inventory with the ping command, we get :  

![image](https://github.com/Cotterding/tpDocker/assets/107808913/5ffe7cb8-dd7d-4513-9be1-89fd6352c11f)  

Request your server to get your OS distribution, thanks to the setup module : 
```ansible all -i inventories/setup.yml -m setup -a "filter=ansible_distribution*"```  

Remove the installed Apache httpd server on the machine :
```ansible all -i inventories/setup.yml -m yum -a "name=httpd state=absent" --become```

## 3-2 Document your playbook
Creation of a playbook :
```
- hosts: all

  gather_facts: false

  become: true

  tasks:

   - name: Test connection

     ping:
```
Exécution of the playbook :  

![image](https://github.com/Cotterding/tpDocker/assets/107808913/cdb96206-0362-4739-9531-50f04886d518)

Addition in the playbook to install docker on the server:
```
tasks:
  - name: Clean packages
    command:
      cmd: yum clean -y packages

  - name: Install device-mapper-persistent-data
    yum:
      name: device-mapper-persistent-data
      state: latest

  - name: Install lvm2
    yum:
      name: lvm2
      state: latest

  - name: add repo docker
    command:
      cmd: sudo yum-config-manager --add-repo=https://download.docker.com/linux/centos/docker-ce.repo

  - name: Install Docker
    yum:
      name: docker-ce
      state: present

  - name: Make sure Docker is running
    service: name=docker state=started
    tags: docker
```
Creation of folder for each roles : ```ansible-galaxy init roles/docker```

## 3-3 Document your docker_container tasks configuration.
Addition of tasks for each role : 
```
- name: Lauch app
  docker_container:
    name: mybackend
    image: juliedlg/tp-devops-simple-api-student-main:latest
    networks: 
      - name: app-network

- name: Create a network
  community.docker.docker_network:
    name: app-network

- name: Run HTTPD
  docker_container:
    name: myfrontend
    image: juliedlg/tp-devops-httpd:latest
    networks: 
      - name: app-network
    ports: 
      - 8080:80

- name: Launch database
  docker_container:
    name: mydatabase
    image: juliedlg/tp-devops-database:latest
    networks:
      - name: app-network
    env:
      POSTGRES_DB: "db"
      POSTGRES_USER: "usr"
      POSTGRES_PASSWORD: "pwd"
```
And addition of roles in playbook :
```
- hosts: all
  gather_facts: false
  become: true

  roles:
    - docker
    - network
    - database
    - app
    - proxy
 ```
Following command in ansible folder :  ```ansible-playbook -i inventories/setup.yml playbook.yml ```




