---
title: A Very Brief Introduction to Docker
date: 2017-12-30
---

I prepared this short introduction to Docker for my research group a few months ago. I decided to turn it into a blog post since I have already gone through the effort of writing it, so I might as well share it with the world!

**Disclaimer:** Some bits in this post are copied directly from other onlinr sources. It is not my intention to plagiarise anyone else's work. 

## What is Docker?
### From the Website:
Docker is the world’s leading software container platform. Developers use Docker to eliminate “works on my machine” problems when collaborating on code with co-workers. Operators use Docker to run and manage apps side-by-side in isolated containers to get better compute density. Enterprises use Docker to build agile software delivery pipelines to ship new features faster, more securely and with confidence for both Linux and Windows Server apps.

### Translation:
Docker provides the ability to package code and dependencies into a single "Container".

### What is a Container?
A container packages an entire runtime envirmoment, including code, libraries and dependencies bundled in one package.

### Wait, this sounds like a Virtual Machine!
Except it isn't! A VM bundles an entire OS; whereas a container only bundles the libraries and settings required to run a specific piece of code.

![](http://content.serverspace.co.uk/hubfs/Blog_Images/container-vs-vm.jpg)

## Why should I care about all this?
### 1. To avoid unncessary headeaches
For example, sometimes some libraries in R or Python do not play well with each other. Other times some libaries have weird dependencies that may break when you update other libraries or your OS, causing much frustration.

### 2. For the sake of reproducibility
When you do an analysis using a piece of code in R or Python that depends on a specific version of some library, you want your analysis and its results to still be valid at a later point in time. This is not trivial if the library you are using is updated or changes API (looking at you Tensorflow!).

## A very short tutorial
### Building containers
Containers are created from images. Images are standalone lightweight executable packages that include everything needed to run the software.

Images are defined by a `Dockerfile`, which looks like this:
```dockerfile
FROM debian:8.5

MAINTAINER Kamil Kwiek <kamil.kwiek@continuum.io>

ENV LANG=C.UTF-8 LC_ALL=C.UTF-8

RUN apt-get update --fix-missing && apt-get install -y wget bzip2 ca-certificates \
    libglib2.0-0 libxext6 libsm6 libxrender1 \
    git mercurial subversion

RUN echo 'export PATH=/opt/conda/bin:$PATH' > /etc/profile.d/conda.sh && \
    wget --quiet https://repo.continuum.io/archive/Anaconda3-4.3.1-Linux-x86_64.sh -O ~/anaconda.sh && \
    /bin/bash ~/anaconda.sh -b -p /opt/conda && \ rm ~/anaconda.sh

RUN apt-get install -y curl grep sed dpkg && \
    TINI_VERSION=`curl https://github.com/krallin/tini/releases/latest | grep -o "/v.*\"" | sed 's:^..\(.*\).$:\1:'` && \
    curl -L "https://github.com/krallin/tini/releases/download/v${TINI_VERSION}/tini_${TINI_VERSION}.deb" > tini.deb && \
    dpkg -i tini.deb && \ rm tini.deb && \
    apt-get clean

ENV PATH /opt/conda/bin:$PATH

ENTRYPOINT [ "/usr/bin/tini", "--" ]

CMD [ "/bin/bash" ]
```

You can build the image using `docker build`, and run an already built image using `docker run`.

### Example: Anaconda Python Environment for Data Science
Download the Anaconda image from Docker Hub:
```bash
docker pull continuumio/anaconda3
```

Run the image:
```bash
docker run -i -t continuumio/anaconda3 /bin/bash
```

For Jupyter notebook run the following:
```bash
docker run -i -t -p 8888:8888 continuumio/anaconda3 /bin/bash -c "/opt/conda/bin/conda install jupyter -y --quiet && mkdir /opt/notebooks && /opt/conda/bin/jupyter notebook --notebook-dir=/opt/notebooks --ip='*' --port=8888 --no-browser"
```

There is one caveat though. This container does not have access to the files on my machine. Nor does it have persistence. The former can be solved by mounting a specific directory on my machine to a mount point in the container.

```bash
docker run -v ~/Documents/Learning/Docker/testdir:/home/testdir -i -t continuumio/anaconda3 /bin/bash
```

For Jupyter notebook:
```bash
docker run -v ~/Documents/Learning/Docker/testdir:/home/notebooks/testdir -i -t -p 8888:8888 continuumio/anaconda3 /bin/bash -c "/opt/conda/bin/conda install jupyter -y --quiet && /opt/conda/bin/jupyter notebook --notebook-dir=/home/notebooks --ip='*' --port=8888 --no-browser"
```

## More Info?
This is just the tip of the iceberg. There is a lot more to learn about Docker. For more info, check https://docs.docker.com/
