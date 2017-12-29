---
title: How to Setup a VM in Azure for Deep Learning?
date: 2017-01-02
---

I am quite interested in learning more about deep learning, but I find it quite difficult to implement some of the recent models on my laptop, due to their huge computational overhead on the CPU. Since, I do not have £1000 to spend on a new Titan X, I decided to try a cheaper solution on Microsoft's cloud service Azure. If you are in a similar position to me, you might find this blog post helpful as I detail the setup of the cloud solution I used.

In this blog post, I am going to explain how to set up a **virtual machine (VM) in Azure with GPU support**, to use for deep learning. I am going to walk through the installation processes of **CUDA** and **CuDNN** that are required to enable computation on the GPU. Finally, I am going to explain how to install and configure **Theano** and **Keras** for use on the GPU.

## Step 1: Deploying a VM with GPU Support on Microsoft Azure
First, we will start off by creating a virtual machine (VM) in Azure with GPU support. Microsoft has recently commissioned the Azure N-Series Virtual Machines powered by NVIDIA Tesla GPUs. These come in two flavours: the NC-Series VMs making use of NVIDIA Tesla K80 GPUs and the NV-Series VMs making use of NVIDIA Tesla M60 GPUs. The NC-Series is more suitable for computational tasks like deep learning, while the NV-Series is geared towards visualisation tasks. More on this [here](https://azure.microsoft.com/en-gb/blog/azure-n-series-preview-availability/).

For deep learning, we will make use of the NC-Series VMs. The NC-Series VMs come in three sizes NC6, NC12 and NC24. For the purpose of this blog post, we will use an NC6 instance as it is the cheapest option that contains all the specifications that we need. At the time of writing this post, NC-Series VMs are only available in the US South Central region. Of course, they can be deployed from anywhere else in the world as long as you don't mind an extra bit of latency. 

I will not go through all the details of spinning up a VM on Azure as there is an excellent blog post on that [here](https://docs.microsoft.com/en-us/azure/virtual-machines/virtual-machines-linux-quick-create-portal). However, I will give the specifications of the VM that are needed in order follow the rest of this blog post:

1. When selecting the operating system of the VM, make sure you choose **Ubuntu Server 16.04 LTS**.
2. When selecting the **VM Disk Type**, make sure you select **HDD**, as NC-Series instances are only available for these at the moment.
3. When selecting the **location** make sure you select **South Central US**.
4. When selecting the VM **size**, click on "View all" and select **NC6 Standard**.

![](https://i.imgur.com/fHjCsPG.png)

## Step 2: Installing CUDA Toolkit 8.0
Having setup a VM with GPU support on Azure, we now need to install the CUDA Toolkit to enable using the GPU for computational tasks (More on CUDA [here](http://www.nvidia.com/object/cuda_home_new.html)). We start off by logging into the new VM using `ssh` (for Linux or MacOS users) or `PuTTY` (for Windows users). Once inside the VM, we install some useful packages and make sure that all our packages are up-to-date.

```bash
#Update software repository
sudo apt-get update
#Download useful packages
sudo apt-get install -y libatlas-base-dev libopencv-dev libprotoc-dev python-numpy python-scipy make unzip git gcc g++ libcurl4-openssl-dev libssl-dev
#Upgrade all remaining packages
sudo apt-get upgrade
```

Next, we download the CUDA Toolkit 8.0 installer from the NVIDIA website.

```bash
#Navigate to the home directory
cd ~
#Create a Downloads directory
mkdir Downloads
#Navigate to the Downloads directory
cd Downloads
#Download the deb package from the NVIDIA website
wget "https://developer.nvidia.com/compute/cuda/8.0/prod/local_installers/cuda-repo-ubuntu1604-8-0-local_8.0.44-1_amd64-deb"
#Rename the downloaded file to end with a .deb extension
mv cuda-repo-ubuntu1604-8-0-local_8.0.44-1_amd64-deb cuda-repo-ubuntu1604-8-0-local_8.0.44-1_amd64.deb
```

Now we can install the CUDA software package.

```bash
#Install CUDA from deb package
sudo dpkg -i "cuda-repo-ubuntu1604-8-0-local_8.0.44-1_amd64.deb"
#Update software repository
sudo apt-get update
#Install dependencies for CUDA
sudo apt-get install cuda
```

Finally we need to create a couple of environment variables and append our path. To do this, we append the `.bashrc` file in the home directory.

```bash
#Navigate to the home directory
cd ~
#Open .bashrc using vim 
#You can use nano or emacs or any other editor you prefer
vim .bashrc
```

Append the `.bashrc` file with the following:

```bash
#Define CUDA_HOME environment variable
export CUDA_HOME=/usr/local/cuda-8.0 
#Define LD_LIBRARY_PATH environment variable
export LD_LIBRARY_PATH=${CUDA_HOME}/lib64 
#Add CUDA_HOME to PATH
export PATH=${CUDA_HOME}/bin:${PATH}
```

Finally, we need to apply the changes to the `.bashrc` file by executing its contents.

```bash
#Execute bashrc
source .bashrc
```

And voila! Now - fingers crossed - CUDA should be installed.

## Step 3: Installing CuDNN
After installing CUDA, we now need to install the CuDNN (CUDA Deep Neural Networks) library. CuDNN is a GPU-accelerated library of primitives for deep neural networks used in frameworks like Tensorflow and Theano (More information [here](https://developer.nvidia.com/cudnn)). Since CuDNN is a proprietary library, you need to register for the NVIDIA Developer programme to be allowed to download it. Registration is free and all you need is an email address. You can register [here](https://developer.nvidia.com/cudnn). 

Once registered, login and navigate to the previous [link](https://developer.nvidia.com/cudnn). Click on Download and you will be presented with a Software License Agreement. If you agree on that, you will be presented with download links for multiple versions of the library. At the time of writing this post, the latest version was 5.1, which is the one I will use for the remainder of this tutorial (Obviously the one compatible with CUDA 8.0). I suspect that the same installation procedure will work on other versions.

![](https://i.imgur.com/DWdhF3q.png)

After you pick the required version, you will be presented with multiple download links for different platforms. You should choose the generic Linux library i.e. cuDNN vX.X Library for Linux. Download this one to your local machine. **Then you need to copy the downloaded file to the VM from your local machine.** You can do this with `scp` (Linux and MacOS users) or `filezilla` (Windows users). ~~Alternatively, you can download the file directly to your VM, but I was not able to find an easy way to do this!~~

![](https://i.imgur.com/Zmhyl8R.png)

Assuming that the file was copied to the Downloads directory on the VM, we begin by extracting the library from the tar file and moving it into a different repository.

```bash
#Navigate to Downloads
cd ~/Downloads
#Extract the library from the tar file
tar xvzf cudnn-8.0-linux-x64-v5.1.tgz
#Move the library to /usr/local
sudo mv cuda /usr/local/cudnn
```

We then create symbolic links to the header files of the library. We place the links in the CUDA directory.

```bash
#Create symlink to cudnn.h in cuda/include
sudo ln -s /usr/local/cudnn/include/cudnn.h /usr/local/cuda/include/cudnn.h
#Create symlink to shared libraries
sudo ln -s /usr/local/cudnn/lib64/libcudnn* /usr/local/cuda/lib64/
```

And we are done! CuDNN should now be working when called in Theano.

## Step 4: Installing and Configuring Theano
Now that we set up CUDA and CuDNN, we need to install Theano. Theano is a Python deep learning framework that can be used to build, optimise and evaluate computational graphs. Theano makes use of the GPU to significantly speed up computation, which is handy when training deep neural networks. More on Theano [here](http://deeplearning.net/software/theano/).

When using Python for scientific computing or data science, I prefer to use the Anaconda distribution of Python. Using Anaconda Python for these tasks has [many advantages](https://www.reddit.com/r/Python/comments/3t23vv/what_advantages_are_there_of_using_anaconda/) over using the system Python. The Anaconda distribution is accompanied with the `conda` package management system, which makes installing new Python libraries virtually hassle-free, as it is able to deal with dependencies. Hence, I will explain how to install Anaconda on your VM and how to use it to install Theano.

First off, we start by downloading then running the Anaconda installer.

```bash
#Navigate to Downloads
cd ~/Downloads
#Download installer
wget "https://repo.continuum.io/archive/Anaconda3-4.2.0-Linux-x86_64.sh"
#Run installer
sudo bash Anaconda3-4.2.0-Linux-x86_64.sh
#Navigate to the home directory
cd ~
#Execute .bashrc
source .bashrc
```

During the installation process, you will be presented with an option at the end to append you `.bashrc` file to add Anaconda to your path. **You should agree on this**. The above process should install Anaconda; however, since we used `sudo` during the installation, only `root` is allowed to change the contents of the `ancaconda3` directory. This is problematic if we need to use `conda` to install new packages. Hence, we need to change the ownership of the `anaconda3` directory to allow the user i.e. you, to modify its contents. We can do this as follows:

```bash
#Navigate to the home directory
cd ~
#Give [user] ownership of anaconda3
#[user] is your username!
sudo chown -R [user] anaconda3
``` 

This completes the Anaconda installation. Now we can use `conda` to install Theano.

```bash
#Install Theano 
conda install theano
```

`conda` will take care to install and configure all the dependencies required to setup Theano.

Having installed Theano, we need to configure it to allow it to use the GPU to speed up computation. In the home directory, we create a configuration file for Theano `.theanorc` and add some attributes to it.

```bash
#Navigate to home directory
cd ~
#Create and edit theanorc using vim
vim .theanorc
```

Add the following to `.theanorc`.

```
[global]
floatX = float32
device = gpu0

[lib]
cnmem = 1
```

Finally, to check that everything is running correctly, run the following script in Python.

```python
from theano import function, config, shared, sandbox
import theano.tensor as T
import numpy
import time

vlen = 10 * 30 * 768  # 10 x #cores x # threads per core
iters = 1000

rng = numpy.random.RandomState(22)
x = shared(numpy.asarray(rng.rand(vlen), config.floatX))
f = function([], T.exp(x))
print(f.maker.fgraph.toposort())
t0 = time.time()
for i in range(iters):
    r = f()
t1 = time.time()
print("Looping %d times took %f seconds" % (iters, t1 - t0))
print("Result is %s" % (r,))
if numpy.any([isinstance(x.op, T.Elemwise) for x in f.maker.fgraph.toposort()]):
    print('Used the cpu')
else:
    print('Used the gpu')
```

If the script prints "Used the gpu", then you know that CUDA and Theano has been configured correctly.

# Step 4: Installing and Configuring Keras
Keras is a Python library for neural networks that is built on top of Theano or Tensorflow. Keras allows for fast and easy prototyping of various neural network architectures via its modular design. More information about Keras [here](https://keras.io/). Keras is available as part of the Anaconda distribution. All you need to do is installing it using `conda`.

```bash
#Install Keras
conda install keras
```

This will install Keras and all its dependencies. By default, Keras ships with a Tensorflow backend, i.e. it uses Tensorflow to build the computational graphs of the neural networks. Hence, we need to change the backend to Theano as we have already configured it to use the GPU. To do this, we first need to run Keras once to generate its configuration files.

```bash
#Run Keras once
python -c "import keras"
```

Running Keras for the first time generates its configuration files which are located in the `.keras` directory in home. Inside this directory, you will find a file called `keras.json` which contains Keras configurations, among which is the backend of choice.

```bash
#Navigate to .keras directory
cd ~/.keras
#Open keras.json using vim
vim keras.json
```

Change the configurations inside `keras.json` to the following:

```
{
    "image_dim_ordering": "th",
    "epsilon": 1e-07,
    "floatx": "float32",
    "backend": "theano"
}
```

Now whenever Keras is imported, the statement "Using Theano backend" should be printed. This tells us that Keras is using the Theano backend.

## Summary
In this blog post, I demonstrated how you can create a VM in Azure equipped with an NVIDIA Testa K80 GPU. I also explained how you can install the CUDA Toolkit and the CuDNN library on the new VM. Finally, I showed how you can install and configure Theano and Keras using Anaconda.

The above setup for CUDA and CuDNN should work on deep learning frameworks other than Theano. Although, you might need to tinker a bit with their configurations. For instance, I managed to install Tensorflow with GPU support with the same CUDA and CuDNN setup that I used for Theano. If you prefer to use Tensorflow, follow the installation instructions [here](https://www.tensorflow.org/get_started/os_setup#anaconda_installation).
