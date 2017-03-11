{% capture outro %}{% include_relative getting-started/outro.md %}{% endcapture %}

<block class="centos prebuilt" />

<block class="centos docker" />

<block class="centos compile" />

Check the cloud instructions for a general guideline on building from source for CentOS.

<block class="centos cloud" />

## AWS Cloud Setup

### Amazon Linux AMI with NVIDIA GRID and TESLA GPU Driver

[NVIDIA GRID and TESLA GPU](https://aws.amazon.com/marketplace/pp/B00FYCDDTE?qid=1489162823246&sr=0-1&ref_=srh_res_product_title)

The above AMI had been tested with Caffe2 + GPU support. The installation currently is a little tricky, but we hope over time this can be smoothed out a bit. This AMI is great though because it supports the [latest and greatest features from NVIDIA](http://docs.aws.amazon.com/AWSEC2/latest/UserGuide/accelerated-computing-instances.html).

### Installation Guide

Note that this guide will help you install Caffe2 on any CentOS distro. Amazon uses their own flavor of RHEL and they've installed CUDA in different spots than normally expected so keep that in mind. Some of these steps will not be required on vanilla CentOS because things will go in their normal places.

#### Get your repos set

For whatever reason, many of the required dependencies don't show up in Amazon's enable repositories. Epel is already provided in this image, but the repos is disabled. You need to enable it by editing the repo config to turn it on. Set `enabled=1` in the `epel.repo` file. This enables you to find `cmake3 leveldb-devel lmdb-devel`.

```
sudo vim /etc/yum.repos.d/epel.repo
```

Next you should update yum and install the Caffe2's core dependencies. These differ slightly from Ubuntu due to availability of ready-to-go packages.

```
sudo yum update
sudo yum install -y \
automake \
python-pip \
python-devel \
git \
cmake3 \
libtool \
protobuf-devel \
gcc \
gcc-c++ \
kernel-devel \
leveldb-devel \
snappy-devel \
lmdb-devel
```

glog is not found in yum for this version of Linux, so install from source:

```
git clone https://github.com/google/glog
cd glog
autoreconf -vfi
./configure
make && sudo make install
```

**Troubleshooting `glog` compilation**

`/bin/sh: aclocal-1.14: command not found`

You need to run `autoreconf -vfi`. This is found as part of `automake`. `autoreconf` also requires `libtool` to process `glog`, so these are now part of the install prerequisites above.

**Resuming the regular install**

Install `gflags` now; this [conflicts](https://github.com/google/glog/issues/140) with `glog`, so you must do it in this order:

```
sudo yum install gflags-devel
```

#### Python Dependencies

Now we need the Python dependencies. Note the troubleshooting info below... for some reason the install path with Python can get difficult.

```
sudo pip install \
flask \
graphviz \
hypothesis \
jupyter \
matplotlib \
numpy \
protobuf \
pydot \
python-nvd3 \
pyyaml \
scikit-image \
scipy \
setuptools \
tornado
```

This may fail with error:
`pkg_resources.DistributionNotFound: pip==7.1.0`

To fix this, upgrade pip then update the pip's config to match the version it upgraded to. For example, in the `/usr/bin/pip` file, you need to change two instances of `7.1.0` to `9.0.1`

```
sudo easy_install --upgrade pip
sudo vim /usr/bin/pip
```

Once you've fixed the config file re-run the `sudo pip install...` command from above.

#### Setup CUDA

This image doesn't come with cuDNN, however Caffe2 requires it. Here we're downloading the files, extracting them, and copying them into existing folders where CUDA is currently installed.

```
wget http://developer.download.nvidia.com/compute/redist/cudnn/v5.1/cudnn-7.5-linux-x64-v5.1.tgz
tar xfvz cudnn-7.5-linux-x64-v5.1.tgz
sudo rsync -av cuda /opt/nvidia/
```

Now you need to setup some environment variables for the build step.

```
export CUDA_HOME=/opt/nvidia/cuda
export LD_LIBRARY_PATY=/opt/nvidia/cuda/lib64:/usr/local/bin
```

Whew! So close. Now you need to clone Caffe2 repo and build it:

```
git clone --recursive https://github.com/caffe2/caffe2
cd caffe2
make
cd build
make install
```

#### Test it out!

To check if Caffe2 is working and it's using the GPU's try the command below. You should see an output of a bunch of arrays, but more importantly, you should see no error messages! Consult the Troubleshooting section of the docs here and for Ubuntu for some help.

```
python -m caffe2.python.operator_test.relu_op_test
```

**Test CUDA**

Here are a series of commands and sample outputs that you can try. These will verify that the GPU's are accessible.

```
$ cat /proc/driver/nvidia/version
NVRM version: NVIDIA UNIX x86_64 Kernel Module 352.99 Mon Jul 4 23:52:14 PDT 2016
GCC version: gcc version 4.8.3 20140911 (Red Hat 4.8.3-9) (GCC)

$ nvcc -V
nvcc: NVIDIA (R) Cuda compiler driver
Copyright (c) 2005-2015 NVIDIA Corporation
Built on Tue_Aug_11_14:27:32_CDT_2015
Cuda compilation tools, release 7.5, V7.5.17

$ nvidia-smi -q | head

==============NVSMI LOG==============

Timestamp : Fri Mar 10 23:15:45 2017
Driver Version : 352.99

Attached GPUs : 1
GPU 0000:00:03.0
Product Name : GRID K520
Product Brand : Grid
```

That's it. You've successfully built Caffe2!

### Setting Up Tutorials & Jupyter Server

If you're running this all on a cloud computer, you probably won't have a UI or way to view the IPython notebooks by default. Typically, you would launch them locally with `ipython notebook` and you would see a localhost:8888 webpage pop up with the directory of notebooks running. The following example will show you how to launch the Jupyter server and connect to remotely via an SSH tunnel.

First configure your cloud server to accept port 8889, or whatever you want, but change the port in the following commands. On AWS you accomplish this by adding a rule to your server's security group allowing a TCP inbound on port 8889. Otherwise you would adjust iptables for this.

![security group screenshot](../static/images/security-group-jupyter.png)

Next you launch the Juypter server.

```
jupyter notebook --no-browser --port=8889
```

Then create the SSH tunnel. This will pass the cloud server's Jupyter instance to your localhost 8888 port for you to use locally. The example below is templated after how you would connect AWS, where `your-public-cert.pem` is your own public certificate and `ec2-user@super-rad-GPU-instance.compute-1.amazonaws.com` is your login to your cloud server. You can easily grab this on AWS by going to Instances > Connect and copy the part after `ssh` and swap that out in the command below.

```
ssh -N -f -L localhost:8888:localhost:8889 -i "your-public-cert.pem" ec2-user@super-rad-GPU-instance.compute-1.amazonaws.com
```

#### Troubleshooting

caffe2.python not found | You may have some PATH or PYTHONPATH issues. Add `/home/ec2-user/caffe2/build` to your path and that can take care of those problems.

{{ outro | markdownify }}