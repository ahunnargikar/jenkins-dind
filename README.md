jenkins-dind
============

# Jenkins Slave Docker-in-Docker Image

This Jenkins slave image needs to be run in privileged mode and is then capable of launching docker containers within itself. It is an extension of https://github.com/jpetazzo/dind/blob/master/Dockerfile

```bash
.
├── Dockerfile (Docker file used to build the image)
├── README.md 
├── supervisord.conf (Supervisor startup configuration)
└── wrapdocker (Script used to launch Docker)
```

## How it works

The assumption here is that you already have your jenkins master instance up and running with the Mesos plugin installed. Lets assume that the Jenkins master server is running at http://192.168.56.101:9000

### Step 1

The Jenkins Mesos plugin will launch this slave image in privileged mode via the Mesos Docker executor. This is a special slave image with docker installed within it (aka docker-in-docker/dind). The actual command that the Jenkins plugin sends to the Mesos executor will look something like this.

```bash
docker run -cidfile /tmp/docker_cid.0f1b144d2f235393 -privileged -c 512 -m 302365697638 -e JENKINS_COMMAND=wget -O slave.jar http://192.168.56.101:9000/jnlpJars/slave.jar && java -DHUDSON_HOME=jenkins -server -Xmx640m -Xms16m -XX:+UseConcMarkSweepGC -Djava.net.preferIPv4Stack=true -jar slave.jar -jnlpUrl http://192.168.56.101:9000/computer/mesos-jenkins-408db551-2287-4b53-b49d-9136fb76af8a/slave-agent.jnlp hashish/jenkins-dind
```

### Step 2

 - Once the Jenkins slave has connected to the master, it will run the following command (as specified in the Jenkins job configuration) to launch a Docker-in-Docker container. This new container can be a generic JDK7+Maven+Tomcat or Nginx+Node.js Ubuntu/Redhat image for applications. This docker run operation will run in non-privileged mode.

```bash
docker build -t <username>/jenkins-jdk7-maven-tomcat - < application_dockerfile
```

 - The dockerfile from the application Git repo will contain all the commands necessary to build the application code, download protected packages, rpms, etc. Once the build has completed and we exit this docker-in-docker image.

### Step 3

The docker image built and pushed into the registry in step 2 is the final deliverable and can be deployed in any Mesos-Docker cluster.

## How to build & run this image

Important: You will need to download the latest JDK7 & Maven files, rename them to jdk.tar.gz and maven.tar.gz respectively and place them next to the Docker file before building.

```bash
docker build -t <username>/jenkins-dind .
```

```bash
docker push <username>/jenkins-dind
```

```bash
docker run -privileged -t -i <username>/jenkins-dind
```