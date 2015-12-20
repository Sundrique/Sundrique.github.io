---
layout: post
title:  "Running Lasagne via Jupyter on AWS GPU instance"
tags: [python, jupyter, lasagne, theano, gpu, aws, cuda, ubuntu]
image:
  feature: cover.jpg
---
[Lasagne](https://github.com/Lasagne/Lasagne) is a powerful Python library to build and train neural networks in [Theano](https://github.com/Theano/Theano). However, you don't get all the benefits of it unless you have CUDA-capable GPU. Fortunately, we can utilize Amazon Web Services which combines convenience and reasonable prices. We only need to create and configure instance once and run/stop it whenever we need later on.

## Notice

[The process  of running GPU instance on AWS](http://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-launch-instance_linux.html) is out of the scope of this article. It's pretty straightforward and described in many other tutorials.
Nevertheless, there are a couple of caveats I would like to mention:

1. It's assumed that you use Ubuntu 14.04 image. 
2. GPU instances are not available in all regions, so if you cannot find them in the list of instance types, simply try another region.
3. A security group has only SSH rule by default, we will also need an HTTP access (Protocol: TCP/IP, port: 80).

Here goes!

## CUDA

1) Install CUDA along with dependencies.
{% highlight sh %}
$ wget http://developer.download.nvidia.com/compute/cuda/repos/ubuntu1404/x86_64/cuda-repo-ubuntu1404_7.5-18_amd64.deb
$ sudo dpkg -i cuda-repo-ubuntu1404_7.5-18_amd64.deb
$ rm -f cuda-repo-ubuntu1404_7.5-18_amd64.deb
$ sudo apt-get update
$ sudo apt-get install build-essential linux-image-extra-virtual linux-source linux-source linux-headers-generic cuda
{% endhighlight %}

2) Set environment variables.
{% highlight sh %}
$ echo -e "\nPATH=/usr/local/cuda/bin:\$PATH" >> ~/.profile
$ sudo sh -c "echo /usr/local/cuda/lib64 > /etc/ld.so.conf.d/cuda.conf"
$ sudo ldconfig
{% endhighlight %}

3) Reboot and add `nvidia` module to a kernel.
{% highlight sh %}
$ sudo reboot
$ sudo modprobe nvidia
{% endhighlight %}

## Theano + Lasagne

4) Install the latest versions of Theano and Lasagne directly from GitHub.
{% highlight sh %}
$ sudo apt-get install git gfortran libopenblas-dev python-dev python-pip python-nose python-numpy python-scipy
$ sudo pip install --no-deps git+git://github.com/Theano/Theano.git
$ sudo pip install --no-deps git+git://github.com/Lasagne/Lasagne.git
{% endhighlight %}

5) Configure Theano to use GPU by default.
{% highlight sh %}
$ echo -e "[global]\nfloatX = float32\ndevice = gpu" > ~/.theanorc
{% endhighlight %}

We will set `THEANO_FLAGS` environment variable in Upstart service configuration, therefore, this step is optional and only needed if you are willing to run scripts via command line.

## Jupyter

6) Install Jupyter
{% highlight sh %}
$ sudo pip install jupyter
{% endhighlight %}

7) and create Upstart configuration in order to start Jupyter on boot or start/stop it via command line when it's needed.
{% highlight sh %}
$ mkdir /home/ubuntu/notebooks
$ cd /etc/init/
$ sudo wget https://gist.githubusercontent.com/Sundrique/6e55772def21f4aa4369/raw/4e92708401df337887c000fde97ea693d5e22050/jupyter-notebook.conf
$ sudo start jupyter-notebook
{% endhighlight %}

## Done

8) You can verify your installation by downloading IPython notebook with Lasagne MNIST example and running it in a browser.
{% highlight sh %}
$ cd /home/ubuntu/notebooks
$ wget https://gist.githubusercontent.com/Sundrique/5e401c96b5fde428d268/raw/61267f7abaa40b06e9e64374c1b33978a4584556/MNIST.ipynb
{% endhighlight %}

9) Open your instance's public DNS in a browser and you should be able to see Jupyter interface there.

10) Don't forget to stop instance once you don't need it anymore.