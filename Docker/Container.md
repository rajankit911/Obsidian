- A container is a virtual environment for running a program.
- It is a lightweight, portable, self-contained and isolated execution environment.
- Docker image becomes container when it runs on Docker Engine.
- Docker container is running instance of a Docker 

![[docker_container.png|500]]
### Features of Docker Containers

-   **Standard:**
	Docker created the industry standard for containers, so they could be portable anywhere

-   **Lightweight:**
	Containers share the machine’s OS system kernel and therefore do not require an OS per application

-   **Isolation:**
	 A container lets you isolate a process (and its files) on an operating system. Containers are ==“process-level”== isolation. Typically, when a container starts, the process inside the container also starts. And when the process ends, the container also stops.

-   **Secure:**
	Applications are safer in containers and Docker provides the strongest default isolation capabilities in the industry

Each Docker container runs in its own isolated environment, with its own filesystem, networking, and resources, but shares the same kernel as the host operating system. This makes Docker containers more efficient than traditional virtual machines, as they do not require a separate operating system for each container.

Docker containers are widely used for deploying and scaling applications in various environments, including development, testing, staging, and production. They provide a consistent and predictable runtime environment, which makes it easier for developers to build, test, and deploy their applications.

### A container puts a program in jail

When an app is running inside a container, it gets isolated and quarantined from the rest of the system. Container restricts its access to the files and resources on the host computer.

>**Creating a container is a bit like creating a sandbox for an application.**

>[!faq]- What is a sandbox?
>A sandbox is a technical term that means separating a program from others on a server. When you sandbox a program, you usually do it to protect other programs or data on the system, in case something goes wrong.


A containerised process can only do a limited amount of stuff:

-   It can only see a certain set of files
-   It can’t see other processes running on the same host
-   It can be allocated a restricted amount of memory or CPU
-   Its network ports aren’t exposed outside the container

>[!faq]- How does the isolation work?
>The container runtime (e.g. Docker) uses some advanced Linux features – like ==**cgroups**== (which controls access to resources), ==**namespaces**== (which controls the visibility of resources) and ==**chroot**== (which controls access to the filesystem) – to restrict what a process can see and do.


## Why should a container have only one process?

Container-based application design encourages the principle of **one container, one process**. That is to say, a Docker container should have just one program running inside it.

>Docker is efficient at creating and starting containers. It allocates PID (Process ID) 1 to the process running inside the container.

There are various benefits to running one process per container:

-   **Isolation:**
	By placing each process in a separate container, you gain the benefits of isolating the process so that it can’t interfere with others.

-   **Easier to scale:** 
	When a container consists of just one single process, it is easier to scale the application by creating more instances of the container.
	If an application consists of a web server and a database, it is better to run two containers – one for the web server and one for the database – so that they can be scaled independently. For example, if the web server is serving a lot of traffic, it can be scaled separately from the database.

-   **Easier to build and test:**
	When processes are isolated, it makes it easier to build container images because there is less work to do. Developers can build container images without being impacted by others who may be working on another process inside the same container.

-   **Components can be upgraded independently:**
	If you separate your application components into different containers, you can maintain and patch them separately. For example, if a new version of your web server is released, you can deploy the latest version, without impacting processes running in other containers.

-   **Better reusability:**
	One of the benefits of a container-based application is that it can be run for different purposes and in different environments, by just changing its configuration. This makes a container like a building block. However when a container has more than one process, it is significantly more specialised and the potential for reuse is limited.

-   **Easier to collect logs:**
	If there is a single foreground process running in a container, it is easier to identify logs that are sent to _standard out_. If there are multiple processes running, you will either do some work to identify the logs from the different processes in _standard out_, or you will need to redirect the logs to separate files, neither of which is a nice solution.

-   **Simpler to manage with Docker:**
	Docker watches your application’s process (PID 1), and uses this to report the event that your container has stopped. So it knows if your application has terminated nicely, or if it has crashed horrifically! If multiple processes are running inside the container, this becomes harder, and you might need to monitor your processes manually.


## Can you run multiple processes in a container?

For an application that consists of two very tightly-coupled processes, we may need to run multiple processes inside the same container. You can do this using a couple of different approaches:

-   **Use a process manager which can run multiple processes:**
	You can set the container’s _entrypoint_ to a specialised program which is capable of running and managing multiple processes.
	
    You can use _supervisord_ as your container _entrypoint_, which will then load the services that you need.

-   **Write your own startup or wrapper script:**
	You can write your own startup script which starts multiple processes, but then you will need to handle things like _shutdown signals_ and _zombie processes_ appropriately, and clean them up when they finish.



## Docker Container Lifecycle

Docker container goes through various stages of life

![[docker_container_lifecycle.png]]

A container is simply an isolated process running on your computer. A Docker container can be in one of several core states:

-   **Created** - The Docker container has been created, but not started (e.g. after using `docker create`)

-   **Up** - The Docker container is currently running. That is, the process inside the container is running. Can happen using `docker start` or `docker run`.

-   **Paused** - The Docker container has been paused, usually with the command `docker pause`.

-   **Exited** - The Docker container has exited, usually because the process inside the container has exited.


## Why Does My Docker Container Stop?

There are several reasons why a Docker container might end:
-   **The main process inside the container has ended successfully:**
	When the process running inside your container ends, the container will exit. You run a container, which runs a shell script to perform some tasks. When the shell script completes, the container will exit, because there’s nothing left for the container to run.

- **The process inside the container has been terminated:**
	When the program running inside the container is given a signal to shut down by pressing _Ctrl+C_. When this happens, the program will stop, and the container will exit.

- **You’re running a shell in a container, but you haven’t assigned a terminal:**
	When you’re running a container with a shell (like ==bash==) as the default command, if you haven’t attached an interactive terminal, then your shell process will exit, and so the container will exit.

```sh
docker run -it (or --interactive --tty) <image_id>
```

-   **Stop the container:**
	A container can be manually stopped by using the `docker stop` command.

- **The Docker daemon has restarted the container:**
	Docker can restart containers if you need it to by using `docker restart`. By default, Docker doesn’t automatically restart containers when they exit, or when Docker itself restarts.
	
	To configure Docker to restart containers automatically, use a [restart policy](https://docs.docker.com/config/containers/start-containers-automatically/) using the `--restart` switch, when you run a container using `docker run`.

- **The container has consumed too much memory, and has been killed by the host OS:**
	You can set hard memory limits on the container by using the `-m` switch with `docker run`. If a container is running out of memory, it might be killed by the OS.


## How to find out why a Docker container exits

### 1. Look at the logs
The best way to explore what goes on inside a Docker container is to look at the logs using `docker logs <container_id>`:
-   Do the logs show the application starting up properly?
-   Can you see any helpful debug messages from the application?
-   Does the application crash with a stack trace (e.g. Java) or some other debugging information?

### 2. Check the state of the container
You can view details about a container by using `docker inspect <container_id>`.

```json
"State": {
    "Status": "exited",
    "Running": false,
    "Paused": false,
    "Restarting": false,
    "OOMKilled": false,
    "Dead": false,
    "Pid": 0,
    "ExitCode": 0,
    "Error": "",
    "StartedAt": "2020-10-30T11:44:56.344308725Z",
    "FinishedAt": "2020-10-30T11:48:23.434250526Z",
    "Healthcheck": {
        "Status": "",
        "FailingStreak": 0,
        "Log": null
    }
}
```

The _State_ includes information like the container’s _exit code_ and perhaps the error message that was returned. An _ExitCode_ of `0` generally means that the application successfully completed. But any other exit code indicates an unsuccessful exit.


## What if my Docker container dies immediately?

If you run a container and it terminates immediately, you can debug this situation and find out why the container is stopping. Create and start a container from the same failing image, and override the ==entrypoint== with a shell, like `sh` or `bash`. This will start the container, and drop you in to a shell session, where you can run your script, and see whether it is exiting unexpectedly.

```sh
docker run -it --entrypoint /bin/sh nginx:latest
```

## How to prevent a container from stopping

There are a few ways that you can prevent a container from stopping and therefore, keep a process alive:

-   **Run a long-lived service inside the container:**
	Most containers run a long-living service, like a database or a web server. These processes keep running until they receive a shutdown signal.

-   **Add an artificial sleep or pause to the entrypoint:**
	If your container is running a short-lived process, the container will stop when it completes. You can artificially extend this by adding an artificial sleep or pause to the entrypoint script. For example, in _bash_, you can create an infinite pause: `while true; do sleep 1; done`.

-   **Change the entrypoint to an interactive shell:**
	If you override the _entrypoint_ to the container with a shell, like `sh` or `bash`, and run the container with the `-itd` switches, Docker will run the container, detach it (run it in the background), and attach an interactive terminal. Note that this won’t run the container’s default entrypoint script, but run a shell instead. Effectively, you will have a container that stays running permanently. For example: `docker run --entrypoint /bin/sh -itd mycontainer:latest`. This is useful if you want to use a container like a virtual machine, and keep it running in the background when you’re not using it. 

