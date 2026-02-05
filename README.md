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