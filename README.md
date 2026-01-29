<h1> CHAPTER 1</h1>
benif of using docker or containers in general : 
    -saving time , by reducing time for VM to boot 
    -resource, CPU , RAM
    -capital, 

Windows containers vs Linux containers : 
    - containerized Widnows app will not run on a Lunux-based Docker host and vice-versa 
    - how ever it is possible to run linux containers on windows machine . if docker desktop on windows has two modes , ( windows container and linux containers ) , linux containers run either inside a lightweigh Hyper-v VM or WSL (windows sub linux) 

What about kubernetes ?? :
    - `containerd` is the small specialized part of Docker that does the low-level tasks of starting and stopping containers.

Docker technology :
    1- The runtime
    2- The daemon (engine)
    3- The orchestrator 

1) The runtime :
    - there is 2 levels or runtime 
    1- the low-level 
    2- the higher-level

    #low-level is called `runc` and is the reference implemetation of Open COntainers Initiative 
    its job is to interface with the undelying OS and start and stop containers. each Docker node has a runc instance managing it 

    #higher-level runtime called `containerd` . this one does a lot more than `runc` is manage the entire lifecycle of a container, including pulling images, creating network interfaces , and managing lowe-level runc isntances. 

2) docker daemon (dockerd) :
    - sits above `containerd` and performs higher-level taks such as: exposing the docker remote API , managing images , managing volumes, networks and etc ...  

$notes:  managing clusters , Docker Swarm vs Kubernetes 


https://labs.play-with-doer.com/


#commands 
-docker image ls 
-docker image pull ubuntu:latest
-docker container run -it ubuntu:latest /bin/bash (-it switch your sell into the terminal of the container)
-docker container exec <options> <container_name or id> <command / app to run in the container>
-docker container stop <container_name>
-docker cotainer rm <cotnainer_name>
-docker container ls -a (to list cotnainer even those in the stopped state)

-docker image build (to create an image)
-docker image build -t <ubuntu:latest>


<h1> CHAPTER 2</h1>


TLDR => Docker engine is the software that runs and manages containers

-Docker engine is made from many specialized tools that works together to create and run conainers "API , excution driver, etc ..."

what make the docker engine are : 
{docker daemon, containerd, runc , networking and storage .}

#Docker daemon was a monolithic binary, it contained all of the code for the Docker client, the Docker Api , container runtime. omage builds etc .. 



As previously mentioned, runc is the reference implementation of the OCI container-runtime-spec. Doer, Inc.
was heavily involved in deﬁning the spec and developing runc.

runc has a single purpose in life — create containers.


container'd: 
Its sole purpose in life
was to manage container lifecycle operations — start | stop | pause | rm....


docker socket file : 
sudo curl --unix-socket /var/run/docker.sock http://localhost/version
using this we could by pass the Docker CLI , CLI convert text to API request 

docker daemon api socket network can be exposed. On linux 
is /var/run/docker.sock 
On windows
is \pipe\docker_engine

once the daemon receices the command the create a new conainer , it makes a call to containerdd


the daemon communicates with containerd via a CRUD stype API over gRPC


containerd cannot actualy create containers. it uses runc to do that it converts the required Docker image into a OCI bundle and tells runc to use this to create a new conatiner.


runc interfaces with the OS kernel to pull together all of the constructs necessary to create a container (namespaces, cgroups etc)
.The container Process is started as a child-process of runc , and soon as it started runc will exit


runc exits after container start 
,shim become conatiner's parent process 


with this we decoupled  from the docker daemon , called daemonless containers , to make it possible to perform maintenance and upgrade on the docker daemon without impacting running containers!



Some of the responsibilities the shim performs as a container’s parent include:
• Keeping any STDIN and STDOUT streams open so that when the daemon is restarted, the container
doesn’t terminate due to pipes being closed etc.
• Reports the container’s exit status ba to the daemon.

<h3> How its implemented on Linux</h3>
.dockerd (the Doer daemon)
• docker-containerd (containerd)
• docker-containerd-shim (shim)
• docker-runc (runc)

#secure connection 
default port for 2375/tcp.


1) creat a Ca (self-signed certs)
    -
    -openssl genrsa -aes256 -out ca-key.pem 4096 (generating RSA private key )

    - we use the ca-key.pem private key to generate public key (certificate)
    - openssl req -new -x509 -days 730 -key ca-key.pem -sha256 -out ca.pem
    generate a key a.k.a certificate


Warning! Linux systems running systemd don’t allow you to use the “hosts” option in daemon.json. Instead,
you have to specify it in a systemd override ﬁle. You may be able to do this with the sudo systemctl edit
docker command. is will open a new ﬁle called /etc/systemd/system/docker.service.d/override.conf in
an editor. Add the following three lines and save the ﬁle.
[Service]
ExecStart=
ExecStart=/usr/bin/dockerd -H tcp://node3:2376


2) Now for the security part:
    Use TLS, to secure both  the client and the daemon. 

    there is 2 modes:
        -client mode 
        -daemon mode

e high-level process will be as follows:
1. Conﬁgure a CA and certiﬁcates
2. Create a CA
3. Create and sign keys for the Daemon
4. Create and sign keys for the Client
5. Distribute keys
6. Conﬁgure Doer to use TLS
7. Conﬁgure daemon mode
8. Conﬁgure client mode



<h1>Chapter 3</h1>

**images**
you can think of images as class 
meaning a blue print how to make the container , or we could say that a image is read only container .

you can pull images from the registry most common registry `Docker Hub` . you pull the image to ur docker host .where docker can use it to start one or more containers.

Images are mad up of multiple layers that are stacked on top of each others.

**Docker images - Deep dive**

the two constructs become dependent or each other , you cannot delete the image untile the last container using it has been stopped and destroyed. 


images get stored in centralised place called iamge registries.


for alpine:edge to pull the latest version 


