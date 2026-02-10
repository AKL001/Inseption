# Docker Learning Notes

## Table of Contents

- [Chapter 1: Docker Fundamentals](#chapter-1-docker-fundamentals)
  - [Benefits of Using Docker](#benefits-of-using-docker-or-containers-in-general)
  - [Windows vs Linux Containers](#windows-containers-vs-linux-containers)
  - [Kubernetes](#what-about-kubernetes)
  - [Docker Technology](#docker-technology)
  - [Common Commands](#common-commands)
- [Chapter 2: Docker Engine Architecture](#chapter-2-docker-engine-architecture)
  - [Components of Docker Engine](#components-of-docker-engine)
  - [runc](#runc)
  - [containerd](#containerd)
  - [Docker Socket File](#docker-socket-file)
  - [Container Creation Process](#container-creation-process)
  - [The Shim](#the-shim)
  - [Secure Connection](#secure-connection)
- [Chapter 3: Images and Containers](#chapter-3-images-and-containers)
  - [Images](#images)
  - [Containers](#containers)
  - [Container Lifecycle](#container-lifecycle)
  - [Port Mappings](#port-mappings)
  - [Container Commands Reference](#container-commands-reference)
- [Chapter 4: Containerizing an Application](#chapter-4-containerizing-an-application)
  - [Example Dockerfile](#example-dockerfile)
  - [Building the Image](#building-the-image)
  - [Multi-stage Builds](#multi-stage-builds)
  - [Squashing Images](#squashing-images)
  - [Dockerfile Commands Summary](#dockerfile-commands-summary)
- [Chapter 5: Deploying Apps with Docker Compose](#chapter-5-deploying-apps-with-docker-compose)
  - [Microservices](#microservices)
  - [YAML File Structure](#yaml-file-structure)
  - [Example docker-compose.yml](#example-docker-composeyml)
  - [Docker Compose Commands](#docker-compose-commands)
  - [Docker Compose Commands Summary](#docker-compose-commands-summary)
- [Chapter 6: Docker Networking](#chapter-6-docker-networking)
  - [CNM Components](#cnm-components)
  - [The CNM Building Blocks](#the-cnm-building-blocks)
  - [Single-host Bridge Networks](#single-host-bridge-networks)
  - [Network Types](#network-types)
- [Chapter 7: Docker Volumes and Persistent Data](#chapter-7-docker-volumes-and-persistent-data)
  - [Data Types](#data-types)
  - [Container Storage](#container-storage)
  - [Persistent Data Volumes](#persistent-data-volumes)
  - [Sharing Storage Across Hosts](#sharing-storage-across-hosts-or-cluster-nodes)
- [Chapter 8: Docker Security](#chapter-8-docker-security)
  - [Linux Security Technologies](#linux-security-technologies)
  - [Namespaces](#namespaces)
  - [Control Groups (cgroups)](#control-groups-cgroups)

---

## Chapter 1: Docker Fundamentals

### Benefits of Using Docker or Containers in General

- Saving time by reducing time for VM to boot
- Resource efficiency (CPU, RAM)
- Capital savings

### Windows Containers vs Linux Containers

- Containerized Windows apps will not run on a Linux-based Docker host and vice-versa
- However, it is possible to run Linux containers on Windows machines. Docker Desktop on Windows has two modes (Windows containers and Linux containers). Linux containers run either inside a lightweight Hyper-V VM or WSL (Windows Subsystem for Linux)

### What About Kubernetes?

- `containerd` is the small specialized part of Docker that does the low-level tasks of starting and stopping containers

### Docker Technology

1. The runtime
2. The daemon (engine)
3. The orchestrator

#### 1) The Runtime

There are 2 levels of runtime:
1. The low-level
2. The higher-level

**Low-level runtime** is called `runc` and is the reference implementation of Open Containers Initiative. Its job is to interface with the underlying OS and start and stop containers. Each Docker node has a runc instance managing it.

**Higher-level runtime** called `containerd`. This one does a lot more than `runc` - it manages the entire lifecycle of a container, including pulling images, creating network interfaces, and managing lower-level runc instances.

#### 2) Docker Daemon (dockerd)

- Sits above `containerd` and performs higher-level tasks such as: exposing the Docker remote API, managing images, managing volumes, networks, etc.

**Notes:** Managing clusters: Docker Swarm vs Kubernetes

**Practice Lab:** https://labs.play-with-docker.com/

### Common Commands

```bash
docker image ls
docker image pull ubuntu:latest
docker container run -it ubuntu:latest /bin/bash  # -it switch your shell into the terminal of the container
docker container exec <options> <container_name or id> <command/app to run in the container>
docker container stop <container_name>
docker container rm <container_name>
docker container ls -a  # to list containers even those in the stopped state

docker image build  # to create an image
docker image build -t <ubuntu:latest>
```

---

## Chapter 2: Docker Engine Architecture

**TLDR:** Docker engine is the software that runs and manages containers.

Docker engine is made from many specialized tools that work together to create and run containers (API, execution driver, etc.)

### Components of Docker Engine

- Docker daemon
- containerd
- runc
- Networking and storage

**Note:** The Docker daemon was originally a monolithic binary. It contained all of the code for the Docker client, the Docker API, container runtime, image builds, etc.

### runc

As previously mentioned, runc is the reference implementation of the OCI container-runtime-spec. Docker, Inc. was heavily involved in defining the spec and developing runc.

runc has a single purpose in life — create containers.

### containerd

Its sole purpose in life was to manage container lifecycle operations — start | stop | pause | rm...

### Docker Socket File

```bash
sudo curl --unix-socket /var/run/docker.sock http://localhost/version
```

Using this we could bypass the Docker CLI. The CLI converts text to API requests.

The Docker daemon API socket can be exposed:
- On Linux: `/var/run/docker.sock`
- On Windows: `\\pipe\\docker_engine`

Once the daemon receives the command to create a new container, it makes a call to containerd.

The daemon communicates with containerd via a CRUD-style API over gRPC.

### Container Creation Process

containerd cannot actually create containers. It uses runc to do that. It converts the required Docker image into an OCI bundle and tells runc to use this to create a new container.

runc interfaces with the OS kernel to pull together all of the constructs necessary to create a container (namespaces, cgroups, etc). The container process is started as a child-process of runc, and as soon as it starts, runc will exit.

### The Shim

After runc exits, the shim becomes the container's parent process.

This decouples the container from the Docker daemon, called **daemonless containers**. This makes it possible to perform maintenance and upgrades on the Docker daemon without impacting running containers!

#### Shim Responsibilities

Some of the responsibilities the shim performs as a container's parent include:

- Keeping any STDIN and STDOUT streams open so that when the daemon is restarted, the container doesn't terminate due to pipes being closed, etc.
- Reports the container's exit status back to the daemon

### Linux Implementation

- `dockerd` (the Docker daemon)
- `docker-containerd` (containerd)
- `docker-containerd-shim` (shim)
- `docker-runc` (runc)

### Secure Connection

Default port: `2375/tcp`

#### 1) Create a CA (Self-Signed Certs)

```bash
# Generating RSA private key
openssl genrsa -aes256 -out ca-key.pem 4096

# Use the ca-key.pem private key to generate public key (certificate)
openssl req -new -x509 -days 730 -key ca-key.pem -sha256 -out ca.pem
```

**Warning!** Linux systems running systemd don't allow you to use the "hosts" option in daemon.json. Instead, you have to specify it in a systemd override file. You may be able to do this with the `sudo systemctl edit docker` command. This will open a new file called `/etc/systemd/system/docker.service.d/override.conf` in an editor. Add the following three lines and save the file:

```ini
[Service]
ExecStart=
ExecStart=/usr/bin/dockerd -H tcp://node3:2376
```

#### 2) Security with TLS

Use TLS to secure both the client and the daemon.

There are 2 modes:
- Client mode
- Daemon mode

The high-level process will be as follows:

1. Configure a CA and certificates
2. Create a CA
3. Create and sign keys for the Daemon
4. Create and sign keys for the Client
5. Distribute keys
6. Configure Docker to use TLS
7. Configure daemon mode
8. Configure client mode

---

## Chapter 3: Images and Containers

### Images

You can think of images as classes - a blueprint for how to make a container. Or we could say that an image is a read-only container.

You can pull images from the registry. The most common registry is **Docker Hub**. You pull the image to your Docker host, where Docker can use it to start one or more containers.

Images are made up of multiple layers that are stacked on top of each other.

#### Docker Images - Deep Dive

The two constructs become dependent on each other. You cannot delete the image until the last container using it has been stopped and destroyed.

Images get stored in a centralized place called **image registries**.

For `alpine:edge`, to pull the latest version.

**The relationship between images and containers:**

### Containers

```bash
docker version  # to check if Docker is running
service docker status
systemctl is-active docker
```

#### What Happens When You Run Multiple Containers from a Single Image?

**What happens when you `docker run`:**

Docker will:
- Pull/use an image
- Create a new container ID
- Create a new writable layer
- Store metadata like:

```bash
container_id → {
  image_id,
  graphdriver: overlay2,
  upperdir: /var/lib/docker/overlay2/XYZ/diff,
  lowerdir: image layers,
  mergeddir: /var/lib/docker/overlay2/XYZ/merged
}
```

This writes to disk immediately.

**When you run `docker start`:**

Docker will:
- Look up the container ID
- Load its saved metadata
- Remount:
  - Same UpperDir (writable layer)
  - Same LowerDir (image layers)
- Recreate the merged filesystem view
- Start the process

To see this, run: `docker inspect <container>`

### Container Lifecycle

You start, stop, delete, and remove a container.

Before removing a container, best practice is to shut it off (stop the container).

To save the data of a container, Docker provides **volumes** that exist separately from the container lifecycle but can be mounted into the container at runtime.

#### How Volumes Work

1. Docker creates a volume on the Docker host
2. The volume is not part of the container
3. When the container starts, Docker mounts that volume into a folder inside the container
4. The container reads/writes data there as if it were normal files
5. If the container stops or is deleted, the volume (and data) stays

**Example:**

```bash
docker volume create mydata
docker run -v mydata:/app/data myimage
```

#### Starting a Container in Daemon Mode

We use the `-d` flag to tell the container to run in the background.

### Port Mappings

Format: `(0.0.0.0:80->8080/tcp)`
- `host-port:container-port`

---

## Container Commands Reference

- **`docker container run`** - Command used to start new containers. In its simplest form, it accepts an image and a command as arguments. The image is used to create the container and the command is the application the container will run when it starts. This example will start an Ubuntu container in the foreground and tell it to run the Bash shell: `docker container run -it ubuntu /bin/bash`

- **`Ctrl-PQ`** - Will detach your shell from the terminal of a container and leave the container running (UP) in the background

- **`docker container ls`** - Lists all containers in the running (UP) state. If you add the `-a` flag you will also see containers in the stopped (Exited) state

- **`docker container exec`** - Runs a new process inside of a running container. It's useful for attaching the shell of your Docker host to a terminal inside of a running container. This command will start a new Bash shell inside of a running container and connect to it: `docker container exec -it <container-name or container-id> bash`. For this to work, the image used to create the container must include the Bash shell

- **`docker container stop`** - Will stop a running container and put it in the Exited (0) state. It does this by issuing a SIGTERM to the process with PID 1 inside of the container. If the process has not cleaned up and stopped within 10 seconds, a SIGKILL will be issued to forcibly stop the container. `docker container stop` accepts container IDs and container names as arguments

- **`docker container start`** - Will restart a stopped (Exited) container. You can give `docker container start` the name or ID of a container

- **`docker container rm`** - Will delete a stopped container. You can specify containers by name or ID. It is recommended that you stop a container with the `docker container stop` command before deleting it with `docker container rm`

- **`docker container inspect`** - Will show you detailed configuration and runtime information about a container. It accepts container names and container IDs as its main argument

---

## Chapter 4: Containerizing an Application

### Example Dockerfile

```bash
$ cat Dockerfile
  FROM alpine     # (image layer)
  LABEL maintainer="youremail@hotmail.com" 
  RUN apk add --update nodejs nodejs-npm  # (image layer) 
  COPY . /src  # (image layer) 
  WORKDIR /src # (image metadata)
  RUN npm install  # (image layer)
  EXPOSE 8080   # (also metadata)
  ENTRYPOINT ["node", "./app.js"]
```

Some instructions create image layers and some create metadata.

**Examples of instructions that create layers:** `FROM`, `RUN`, `COPY`

**Examples of instructions that create metadata:** `EXPOSE`, `WORKDIR`, `ENV`, `ENTRYPOINT`

### Building the Image

- **`docker image build -t <REPO:TAG> .`** - We include the `.` at the end of the command to tell Docker to use the shell's current working directory as the build context. `-t` tells Docker to name the image using the tag, making it easy to run with a human-readable name.

- **`docker image history <REPO:TAG>`** - We can inspect or see instructions that added layers and metadata (metadata size would be 0).

You can view the output of the `docker image build` command to see the general process for building an image. As the following snippet shows, the basic process is: spin up a temporary container > run the Dockerfile instruction inside of that container > save the results as a new image layer > remove the temporary container.

### Multi-stage Builds

Multi-stage builds are all about optimizing builds without adding complexity. And they deliver on the promise!

Often when you build a multi-stage image, you would see a `<none>` image - that's called a **dangling image**. These are "blueprints" used to build the final product.

**Pros:** For multi-stage builds we use `--from`. This helps with lightweight image production. Instead of having a production image that is 500MB in size, we divide that into 2 or 3 stages and copy the required dependencies from the stages.

### Squashing Images

**Pros:** It can help with the image size

**Cons:** There is no caching hit, builds may take more time now

### Dockerfile Commands Summary

- **`docker image build`** - The command that reads a Dockerfile and containerizes an application. The `-t` flag tags the image, and the `-f` flag lets you specify the name and location of the Dockerfile. With the `-f` flag, it is possible to use a Dockerfile with an arbitrary name and in an arbitrary location. The build context is where your application files exist, and this can be a directory on your local Docker host or a remote Git repo.

- **`FROM`** instruction in a Dockerfile specifies the base image for the new image you will build. It is usually the first instruction in a Dockerfile and a best practice is to use images from official repos on this line.

- **`RUN`** instruction in a Dockerfile allows you to run commands inside the image. Each RUN instruction creates a single new layer.

- **`COPY`** instruction in a Dockerfile adds files into the image as a new layer. It is common to use the COPY instruction to copy your application code into an image.

- **`EXPOSE`** instruction in a Dockerfile documents the network port that the application uses.

- **`ENTRYPOINT`** instruction in a Dockerfile sets the default application to run when the image is started as a container.

- **Other Dockerfile instructions include:** `LABEL`, `ENV`, `ONBUILD`, `HEALTHCHECK`, `CMD` and more...

---

## Chapter 5: Deploying Apps with Docker Compose

### Microservices

Docker Compose is an external Python binary. You define multi-container (microservices) apps in a YAML file.

### YAML File Structure

YAML files have 4 top-level keys:
- `version`
- `services`
- `networks`
- `volumes`
- `secrets`
- `configs`

#### Key Descriptions

- **`version`** - The version key is mandatory, and it's always the first line at the root of the file. This defines the version of the Compose file format (basically the API). You should normally use the latest version.

- **`services`** - The top-level services key is where you define the different application microservices. This example defines two services: a web front-end called `web-fe`, and an in-memory database called `redis`. Compose will deploy each of these services as its own container.

- **`networks`** - The top-level networks key tells Docker to create new networks. By default, Compose will create bridge networks. These are single-host networks that can only connect containers on the same Docker host. However, you can use the driver property to specify different network types.

### Example docker-compose.yml

```yaml
version: "3.8"
services:
  web-fe:
    build: .
    command: python app.py
    ports:
      - target: 5000
        published: 5000
    networks:
      - counter-net
    volumes:
      - type: volume
        source: counter-vol
        target: /code
  redis:
    image: "redis:alpine"
    networks:
      counter-net:
networks:
  over-net:
    driver: overlay
    attachable: true
volumes:
  counter-vol:
```

### Docker Compose Commands

- **`docker-compose up`** - Launch Docker Compose application

By default, `docker-compose up` expects the name of the Compose file to be `docker-compose.yml`. If your Compose file has a different name, you need to specify it with the `-f` flag.

- **`docker-compose -f <docker-compose-name> up`** - Specify a custom compose file name

- **`docker-compose -f prod-equus-bass.yml up -d`** - Run it in the background (daemon mode)

- **`docker-compose down`** - Bring the application down or stop it

- **`docker-compose ps`** - List each container in the Compose app with current state, command, and network ports

- **`docker-compose top`** - List running processes

- **`docker-compose stop`** - Stop all containers in a Compose app without deleting them

- **`docker-compose restart`** - Restart a Compose app that has been stopped

- **`docker volume inspect counter-app_counter-vol | grep Mount`** - Inspect volume mount points

### Docker Compose Commands Summary

- **`docker-compose up`** - The command to deploy a Compose app. It expects the Compose file to be called `docker-compose.yml` or `docker-compose.yaml`, but you can specify a custom filename with the `-f` flag. It's common to start the app in the background with the `-d` flag.

- **`docker-compose stop`** - Will stop all of the containers in a Compose app without deleting them from the system. The app can be easily restarted with `docker-compose restart`.

- **`docker-compose rm`** - Will delete a stopped Compose app. It will delete containers and networks, but it will not delete volumes and images.

- **`docker-compose restart`** - Will restart a Compose app that has been stopped with `docker-compose stop`. If you have made changes to your Compose app since stopping it, these changes will not appear in the restarted app. You will need to re-deploy the app to get the changes.

- **`docker-compose ps`** - Will list each container in the Compose app. It shows current state, the command each one is running, and network ports.

- **`docker-compose down`** - Will stop and delete a running Compose app. It deletes containers and networks, but not volumes and images.

---

## Chapter 6: Docker Networking

Docker networking is based on an open-source pluggable architecture called **Container Network Model (CNM)**.

### CNM Components

- **The Container Network Model (CNM)** - The design specification
- **`libnetwork`** - Docker's real-world implementation of the CNM
- **Drivers** - Extend the model by implementing specific network topologies

### The CNM Building Blocks

The CNM defines 3 major building blocks:

- **Sandboxes** - An isolated network stack. It includes Ethernet interfaces, ports, routing tables, etc.
- **Endpoints** - Virtual network interfaces. They are responsible for making connections. Their job is to connect a sandbox to a network.
- **Networks** - Software implementation of a switch. They group together and isolate a collection of endpoints that need to communicate.

### Single-host Bridge Networks

Imagine we have 2 different hosts, Host A and Host B, both on the same network, and we have a container running on each of them. You may think that the containers can talk to each other because the hosts are on the same network - not really. Docker creates a bridge by default if you don't specify it. With that, the containers inside the Docker host can communicate with each other, but not with other hosts even if they are on the same network.

- You can override this in the command using the `--network` flag to specify the type of network and name of it.
- Use `docker network inspect bridge` to inspect the IPs, Driver, etc.
- `docker network inspect localnet --format '{{json .Containers}}'` - Format output as JSON
- `brctl show` - A command that shows the bridges

### Network Types

**Docker bridge network** (single-host bridge networks):
- Creating a custom bridge lets you use DNS

**Docker host network**:
- This can be used for production
- The container uses the same IP as the host network

**IPvlan**:
- Advanced networking option for more complex setups

---

## Chapter 7: Docker Volumes and Persistent Data

### Data Types

There are two types of data:
- **Persistent data** - Data you need to keep (customer records, financial data, research results, audit logs, application log data)
- **Non-persistent data** - Data you don't need to keep

To deal with **non-persistent data**, Docker containers handle that automatically. It's tightly coupled to the lifecycle of the container.

To deal with **persistent data**, the container needs to store it in a **volume**. Volumes are separate objects that have their lifecycles decoupled from containers.

### Container Storage

The writable container layer exists in the filesystem of the Docker host, called (local storage, ephemeral storage, or graphdriver storage).

**Linux path:** `/var/lib/docker/<storage-driver>/...`

### Persistent Data Volumes

When you have persistent data or data that you want to save even if you stop the container or delete it, you should use volumes:

- Volumes are independent objects that are not tied to the lifecycle of a container
- Volumes enable multiple containers on different Docker hosts to access and share the same data

#### Creating and Using Volumes

1. **`docker volume create <volume_name>`** - Create a volume

2. Then mount that volume to the container

**Note:** If you did not specify the path for the host, you can find that volume by using `docker volume inspect <name>` and look for `Mountpoint`.

#### How to Delete a Docker Volume

- **`docker volume rm <volume_name>`** - Use this to delete a specific volume
- **`docker volume prune`** - Deletes all the unused volumes

#### Mounting a Volume to a Container

We need to run a container with some specific flags:

- **`docker container run -dit --name <container_name> --mount source=<volume_name>,target=<path_in_container> <image_name>`** - This would run the container and mount it with the source and target.
  - `-d` for detach from the terminal and run the container in the background
  - `-i` keeps STDIN (Standard Input) open even if not attached
  - `-t` makes the container think it's connected to a real terminal screen

- **`docker volume ls`** - To see if the volume is created

### Sharing Storage Across Hosts or Cluster Nodes

These external systems can be cloud storage services or enterprise storage systems in your on-premises data centers.

Building a setup like this requires a lot of things. You need access to a specialized storage system and knowledge of how it works and presents storage. You also need to know how your applications read and write data to the shared storage. Finally, you need a volume driver plugin that works with the external storage system.

Docker Hub is the best place to find volume plugins. Login to Docker Hub, select the view to show plugins instead of containers, and filter results to only show Volume plugins. Once you've located the appropriate plugin for your storage system, you create any configuration files it might need, and install it with `docker plugin install`.

- **`docker plugin ls`** - List installed plugins

---

## Chapter 8: Docker Security

### Linux Security Technologies

Docker leverages several Linux security technologies:

- Namespaces
- Control Groups (cgroups)
- Capabilities
- Mandatory Access Control (MAC)
- seccomp

### Namespaces

- **`pstree -p`** - To see the process tree

Namespaces use the system call `unshare`. Namespaces are the reason why processes in containers think they are isolated.

Docker uses these namespaces for isolation:

- **PID** - The container thinks its main app is PID 1, but on the host, it might actually be PID 4502.

- **NET** - The container has its own IP (172.17.0.2) and doesn't see the host's Wi-Fi or Ethernet.

- **MOUNT** - The container sees `/` as its own image, not the host's `/home` or `/etc` folders.

#### Namespaces Summary

- Namespaces isolate processes on a single OS

Docker on Linux currently utilizes the following kernel namespaces:
- Process ID (pid)
- Network (net)
- Filesystem/mount (mnt)
- Inter-process Communication (ipc)
- User (user)
- UTS (uts)

**Remember:** A container is a collection of namespaces packaged and ready to use.

### Control Groups (cgroups)

If namespaces are about **isolation**, control groups (cgroups) are about **setting limits**.

Containers are isolated from each other but all share a common set of OS resources - things like CPU, RAM, and network bandwidth.

Cgroups let us set limits on each of these so a single container cannot consume everything and cause a denial of service (DoS) attack.