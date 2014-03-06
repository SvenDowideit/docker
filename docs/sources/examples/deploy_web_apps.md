# Deploying Web Applications Using Docker
----------------------------------------------------------------------

## Introduction And Contents
----------------------------------------------------------------------

### Introduction
----------------------------------------------------------------------

Web applications come in many different flavours, usually mixed and
matched with various other elements such as databases. Distinct from
the programming language used or the application size, the deployment
process can be pretty straight forward with the right guidance. When
the entire *application lifecycle* (ALM) considered, however, the
whole thing can easily turn into bit of a mess.

In this example, we are going to see how to use Docker as a powerful 
platform for deploying web applications. We will do this not by 
remodelling or introducing a drastic change, but rather rethinking 
and using *containers* as reliable, platform-agnostic and portable 
building-blocks.

**By the end of this example, you will know how to:**

 - Deploy and manage web-applications using containers.
 - Automate the process using *Dockerfiles*. 

## Understanding The Challenge
----------------------------------------------------------------------

Going online, despite holding an enormous value, is only a single aspect 
of continuous creation. Application lifecycle management (ALM) consists
of various other steps. In reality, rarely do these take place on the
same system, or even the architecture. 

By levelling the differences using containers, a lot of headaches can be 
waved goodbye. From a single page "Hello world!" website to projects 
welcoming hundreds of millions of clients, standardising *your* ALM will 
translate to leverage: reduced times, lowered costs, simpler integration 
and *much more*.

## Introducing Docker Into The Challenge
----------------------------------------------------------------------

Docker is an application agnostic technology. Tools it offers simplify 
many of deployment and management related challenges for web applications. 

Docker achieves this by using *containers*.

Containers provide an isolated, highly portable and manageable platform 
for deploying and running any application, including web applications.

As explained in the
[Introduction articles](/introduction/home),
containers offer a way of keeping everything related to a project in 
one place on top of a base operating-system disk image. By taking 
advantage of this technology, web application deployment can be 
simplified in the following ways:

1. **Simple automation:**  
   Docker containers can be built automatically to your application's 
   specification.
2. **Simple deployments:**  
   Containers are highly portable. They only require system 
   administrators to push them to their servers to get up and 
   running — or scale across any number of machines.
3. **Easy scaling:**  
   Using containers, duplicating instances running your application 
   can be increased (or decreased) by executing a single command for 
   each operation respectively.
4. **Enhanced security:**  
   Containers, and applications deployed inside, can be considered 
   like powerful sandboxes. They do not have access to the outside,
   and the affects of any command executed stays within.

## Building a Web Application Container
----------------------------------------------------------------------

In this example, we will see how to prepare and use a `Dockerfile` to 
create Docker images for different platforms and languages. Using 
these images, we will see how to launch container instances running a 
web application.

> **Note:** Please make sure that commands executed via `RUN` do not 
> prompt messages or bring up interactive installation screens (i.e. 
> dialogs). 

### Hosting Python on Ubuntu
----------------------------------------------------------------------

    ##################################################
    # Dockerfile sample for:                         #
    # Python Web Applications on Ubuntu              #
    ##################################################

    FROM ubuntu:latest
    MAINTAINER O.S. Tezer, sonat@docker.com

    # Install basic tools on Ubuntu for web app. deployments:
    RUN apt-get update
    RUN apt-get install -y -q git mercurial
    RUN apt-get install -y -q tar curl nano wget
    RUN apt-get install -y -q libevent-dev build-essential
    
    # Install Python tools:
    RUN apt-get install -y -q python python-dev
    RUN apt-get install -y -q python-pip python-distribute
    
    # Install Gunicorn web application server:
    RUN pip install gunicorn
    
    # Get your web application source using Git:
    RUN git clone https://github.com/shykes/helloflask.git
    ## Or, by using ADD:
    # ADD /helloflask /helloflask
    
    # Install the requirements:
    RUN pip install -r /helloflask/requirements.txt
    
    # Set the base directory of your application:
    WORKDIR /helloflask
    
    # Set the port to be exposed:
    EXPOSE 8080
    
    # Set the command and arguments to execute upon launch: 
    CMD gunicorn -b 0.0.0.0:8080 app:app

### Hosting Ruby (Rails) on CentOS
----------------------------------------------------------------------

    ##################################################
    # Dockerfile sample for:                         #
    # Ruby (Rails) Web Applications on CentOS        #
    ##################################################

    FROM centos:latest
    MAINTAINER O.S. Tezer, sonat@docker.com

    # Enable EPEL:
    RUN su -c 'rpm -Uvh http://dl.fedoraproject.org/pub/epel/6/x86_64/epel-release-6-8.noarch.rpm'
    
    # Update YUM:
    RUN yum update

    # Install basic tools on CentOS for web app. deployments:
    RUN yum install -y git mercurial
    RUN yum install -y nano wget dialog curl-devel
    RUN yum install -y which libevent-devel
    RUN yum groupinstall -y 'development tools'
    
    # Important packages:
    RUN yum install -y zlib-dev bzip2-devel openssl-devel sqlite-devel
    
    # Install Ruby:
    RUN ln -sf /proc/self/fd /dev/fd
    RUN curl -L get.rvm.io | bash -s stable
    RUN source /etc/profile.d/rvm.sh
    RUN rvm reload
    RUN rvm install 2.1.0
    
    # Install Gunicorn web application server:
    RUN gem install unicorn
    
    ## Get your web application source using Git:
    # RUN git clone https://github.com/shykes/helloflask.git
    ## Or, by using ADD:
    # ADD /helloflask /helloflask
    ## Or, by using a package manager:
    # RUN gem install package_name --no-ri --no-rdoc
    
    # Install the requirements:
    RUN bundle install
    
    # Set the base directory of your application:
    WORKDIR /my_application
    
    # Set the port to be exposed:
    EXPOSE 8000
    
    # Set the command and arguments to execute upon launch: 
    CMD unicorn -c /path/to/config/file

## Building Docker Images And Running Containers
----------------------------------------------------------------------

Run the following command to build a new image from your Dockerfile:

    # Usage: docker build -t [image tag / name] .
    # Example:
    docker build -t my_python_app_img .

And you can start new containers using the image:

    docker run --rm -P \
               -name helloflask_app_container \
               -d my_python_app_img

> **Tip:** You can use `--rm` option to remove the container when it 
> exists successfully.
> 
> **Tip:** The `-P` flag exposes all the ports specified in the 
> Dockerfile with random public ports attached.