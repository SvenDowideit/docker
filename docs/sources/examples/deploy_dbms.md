# Deploying Databases Using Docker
----------------------------------------------------------------------

## Introduction And Contents
----------------------------------------------------------------------

### Introduction
----------------------------------------------------------------------

Deployment and management of databases can be tricky. Especially when
you need to prevent data loss, provide failover and ensure data security.

In this example, we will see how to use Docker to simplify this process 
by using *containers*. Through this tutorial we will also learn the best 
and most common practices of running database applications inside them.

**By the end of this example, you will know how to:**

 - Deploy and manage databases using containers.
 - Separate the data persistence layer from *inside* to the *outside*
 of containers.
 - Automate the process using *Dockerfiles*.

## Understanding The Challenge
----------------------------------------------------------------------

### Deploying And Managing Databases
----------------------------------------------------------------------

In many modern applications it is not uncommon to have scenarios 
whereby multiple databases are used, for example combining the use of 
NoSQL and relational databases.

For deployments, all this translates to preparing different server(s) 
with custom configuration options for each. When coupled with challenges 
of management, the curve of dealing with all this can get really steep.

With its objective approach towards application hosting, Docker 
containers can help you with implementing fail-over mechanisms,
high-availability, and much more in an easier and more logical way.

### Docker For Databases
----------------------------------------------------------------------

For databases such as *MySQL* or *PostgreSQL*, keeping the data 
safe and secure is a must. The ability to share volumes (i.e. parts of 
the storage on the host machine) makes it possible to back-up, upgrade,
and cluster (multiple) database servers easily compared to working with
basic servers or server images.

## Data Storage With Containers
----------------------------------------------------------------------

### Default Storage
----------------------------------------------------------------------

Docker containers are designed to be lightweight and easy to handle for 
the host machine.

When you run a process inside a container (i.e. `docker run`), it does
not have outside access. Unless explicitly specified, all the data kept
inside is disposed when the container stops — leaving the base image 
as it was without any changes.

Put simply, all the data that gets downloaded and installed, together 
with outputs and other information the running process generates are 
temporary. Only *committed* containers do keep the data (i.e. state) 
after being stopped — which translates to creating a new image with a
much larger footprint for applications like databases.

### Persistent Data Storage
----------------------------------------------------------------------

Databases usually require the data to be persistent. Meaning that no 
matter what happens to the process, the data should stay on the host 
and it should be accessible. This way of working also ensures that 
containers are are kept lean with the application logic separated and
portability still intact.

In order to achieve this persistence, we use Docker *volumes*.

Docker volumes come in two forms:

 - Mounted volumes, and;
 - Shared volumes.

> **Note:** To learn more about volumes, consider checking out the 
> documentation on the subject: [Share Directories via Volumes](
> http://docs.docker.io/en/latest/use/working_with_volumes/).

For MySQL, where the database itself defines the format of the data, a 
simple *host mount* (or volume creation) during build can be sufficient.
In fact, it will offer more-than-enough flexibility for deployments.

For some others (e.g. PostgreSQL), trusting the management of volumes to
an independent container (i.e. using *shared volumes*) will be a better
way of deploying databases. Accessing persistent data like this, with the 
`--volumes-from` argument, also increases the security.

In our example, we will go through both approaches, beginning with the 
creation of a Data Volume Container.

## Creating a Data Volume Container
----------------------------------------------------------------------

A *Data Volume Container* is simply a Docker container created with one
or more *volumes* attached. Their sole purpose is to offer a data 
persistence platform for other containers that need consistent access.

> **Note:** DVCs do not need to be running to perform their duty.

Creating a DVC can be done by either using a `Dockerfile`, or manually 
via `docker run`. A good practice would be to (preferably) tag it for 
easier access (when used with the `--volumes-from` argument).

### Creating a DVC By Starting A New Container
----------------------------------------------------------------------

You can simply create a DVC with the `docker run` command.
This way volumes can be passed as arguments using the `-v` flag, i.e.:

    # Usage: docker run [-v /path/to/volume] [name] [image] [process]
    # Example:
    docker run -v /etc/postgresql \
               -v /var/log/postgresql \
               -v /var/lib/postgresql \
               -name PSQL_DATA \
               busybox true

### Creating a DVC By Using A Dockerfile
----------------------------------------------------------------------

In order to automate the process in a maintainable way, you can choose 
to work with a `Dockerfile` instead:

    ####################################################
    # Dockerfile to manage data volumes for PostgreSQL #
    ####################################################
    
    FROM          busybox
    MAINTAINER    O.S. Tezer, sonat@docker.com
     
    VOLUME        ["/etc/postgresql", "/var/log/postgresql", "/var/lib/postgresql"]
    CMD           ["/bin/true"]

Once the Dockerfile is ready, you can build an image by executing:

    # Usage: docker build [options] .    
    # Example:
    docker build -t psql_data .

And then you can run (or initiate) the container with:

    docker run -name PSQL_DATA_CONTAINER psql_data

And that's it! Our DVC is ready to be accessed and used.
               
## Creating A Database Application Container
----------------------------------------------------------------------

We are going to use a `Dockerfile` to create an example of a database
container image. Once you build the image, you can use it to run
database containers working together with an accompanying *DVC* or by 
mounting host volumes.

Below is a `Dockerfile` to prepare a container from the base `ubuntu` 
image to run a PostgreSQL instance:

    #########################################################
    # Dockerfile to host light-weight PostgreSQL instances  #
    # using Data Volume Containers as persistence objects   #
    #########################################################
    
    FROM ubuntu
    MAINTAINER Sven Dowideit, SvenDowideit@docker.com
    
    # Add the PostgreSQL PGP key to verify their Debian packages.
    # It should be the same key as:
    # https://www.postgresql.org/media/keys/ACCC4CF8.asc 
    
    RUN apt-key adv --keyserver keyserver.ubuntu.com --recv-keys B97B0AFCAA1A47F044F244A07FCC7D46ACCC4CF8
    
    # Add PostgreSQL's repository.
    # It contains the most recent stable release
    # of PostgreSQL 9.3
    RUN echo "deb http://apt.postgresql.org/pub/repos/apt/ precise-pgdg main" > /etc/apt/sources.list.d/pgdg.list
    
    # Update the Ubuntu and PostgreSQL repository indexes
    RUN apt-get update
    
    # Install:
    # python-software-properties,
    # software-properties-common,
    # PostgreSQL 9.3
    # There are some warnings (in red) that show up during the build.
    # You can hide them by prefixing each apt-get statement with:
    # DEBIAN_FRONTEND=noninteractive
    RUN apt-get -y -q install python-software-properties software-properties-common
    RUN apt-get -y -q install postgresql-9.3 postgresql-client-9.3 postgresql-contrib-9.3
    # Note: The official Debian and Ubuntu images automatically
    # apt-get clean after each apt-get 
    
    # Run the rest of the commands as the postgres user created by
    # the postgres-9.3 package when it was apt-get installed
    USER postgres
    
    # Create a PostgreSQL role named 
    # docker
    # with
    # docker
    # as the password and then;
    # create a database docker owned by the docker role.
    # Note: here we use &&\ to run commands one after the other
    # the \ allows the RUN command to span multiple lines.
    RUN /etc/init.d/postgresql start && \
        psql --command "CREATE USER docker WITH SUPERUSER PASSWORD 'docker';" && \
        createdb -O docker docker
    
    # Adjust PostgreSQL configuration so that remote connections to the
    # database are possible. 
    RUN echo "host all  all    0.0.0.0/0  md5" >> /etc/postgresql/9.3/main/pg_hba.conf
    
    # And add:
    # listen_addresses to:
    # /etc/postgresql/9.3/main/postgresql.conf
    RUN echo "listen_addresses='*'" >> /etc/postgresql/9.3/main/postgresql.conf
    
    # Expose the PostgreSQL port:
    EXPOSE 5432
        
    # Set the default command to run when starting the container
    CMD ["/usr/lib/postgresql/9.3/bin/postgres", "-D", "/var/lib/postgresql/9.3/main", "-c", "config_file=/etc/postgresql/9.3/main/postgresql.conf"]

Once the Dockerfile is ready, we can use it to build a new container
image, e.g.:

    docker build -t psql_93_image .

And to use it to run the database container(s), e.g.: 

    # Mounted Volumes:
    docker run --rm -P \
               -v /etc/postgresql \
               -v /var/log/postgresql \
               -v /var/lib/postgresql \
               -name pg_test \
               psql_93_image

    # Shared Volumes:
    docker run --rm -P \
               --volumes-from PSQL_DATA_CONTAINER \
               -name POSTGRES_EXAMPLE \
               psql_93_image               

> **Tip:** The `--rm` removes the container when it exists successfully.
> 
> **Tip:** The `-P` flag exposes all the ports specified in the 
> Dockerfile with random public ports attached.