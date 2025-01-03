# Containerization 

When installing software, you may come across applications that have complex chains of dependencies that are challenging to compile and install. Some software may require very specific versions of libraries that may not be available on CCR's systems or conflict with libraries needed for other applications. You may also need to move between several workstations or HPC platforms, which often requires reinstalling your software on each system. Containers are a good way to tackle all of these issues and more.

## Containerization Fundamentals

Containers build upon an idea that has long existed within computing: hardware can be emulated through software. **Virtualization** simulates some or all components of computation through a software application. Virtual machines use this concept to generate an entire operating system as an application on a host system. Containers follow the same idea, but at a much smaller scale and contained within a system's kernel.

**Containers** are portable compartmentalizations of an operating system, software, libraries, data, and/or workflows. Containers offer portability and reproducibility. 

- Portability: containers can run on any system equipped with its specified container manager.
- Reproducibility: because containers are instances of prebuilt isolated software, software will always execute the same every time.

Containers distinguish themselves through their low computational overhead and their ability to utilize all of a host system’s resources. Building containers is a relatively simple process that starts with a container engine.

## Container engines

[Docker](https://www.docker.com/) is the most widely used container engine, and  can be used on any system where you have administrative privileges. _Docker cannot be run on high-performance computing (HPC) platforms because users do not have administrative privileges._

[Apptainer](https://apptainer.org/) (formerly Singularity) is a container engine that does not require administrative privileges to execute. Therefore, it is safe to run on HPC platforms.    

Because Docker images are widely available for many software packages, a common use case is to use Apptainer to run Docker images.

## Apptainer

Apptainer is a containerization software package that does not require users to have administrative privileges when running containers, and can thus be safely used on Research Computing resources. Apptainer is functionally the same as Singularity and the commands can be used interchangeably at this time.  

Much like Docker, Apptainer is a containerization software designed around compartmentalization of applications, libraries, and workflows. This is done through the creation of compressed images in the `.sif` format which can be run as ephemeral containers. Unlike Docker, however, Apptainer does not manage images, containers, or volumes through a central application. Instead, Apptainer generates saved image files that can either be mutable or immutable based on compression.

### Setting up Paths
Images generated by Apptainer can be _large_. It is common for these containers to exceed 10GB. This means you can easily exceed your 25GB home directory quota while pulling a single container! You may also see error messages when pulling containers from remote repositories mentioning how `/tmp` does not have write permissions. All of these problems and more can be avoided by setting a few environmental variables before pulling Apptainer containers.  Below is an example of how to set the environment variables.  You'll need to set the path name to your group's shared project directory, which may not be in `/projects/academic`, and make sure these directories are created:

```
export APPTAINER_LOCALCACHEDIR=/projects/academic/[YourGroupName]/[CCRusername]/tmp
export APPTAINER_CACHEDIR=/projects/academic/[YourGroupName]/[CCRusername]/tmp
export APPTAINER_TMPDIR=/projects/academic/[YourGroupName]/[CCRusername]/tmp
```

This will also ensure all large intermediate files are not being stored in your home directory leading to quota issues.

### Pulling Images

Because we cannot build our own Apptainer images on HPC systems, we must instead bring our images over from another location. Pulling images from public repositories is often the easiest method of using a containerized application. 

We can use the `apptainer pull` command to remotely download our chosen image file and convert it to the Apptainer `.sif` format. The command requires the container registry we would like to use, followed by the repository’s name:

```
apptainer pull <localname>.sif <container-registry>://<repository-name>
```

Where `<localname>.sif` is the name you choose for the Apptainer image. 

A container registry is simply a server that manages uploaded containers. Docker Hub is the most widely used register. To pull a container image from Docker Hub:

```
apptainer pull docker://another:example
```

### Running a SIF image as a container

SIF images can be run as containers much like Docker images. Apptainer commands, however, follow a bit more nuanced syntax depending on what you’d like to do. After pulling your image from Docker Hub you can run the image by using the `apptainer run` command. Type:

```
apptainer run <image-name>
```

Running a container will execute the default program that the container developer will have specified in container definition file. To execute specific programs in your container, we can use the `apptainer exec` command, and then specify the program:

```
apptainer exec <image-name> <program>
```

Much like specifying an application in Docker, this will allow a user to execute any program that is installed within your container. Unlike Docker however, you do not need to specify a shell application to shell into the container. We can simply use the `apptainer shell` command:

```
apptainer shell <image-name>
```

*Example:*

Say we have an image that contains python 3.7 as the default software, and we want to run python from the container. We can do this with the command:

```
apptainer run python-cont.sif
```

If the default application for the image is not python we could run python as follows:

```
apptainer exec python-cont.sif python
```

### File Access

By default, only `/user/$USER` is available within any given container. This means that a user will need to bind any other required folders to the container’s directory tree. Furthermore, a container will also have access to the files in the same folder where it was initialized (`$PWD`). 

To bind any additional folders or files to your container, you can utilize the `-B` flag in your Apptainer run, exec, and shell commands:

```
apptainer run -B /source/directory:/target/directory sample-image.sif
```

Additionally you can bind directories by utilizing the `APPTAINER_BINDPATH` environment variable. Simply export a list of directory pairs you would like to bind to the your container:

```
export APPTAINER_BINDPATH=/source/directory1:/target/directory1,\
/source/directory2:/target/directory2
```

Then run, execute, or shell into the container as normal.

### Building a SIF image

In the event that a container is unavailable for a given application, you may need to build your own container from scratch. You will need to download and install Apptainer on your own machine. Instructions to do this can be found in the [documentation for Apptainer](https://apptainer.org/docs/admin/main/installation.html).

Apptainer allows a user to build images using a *definition file*. Just like a Dockerfile, this file has a variety of directives that allow for the customization of your image. A sample image would look something like this: 

```
Bootstrap: docker
From: ubuntu:20.04

%help
	I am help text!

%setup		
	apt-get update
	apt-get install nano
	apt-get install gcc 

%runscript
	echo “hello! I am a container!”
```

#### Apptainer Build

Once you have written your Apptainer definition file, you can build the application locally with the `apptainer build` command, as follows:

```
apptainer build <localname>.sif <recipe-name>.def
```

### GPU-enabled Containers 

It is possible to run GPU workloads within Apptainer containers. This will allow you to run more up-to-date versions of software like PyTorch which may have older versions installed as modules. To do so, merely add the `--nv` flag when you use `run` or `exec` commands like so:

`apptainer run --nv <image_name>.sif`

NVIDIA also hosts a [number of containers](https://catalog.ngc.nvidia.com/containers) as part of their own container library. These can be pulled and run by Apptainer, though the pulling process can take several hours depending on the image size.  We recommend you do this on a compile node or a compute node in a job.  If run from a login node, your download will get cancelled if it goes beyond our time out limits.  See more on node types [here](../hpc/clusters.md)

### Building MPI-enabled images
MPI-enabled Apptainer containers can be deployed on CCR's systems with the caveat that the MPI software within the container may have a similar (not necessarily exact) version of the MPI software available on the system. This requirement diminishes the portability of MPI-enabled containers, as they may not run on other systems without compatible MPI software. Regardless, MPI-enabled containers can still be a very useful option in many cases. 

Here we provide an example of using a gcc compiler with OpenMPI. CCR's system uses an Infiniband interconnect. In order to use an Apptainer container with OpenMPI (or any MPI) on the cluster, OpenMPI needs to be installed both inside and outside of the container. More specifically, the _same_ version of OpenMPI needs to be installed inside and outside (at least very similar, you can sometimes get away with two different minor versions, e.g. 2.1 and 2.0). 


Once you’ve built the container with one of the methods outlined above, you can place it in your home or project directory and run it on a compute node. The following is an example of running a gcc/OpenMPI container with Apptainer. The syntax is a normal MPI run where multiple instances of an Apptainer image are run. The following example runs `mpi_hello_world` with MPI from a container.

```
module load gcc/11.2.0
module load openmpi/4.1.1

mpirun -np 4 apptainer exec openmpi.sif mpi_hello_world"
```
