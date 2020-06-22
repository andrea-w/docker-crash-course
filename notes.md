
## Docker Terminology

### Images
- read-only templates used to create containers
- created with `docker build`
- composed of layers of other images
- stored in an image registry, such as DockerHub

### Containers
- lightweight & portable encapsulations of an enviro in which to run applications
- created from images
- inside a container are all the binaries & dependencies needed to run the app
- if an image is a class, then a container is an instance of a class -- a runtime object

### Registries & Repositories
- images are stored in a registry
- can host your own registry or use Docker's public registry (DockerHub)
- inside a registry, images are stored in repositories
- Docker repo is collection of different Docker images w/ the same name, that have diff tags. Each tag usually represents a diff version of the image
- best practice is to use official images from DockerHub
- if you don't specify tag when pulling an image, Docker will use `latest` tag by default


## Run Hello World Container
- use `busybox` image (official) from DockerHub
- use `docker images` command to see what images you have stored locally
- `docker run` will create a container using the specified image, then spin up the container & run it
  - `docker run <repository>:<tag> <command> [<arguments>]
  - e.g., `docker run busybox:1.24 echo "hello world"`
  - Docker will first look to find the specified image locally. If it isn't found locally, Docker will go to DockerHub and search for the specified image there. If the image is found in DockerHub, Docker will pull it from the repo and then continue spinning up a container for it
  - once an image has been pulled from a non-local registry, running `docker images` again will show the pulled images in your local Docker registry. Execution is much faster when the image is local.
- `docker run <repository>:<tag> ls /` will list all contents of the image's container
- `-i` flag starts an interactive container
- `-t` flag creates pseudo-TTY that attaches `stdin` and `stdout`
  - e.g., `docker run -i -t busybox:1.24` will start up interactive container (effectively shelling yourself into the container)
  - once you `exit` an interactive container, Docker also shuts down the container. Running the same command again will spin up and run a new container, not the same one that was just exited.


## Deep Dive into Docker containers

### Foreground vs Detached

|  | Run Container in Foreground | Run Container in Background |
|-- |---|--------------|
Description | `Docker run` starts the process in the container & attaches the console to the process's standard input, output, and stderr. | Containers started in detached mode and exit when the root process used to run the container exits. |
| How to specify | (default) | `-d` |
| Can the console be used for other commands after the container is started | No | Yes |

- `docker ps` shows which containers are currently running in the local box
  - `-a` flag at end will list all containers that have previously run in the local box (including the containers that are now stopped)
- `docker run --name <container-name> <repository>:<tag>` will spin up and run a container with the specified container name using the image:tag given. Otherwise Docker generates random name for container of (adjective)_(noun)
- `docker inspect <container_id>` will output all details about the container


## Docker Port Mapping & Docker Logs Commands

- `docker run -it -p <host_port>:<docker_container_port> <repository>:<tag>` will cause the Docker container port to be exposed to the specified host_port on the Docker daemon (VM on local box)
  - e.g. `docker run -it -p 8888:8080 tomcat:8.0` will expose the container's port 8080 on the local machine's 8888 port. At the start of the Docker CLI, there will be an output message saying "docker is configured to use the default machine with IP xxx.xxx.xx.xxx". Going to IP:<host_port> in local browser will allow you access to what's running within the Docker container
  - If you are running Docker on Linux, or if you're running Docker for Mac or Docker for Windows, the host IP is just `localhost`
- `docker logs <container_id>` will output the logs for the given container 


## Docker Image Layers

- an image is built up of multiple image layers. Each image references the parent image (the image directly below it). The bottom image is the "base image", which sits on top of the kernel. 
- `docker history <repository>:<tag>` shows the full set of image layers that make up an image
- all changes made into the running containers will be written into the writable image layer
- when the container is deleted, the writable layer is also deleted, but the underlying image remains unchanged
- multiple containers can share access to the same underlying image


## Build Docker Images

There are 2 ways to build a Docker image: commit changes made in a Docker container; or write a Dockerfile

### Committing changes in a Docker container

1. Pull an image from DockerHub
   - `docker run -it debian:jessie`
   - `apt-get update && apt-get install -y git` --> installs git inside the running container

2. Commit the modified image to a Docker repository
    - `docker commit <container_id> <your_docker_id>/<repo_name>:<tag>
    - can now spin up new containers for this image from your local box by referencing <your_docker_id>/<repo_name>:<tag>


### Docker Commit

`docker commit <container_id> <repository>:<tag>` will save the changes made to the container's file system to a new image
    - to commit an image to your Docker repository, the repo name would be <your_docker_id>/<repo_name>. Need to set up a Docker account in order to save images to your own public repo


### Building Images by Writing a Dockerfile

- Dockerfile: a text document that contains all the instructions users provide to assemble an image. Each instruction will create a new image layer to the image. Instructions specify what to do when building the image.
- the file must be called "Dockerfile" exactly
- first instruction in the Dockerfile must be 
    `FROM <repo>:<tag>` to specify the base image
- the `RUN` instruction can be any command that can be executed in a Linux terminal
  -  e.g., `RUN apt-get update`
  -  when installing packages using `apt-get`, must include the `-y` flag to confirm installation, because there's no other way to answer the prompt. E.g., `RUN apt-get install -y git`
- to spin up a container from a Dockerfile, run `docker build -t <name> <path-to-build-context> <path-to-Dockerfile>`. The `-t` flag is used to tag the container with a name. If the location of the Dockerfile is in the same directory as where the build command is being executed, path-to-Dockerfile = "."
- the docker daemon runs each instruction in a Dockerfile inside an "intermediate" container. When the instruction has finished execution, the intermediate container will be removed (hence the "Removing intermediate container {{ container_id }}" output when the Dockerfile is building)
- once the `docker build` has run successfully, can use `docker images` to see the new built image in list of local images

#### Docker Build Context

Docker `build` command takes the path to the build context as an argument. This is necessary because the daemon could be running on a remote machine.
When build starts, docker client would pack all the files in the build context into a tarball, then transfer the tarball to the daemon.
By default, docker will search for the Dockerfile in the build context path (in the root directory).

## Dockerfile In Depth

### Chaining RUN Instructions

- Each RUN command will execute the command on the top writable layer of the container, then commit the container as a new image
- The new image is used for the next step in the Dockerfile. So each RUN instruction will create a new image layer
- Recommended to chain the RUN instructions in the Dockerfile to reduce the number of image layers created
    - Example: previously had
```
FROM debian:jessie
RUN apt-get update
RUN apt-get install -y git
RUN apt-get install -y vim
```
    
This can be refactored to

```
FROM debian:jessie
RUN apt-get update && apt-get install -y \
    git \
    vim
```

This reduces build steps from 4 to 2, so fewer intermediate containers are created.

- best practice is to sort multi-line arguments alphanumerically. This helps avoid duplication of packages and will make the list easier to update

### CMD Instructions

- CMD instructions specifies what command you want to run when the container starts up
- if we don't specify CMD instruction in the Dockerfile, Docker will use the default command defined in the base image (e.g, bash)
- CMD instruction doesn't run when building the image, only runs when the container starts up
- can specify the command in either `exec` form (preferred) or in shell form

### Docker Cache

- each time Docker executes an instruction it builds a new image layer. The next time docker executes Dockerfile, if the instruction hasn't changed, docker will reuse the existing layer
  - will see "Using cache" in output when re-running 

#### Pitfall: Aggressive Caching

Example:

Original Dockerfile:
```
FROM ubuntu:14.04
RUN apt-get update
RUN apt-get install -y git
```

then last line is modified to 
```
RUN apt-get install -y git curl
```

Docker will see that the first 2 lines of the Dockerfile have not changed, so it will reuse the cached image layers, and will only build the third line from scratch. But if some time has passed since the original Dockerfile was last built, in that time a newer version of apt-get may have been released, so that when a new image layer is created for the modified line, you're installing an older version of git and curl because apt-get has not been updated.

Solution 1 is to chain commands:
`RUN apt-get update && apt-get install -y git curl`

Solution 2 is to specify `no-cache`:
`docker build -t <docker_id>/<image_name> . --no-cache=true`

#### COPY Instruction

`COPY` instruction copies new files or directories from build context and adds them to the container's file system.

Example: in Dockerfile:
`COPY abc.txt /src/abc.txt` will copy the file "abc.txt" from the local build context over to the container's "/src/abc.txt" file location.

#### ADD Instruction

`ADD` instruction can copy files and also allows you to download a file from online and copy it to the container. It also has the ability to automatically unpack compressed files.

Best practice is to use COPY instruction for sake of transparency, unless ADD is absolutely needed.

## Pushing Images to DockerHub

Easiest way is to use "https://hub.docker.com" with your DockerHub ID.

1. `docker images` will list all images on your local box.
2. `docker tag <image_id> <repo_name>:<tag>` to name and tag the image as desired before being made public.
3. Push the image to DockerHub using `docker login --username=<dockerhub_id>` and enter DockerHub password when prompted. Then run `docker push <repo_name>:<tag>`

Best practice is to specify a tag rather than leaving it blank. If left blank, Docker will set the tag to `latest` by default. 

### `latest` tag

- Docker will use `latest` as default tag when no other tag is provided
- Many repos use `latest` to tag the most up-to-date stable image. However, this is only by convention and is not enforced.
- Images that are tagged `latest` won't be updated automatically when a newer version of the image is pushed to the repo.
- Avoid using `latest` tag

### Redis

- Redis: in-memory data structure store, used as DB, cache and message broker
- built-in replication & different levels of on-disk persistence

## Docker Container Links

- Docker Links allow containers to discover each other and to securely transfer data between them
- can have one-way link so that the recipient can access data from the source container, but source container cannot access the recipient's data
- links are established using container names

Example:
1. `docker run -d --name redis redis:3.2.0`
2. `docker built -t dockerapp:v0.3 .`
3. `docker run -d -p 5000:5000 --link redis dockerapp:v0.3`

### Benefits of Docker Container Links

- main use for docker container links is when we build an app with a microservice architecture. Container links allow us to run many independent components in different containers
- Docker creates secure tunnel between the containers that **doesn't need to expose any ports externally on the container**

## Automate Current Workflow with Docker Compose

### Why Docker Compose?

Manually linking containers & configuring services becomes impractical when the number of containers grows.

Compose is tool for defining & running multi-container Docker apps. Use a Compose file to configure your app's services. Then using single command, you create & start all services from your config.
Three-step process:

1. Define your app's environment with a Dockerfile so it can be reproduced anywhere
2. Define the services that make up your app in `docker-compose.yml` so they can be run together in isolated environment
3. Run `docker-compose up` and Compose will start and run your entire app.

Docker Compose is a separate tool than Docker engine. Docker Compose must also be downloaded & installed.

### `docker-compose.yml`

```
version: '3'                ## the version of Docker Compose to be used
services:                   ## list of microservices that make up the app
    dockerapp:              ## name of first microservice
        build: .            ## specifies filepath to service's Dockerfile (relative to the docker-compose.yml file)
        ports:
            - "5000:5000"   ## host_port:container_port
        depends_on:
            - redis         ## dockerapp is effectively a client of the redis service. This lets Docker know that the 
                            ## redis service should be started before the dockerapp service
                            ## with microservices, it is v. important that the microservices start up in the correct order!
    redis:
        image: redis:3.2.0  ## defines which image should be pulled to run the service named "redis"
                            ## all services need to have either a `build` instruction (to build via Dockerfile)
                            ## or a defined image to pull
```

**Your `docker-compose.yml` file doesn't need to define links if you are using Docker compose v.2 or higher.**

## Deep Dive into Docker Compose Workflow

### Docker Compose Commands

| Command | Description |
|---------|-------------|
| `docker-compose up` | starts up all the containers |
| `docker-compose ps` | checks the status of the containers managed by docker compose |
| `docker-compose logs` | outputs colour-coded & aggregated logs for the compose-managed containers |
| `docker-compose logs -f` | outputs appended log when the log grows |
| `docker-compose logs <container_name>` | outputs the logs of the specified container |
| `docker-compose stop` | stops all the running containers without removing them |
| `docker-compose rm` | removes all the containers |
| `docker-compose build` | rebuilds all the images (necessary when changes have been made before running docker-compose up) |


## Docker Networking

4 docker network types:
- Closed Network / None Network
- Bridge Network
- Host Network
- Overlay Network

Networking allows Docker containers to communicate with each other and with the outside world (e.g., the internet)

`docker network ls` will list the docker networks on your local box. By default, there will be 3 network installed when you install Docker: "bridge", "host", and "none".

## None Network

- doesn't have any access to outside world
- provides maximum level of network protection
- best option when a container requires maximum level of network security and doesn't require network access
- Example: `docker run -d --net none busybox sleep 1000` will start up a closed container in the none network from the busybox image

## Bridge Network

- default type of network in docker containers
- can create a new network with `docker network create --driver bridge my_bridge_network` 
  - the type of network is specified by the `--driver bridge` flag
  - the name `my_bridge_network` is assigned to the network
- use `docker network inspect <network_name>` to view all settings for the network
- Example: `docker run -d --name container_3 --net my_bridge_network busybox sleep 1000` to create a new container within the my_bridge_network work
- to connect container_3 to another network, use `docker network connect <network_name> container_3`
- can then run `docker exec -it container_3 ifconfig` to view the networks within container_3. 
- containers within the same network can ping each other & get response back
- disconnect a container from a network using `docker network disconnect <network_name> <container_name>`

- in a bridge network, containers have access to 2 network interfaces: loopback interface & private interface
- all containers in the same bridge network can communicate with each other
- containers from different bridge networks can't connect with each other by default
- bridge networks reduce level of network isolation in favour of better outside connectivity
- bridge networks most suitable when you want to set up relatively small network on a single host

### Host Network

- least protected network model
- adds a container on the host's network stack
- containers deployed on host stack (aka **open containers**) have full access to the host's interface
- `docker run -d --name container_4 --net host busybox sleep 1000`
- minimum network security level
- no isolation on this type of open container, thus leaving container widely unprotected
- Benefit: containers running in host network stack should have higher level of performance than containers traversing the docker0 bridge & iptables port mappings
- not likely to be used in PROD

### Overlay Network

- supports multi-host networking out-of-the-box
- require some pre-existing conditions before it can be created:
  - running Docker engine in Swarm mode
  - a key-value store service such as consul
- widely used in production

## Define Container Networks with Docker Compose

- specify the network name, type, and usage within docker-compose.yml file by adding

```
networks:
  network_name_1:
    driver: driver_type_1
  network_name_2:
    driver: driver_type_2
    driver_opts:
      foo: "1"
      bar: "2"
```

and then configure the services to use the desired networks by modifying the service's section of the file:

```
services:
  app:
    ...
    networks:
      - network_name_1
      - network_name_2
```

- network isolation "tricks" are popular in multi-tier applications

## Write & Run Unit Tests inside Containers

### Unit Tests in Containers

- unit tests should test some basic functionality of docker ap code, with no reliance on external services
- unit tests should run as quickly as possible so developers can iterate much faster w/o being blocked by waiting for test results
- Docker containers can spin up in seconds & can create a clean and isolated environment - great tool for running unit tests
- `docker-compose run <service_name> <command [args]>` will execute the given command(s) within the service specified. This can be used for running test suites

### Incorporating Unit Tests into Docker Images

**Pros:**
- a single image is used through dev, testing and production, which greatly ensures the reliability of tests

**Cons:**
- increases the size of the image. As long as the test suite isn't extremely large, the unit tests should be included in the same image as the app

## Continuous Integration

- a software engineering practice in which isolated changes are immediately tested & reported when they are added to a larger code base
- goal is to provide rapid feedback so that if a defect is introduced into the code base, it can be identified & corrected ASAP

## Setting up CI Workflow with CircleCI

- CircleCI's config set up with `config.yml` file inside `.circleci` dir from `dockerapp` repo for course

`config.yml`
```yaml
version: 2    # specifies which version of CircleCI to use. 2 is the latest
jobs:
  build:    # every config file must have a build job. This is the only job that will automatically be picked up by CircleCI
    working_directory: /dockerapp  # specifies default working dir for rest of job
    docker:
      - image: docker:17.05.0-ce-git  # enviro where steps will be executed.  '-git' means that git is pre-installed
    steps:
      - checkout
      - setup_remote_docker # creates separate env for each build for security. This env is remote and fully configured to execute Docker cmds
      # this step generally req'd every time running 
      - run:
          name: Install dependencies
          # install pip, then use pip to install docker-compose
          command: |
            apk add --no-cache py-pip=9.0.0-r1  
            pip install docker-compose=1.15.0
      - run:
          name: Run tests
          # run unit tests inside Docker container
          command: |
            docker-compose up -d
            docker-compose run dockerapp python test.py
      - deploy:
          name: Push application Docker image
          # login uses env variables for dockerhub login
          command: |
            docker login -e $DOCKER_HUB_EMAIL -u $DOCKER_HUB_USER_ID -p $DOCKER_HUB_PWD
            docker tag dockerapp_dockerapp $DOCKER_HUB_USER_ID/dockerapp:$CIRCLE_SHA1
            docker tag dockerapp_dockerapp $DOCKER_HUB_USER_ID/dockerapp:latest
            docker push $DOCKER_HUB_USER_ID/dockerapp:$CIRCLE_SHA1
            docker push $DOCKER_HUB_USER_ID/dockerapp:latest
```

- Go to https://circleci.com and log in with GitHub account. Add new project. Can view results of CI run from here.
- CircleCI will monitor commits to your configured GH repo and will automatically trigger builds whenever new commit is pushed

## Publish Docker Images to DockerHub from CircleCI

Complete CI Workflow:
GH repo --> CircleCI --> DockerHub --> Staging/Prod

Will need a DockerHub account. Can save DockerHub account credentials (`DOCKER_HUB_EMAIL`, `DOCKER_HUB_PWD`, `DOCKER_HUB_USER_ID`) to env variables in CircleCI project settings (from CircleCI website)

Tag the Docker image with 2 tags:
- commit hash of the source code
- `latest`

`docker tag [local_image_name] [repository_name]/[image_name]:[tag]`
  
the `$CIRCLE_SHA1` used for tagging is a pre-set CircleCI env var

## Intro to Running Docker in PROD

- generally, devs have mixed feelings about deploying to prod in Docker. Some concerns that Docker workflow is too complex and/or unstable for real use cases

### Concerns about running Docker in PROD
- some missing pieces about Docker around data persistence, networking, security & identity management
- ecosystem of supporting Dockerized applications in production, such as tools for monitoring & logging, not fully ready yet

Docker is becoming standard for building distributed applications in production envs

### Running Docker Containers inside VMs in PROD
- to address security concerns: isolation provided by Docker not as robust as segregation provided by hypervisors for VMs
- hardware level isolation: prevents one container from starving other containers of resources when on same host (denial of service)

**Docker Machine**: provision new VMs; install Docker Engine; configure Docker client

`docker-machine ls` will list all the VMs provisioned by Docker Machine

To provision a new VM in Docker machine, run
`docker-machine create --driver digitalocean --digitalocean-access-token <access-token> docker-app-machine`
The final arg is the name of the VM to be created.

Then run `docker-compose -f prod.yml up -d` to spin up a new VM.

Multiple VMs can be running in docker machine at the same time, but only one VM can be **active** at a time. The VM active is the default VM to which all commands will apply unless a command explicitly states which VM it should run on. Whenever a new VM is run, it will be set as the active VM.

## Docker Swarm

- Docker Swarm is a tool that clusters many docker engines and schedules containers
- DS decides on which host to run the container based on your scheduling methods
- a Docker Swarm Manager is set up as a middleman between the docker client and the multiple docker daemons running. Docker client connects to the swarm manager instead of an individual daemon.
- the Docker Swarm Manager knows the status of all nodes in the cluster

### Terminology

- **node**: instance of a docker engine participating in the swarm
- **worker node**: a node that receives and executes tasks dispatched from manager nodes
- **manager node**: a node that dispatches tasks to be executed by the worker nodes.

### How Swarm cluster works

To deploy your app to a swarm, submit your service to a manager node. Manager node dispatches units of work (called **tasks**) to worker nodes.

Manager nodes also perform orchestration and cluster management functions req'd to maintain the desired state of the swarm. Manager nodes elect a single leader to conduct the orchestration tasks for the swarm.

Worker nodes receive & execute tasks dispatched from manager nodes.

By default, manager nodes will also run services as worker nodes, but this can be configured otherwise.

An **agent** runs on each worker node & reports on the tasks assigned to it. Worker node notifies manager node of the current state of its assigned tasks so that the manager can maintain the desired state of each worker.

### Provisioning a Swarm Cluster

1. Deploy two VMs - one to be used for the Swarm manager node, the other to be used as worker node
2. Appoint the first VM as Swarm manager node & initialize a Swarm cluster using `docker swarm init`
3. Let second VM join Swarm cluster as worker node using `docker swarm join`

*Step 1.* Create a swarm manager by creating a new docker machine, exactly as done previously above.

```bash
docker-machine create --driver digitalocean --digitalocean-access-token <access_token> swarm-manager
docker-machine env swarm-manager
eval $(docker-machine env swarm-manager)
docker-machine create --driver digitalocean --digitalocean-access-token <access_token> swarm-node
```

^-- at this stage, the 2 VMs have been created

*Step 2.* First make sure that the swarm-manager is the active docker machine by running `docker-machine ls` and look for * next to swarm-manager under the ACTIVE column

`docker swarm init --advertise-addr <public_IP>` to initialize the swarm-manager node using the public IP network interface

^-- at this stage, the swarm cluster has been initialized. The CLI output of the last command will provide instructions on how to add worker nodes to the swarm cluster.

*Step 3.* Rather than setting the worker node (`swarm-node`) VM to be active, we can `ssh` into the VM from the current active `swarm-manager` VM by running `docker-machine ssh swarm-node`. Once we've ssh'ed into the other VM, we can run the command provided at the end of Step 2 to join the swarm-node to the swarm cluster.

^-- final outcome of this is a 2-node swarm cluster.

### Important Docker Swarm commands

`docker swarm init`
- initialize a swarm. The docker engine targeted by this command becomes a manager in the newly created single-node swarm

`docker swarm join`
- join a swarm as a Swarm node

`docker swarm leave`
- disconnect a node from the swarm

## Deploy Docker App Services to Cloud via Docker Swarm

### Docker Services

Services defined in Docker compose file.
Service definition includes which Docker images to run, port mapping & dependencies between services.

When we are deploying services in swarm mode, we can also set another important config, which is the `deploy` key - only available on Compose file formats version 3.x and up. `deploy` key and its sub-options can be used to load balance & optimize performance for each service.

Example docker-compose.yml:

```yml
version: "3.3"
services:
  nginx:
    image: nginx:latest
    ports:
      - "80:80"
    deploy:
      replicas: 3  # can access nginx by visiting port 80 on any of the 3 hosts
      resources:
        limits:
          cpus: '0.1'  # max 10% of CPU capacity
          memory: 50M
      restart_policy:
        condition: on-failure
```

What happens if there are more nodes than there are replicas running?
- we can connect to the nginx service through a node which doesn't have nginx replicas. Docker calls this **ingress load balancing**
  - all nodes listen for connections to published service ports
  - when that service is called by external systems, receiving node will accept the traffic & internally load balance it using internal DNS service that Docker maintains

### Docker Stack

- a **docker stack** is a group of interrelated services that share dependencies, & can be orchestrated and scaled together.
- can imagine that a stack is a live collection of all the services defined in docker compose file
- create a stack from docker compose file: `docker stack deploy`
- in swarm mode,
  - docker compose files can be used for service definitions
  - docker compose commands can't be reused. Docker compose commands can only schedule the containers to a single node
  - have to use `docker stack` command. Think of docker stack as the docker compose in swarm mode

`docker stack deploy --compose-file prod.yml dockerapp_stack`
  - docker stack will create a default network called `dockerapp_stack_default` that will be used for cross-node communication within the swarm

`docker stack services dockerapp_stack` lists all services running in the dockerapp_stack swarm

### How to Update Services in Production?

UPdate the docker compose file, then re-run docker stack command, and docker will automatically update the services

## Misc.

Think of Kubernetes (a container orchestration platform) as an alternative to Docker Swarm
