# Docker Jenkins Nuke Issue

I'm doing this on a DigitalOcean droplet Docker template from their marketplace.

I'm following these instructions. https://www.jenkins.io/doc/book/installing/docker

Follow these steps on the machine that has docker installed.

Create the network.

```
sudo docker network create jenkins
```

Run the docker-in-docker image. This allows execution of docker commands inside jenkins.

```
sudo docker run --name jenkins-docker --detach \
  --privileged --network jenkins --network-alias docker \
  --env DOCKER_TLS_CERTDIR=/certs \
  --volume jenkins-docker-certs:/certs/client \
  --volume jenkins-data:/var/jenkins_home \
  --publish 2376:2376 \
  docker:dind --storage-driver overlay2
```

Create a Dockerfile that adds to jenkins. This includes dependencies for .NET on Debian 11 (bullseye). The second apt-get line is the .NET dependencies.

```
FROM jenkins/jenkins:2.332.1-jdk11
USER root
RUN apt-get update && apt-get install -y lsb-release \
  libc6 libgcc1 libgssapi-krb5-2 libicu66 libssl1.1 libstdc++6 zlib1g
RUN curl -fsSLo /usr/share/keyrings/docker-archive-keyring.asc \
  https://download.docker.com/linux/debian/gpg
RUN echo "deb [arch=$(dpkg --print-architecture) \
  signed-by=/usr/share/keyrings/docker-archive-keyring.asc] \
  https://download.docker.com/linux/debian \
  $(lsb_release -cs) stable" > /etc/apt/sources.list.d/docker.list
RUN apt-get update && apt-get install -y docker-ce-cli
USER jenkins
RUN jenkins-plugin-cli --plugins "blueocean:1.25.3 docker-workflow:1.28"
```

Build the docker image:

```
sudo docker build -t custom-jenkins-blueocean:2.339-1 .
```

Run the docker blue ocean container.

```
sudo docker run --name jenkins-blueocean --detach \
  --network jenkins --env DOCKER_HOST=tcp://docker:2376 \
  --env DOCKER_CERT_PATH=/certs/client --env DOCKER_TLS_VERIFY=1 \
  --publish 8080:8080 --publish 50000:50000 \
  --volume jenkins-data:/var/jenkins_home \
  --volume jenkins-docker-certs:/certs/client:ro \
  custom-jenkins-blueocean:2.339-1
```

Open a browser to `<ipaddress>:8080`. Finish the Jenkins install. Use the recommended plugins selection.


Click the `Open Blue Ocean` link on the left. Create a new pipeline. Select GitHub. Choose your organization and then repository. This repository should have a Nuke build project with execution engine 6.0.1. Complete the pipeline creation.

Click on a node to configure it. Choose `Shell Script`. Add this script.

```
./build.sh compile --configuration Release
```

Save the pipeline and run it if it didn't automatically start.
