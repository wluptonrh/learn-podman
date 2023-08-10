# Lets learn podman

## Getting started with our lab environment
1. Navigate to the [lab](https://www.redhat.com/en/interactive-labs/red-hat-enterprise-linux-open-lab). Click "Launch." The lab may take a minute or two to provision. This lab anvironment will last for about an hour.

2. Install podman

      ```shell
       dnf install podman 
      ```
3. Optional: Sign into the the RHEL web console.
    - Username: rhel
    - Password: redhat


## Exercise 1: Pulling and Running Containers

### Launching our first container
This command will pull down the RHEL UBI container image. I've included a couple of options so we can access the container interactively. Those options are broken down below:
```shell 
podman run -it --name mycontainer registry.access.redhat.com/ubi9:latest /bin/bash
```

When you see a prompt like the following, you are in the container. Try running a few basic linux commands (ls, whoami, etc).
```shell
[root@fc97e5b55498 /]#  
```
  NOTE:

  - **-it:** Enables an interactive shell session within the container.
  - **registry.access.redhat.com/ubi9:latest:** This is the name of the image used to create the container. In this case, it is the latest version of the Red Hat Universal Base Image (UBI) version 9 from the Red Hat container registry. If the image is not available locally, podman will automatically pull it from the registry.
  - **/bin/bash:** This is the command to be executed inside the container. It launches an interactive Bash shell, providing you with a command-line interface within the UBI 9 container.

Let's exit our container
```shell 
[root@7ca6cc0ae00d /]# exit
```

Since the container is no longer running our interactive shell, it has no reason to keep running in the background. We can verify this by running ```console podman ps -a``` 

You'll notice the container indicates a status of "Exited"

We can also inspect the images stored locally on our system by running
`podman images`. You'll notice our UBI image is now present.




## Exercise 2: Building container images

### Creating our Containerfile

In this exercise, you will create a basic Apache container image. To do so, we will create a Containerfile which is a configuration file that automates the steps of creating a container image.

To start, lets open a text editor (vim) to create a container file. The following command will create a file named "Containerfile" and will open up our text editor.
```shell 
vim Containerfile 
```
Inside of the vim text editor, enter insert mode by pressing the "i" key. Now edit this file the configurations shown below. Once you are finished, you can save and exit this file by typing ":wq" 

```yaml 
FROM registry.access.redhat.com/ubi8/ubi:8.8

RUN yum install -y httpd && \
    yum clean all

RUN echo "Hello, our webserver is working!" > /var/www/html/index.html

EXPOSE 80

CMD ["httpd", "-D", "FOREGROUND"]

```
NOTE: 
- FROM registry.access.redhat.com/ubi8/ubi:8.8: This line sets the base image for the container image you are building. In this case, it uses the Red Hat Universal Base Image (UBI) version 8.8.

- RUN yum install -y httpd && \: This line uses the RUN instruction to execute a shell command within the container during the image build process. It installs the Apache HTTP Server (httpd).

- yum clean all: This command cleans the yum package manager cache after the installation of httpd, reducing the size of the container image.

- RUN echo "Hello from Containerfile" > /var/www/html/index.html: This line creates a simple HTML file with the content "Hello from Containerfile" and saves it in the /var/www/html/ directory. This file will be served by the Apache HTTP server when the container runs.

- EXPOSE 80: This instruction informs podman that the container will listen on port 80 at runtime. However, it does not publish the port to the host machine. To access the container's port 80 from the host machine, you would need to publish it explicitly when running the container.

- CMD ["httpd", "-D", "FOREGROUND"]: This sets the default command to run when the container starts. It starts the Apache HTTP server with the -D FOREGROUND option, which allows the container to stay in the foreground and log output to the terminal.

### Building our container image

In the directory where our Container file exists, we can instruct podman to create our image. We will also tag this image as apache-img.

```shell 
podman build --layers=false -t apache-img .
``` 

We can use ``` podman images``` to see our image stored locally on our machine

### Running the container image

Now let's run our container from our newly created image.

```shell 
podman run --name apache-container -d -p 10080:80 apache-img
```


NOTE
- **--name apache-container:** This flag sets the name of the container to be apache-container. Containers are typically assigned random names if not specified, but with this flag, you explicitly provide a name.

- **-d:** This flag stands for "detached" mode, which means the container will run in the background (daemon mode). It allows you to continue using the terminal after starting the container.

- **-p 10080:80:** This flag is used to publish a container's port to the host machine. It maps port 80 inside the container to port 10080 on the host machine. 

- **apache-img:** This is the name of the image from which the container will be created. In this case, it refers to the image you built earlier using the apache-img tag.

Optional: You start an interactive shell session or run any other command inside of a running container via ```podman exec```

```shell
podman exec -it apache-container /bin/bash
```

### Testing our webserver 

Run the following command to verify our apache webserver is running.
```shell 
curl localhost:10080
```

To stop the container
```shell
podman stop apache-container
```

To remove the container (once it's stopped):
```shell
podman rm apache-container
```


## Exercise 3: Attaching persistent storage to a container

Container storage is said to be ephemeral, meaning its contents are not preserved after the container is removed. In this exercise, we will look at one method we can use to mount a host directory to a container

A volume is a storage device created and managed by Podman. Volumes are created directly using the podman volume command or during container creation.

For this example, lets use a volume mount for logging to the container host for event retention

```shell 
podman run -v /dev/log:/dev/log --rm ubi9 logger Testing logging to the host
```
NOTE
 - **-v /dev/log:/dev/log:** This option creates a volume bind mount, linking the host system's /dev/log to the same location (/dev/log) inside the container. This is done to allow the container to access the system's logging facility.
 - **logger Testing logging to the host:** The command to be executed inside the container. The logger command is used to send a test log message, and in this case, it will log the message "Testing logging to the host."



Search for the container event on the container host log, using the following command.

```shell
journalctl | grep "Testing logging"
```

You should see the following:

```shell
May 26 03:28:42 ip-10-0-2-6.ec2.internal root[25499]: Testing logging to the host
```
