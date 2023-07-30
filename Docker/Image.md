A Docker image is a lightweight, standalone, executable package that contains everything needed to run a program inside container, including code, runtime, libraries, environment variables, and configuration files.

Docker image is a read-only template or blueprint that contains a set of instructions for creating containers.

We can create multiple containers from the same image.

![[container_images.png|450]]

Images can be stored on your local machine, and you can share them with others.

#### Image Tags

Images can have different versions, called ==tags==. Tag lets you create multiple versions of the same image.

>Perhaps you’re developing an app, and you want to publish a different container image for each minor release of your app – e.g. **v1.0, v1.1, v1.2,** etc. When you’re working on a new feature of an app, and you want to share a test version, so you want to publish an image labelled ==‘test’==. You can do that with tags.

## Anatomy of a Docker Image

A Docker image is made up of a collection of files that bundle together all the essentials – such as installations, application code, and dependencies – required to configure a fully operational container environment.

## Building an image

To build a container image, you can use few different ways:

-   **Create an image manually:**
	By running a container from an existing Docker image, make some changes to it, and then commit the changes into a new image.

-   **Write a _Dockerfile_:**
	Write a set of instructions in a plain-text file that tells Docker how to build an image, with the `docker build` command.

-   **Command-line tools:** You can use a dedicated image-building tool, like [Bazel](https://bazel.build/) by Google, [Buildah](https://buildah.io/) or [Pack](https://github.com/buildpacks/pack).



##  Docker Image Layers

Docker images consists of multiple read-only layers. Each layer is built on top of another layer to form a series of intermediate images.

Each layer of a Docker image is viewable under `/var/lib/docker/aufs/diff`, or via the Docker history command in the command-line interface (CLI).

