# spray-basic-authentication-cassandra
We will secure our spray service with Basic Authentication. For this example however, we will only secure the /secure
context, and not the /api context. The data store will be [Apache Cassandra](http://cassandra.apache.org/), a distributed
high-performance and high-availablility datastore. The example will be using the 
[akka-persistence-cassandra](https://github.com/krasserm/akka-persistence-cassandra) extension by 
[Martin Krasser](https://github.com/krasserm) to store Actor state.

# Command Query Responsibility Segregation (CQRS)
CQRS is the principle that uses separate Query and Command objects to retrieve and modify data. It can be used with 
Event Sourcing to store the events and replay them back to recover the state of the Actor aka. Aggregate. Gregg Young
and Eric Evans do a very good job at explaining the concepts so I invite you to search for lectures from them about
these topics.

This solution uses Akka Actors as domain aggregates, companion objects for the bounded contexts, to put the case classes
into, the domain commmands and events, Akka-persistence for event sourcing, views and aggregate separation for CQRS, 
and basic authentication to authenticate access to our precious resource. 

# Docker
This example can be run using [Docker](http://docker.io) and I would strongly advice using Docker and just take a few 
hours to work through the guide on how to use this *great* piece of software.

## Run the image
When you have Docker installed, you can launch a [containerized version](https://registry.hub.docker.com/u/dnvriend/spray-ba-cass/). 
However, this example uses four containers, three that will run the Cassandra cluster and one that will run the example application.
We will use the Cassandra images by [Tushar Pokle](https://github.com/pokle/cassandra):

    $ sudo docker run -d --name cass1 poklet/cassandra start
    $ CASS1_IP=$(sudo docker inspect --format '{{ .NetworkSettings.IPAddress }}' cass1)
    $ sudo docker run -d --name cass2 poklet/cassandra start $CASS1_IP
    $ sudo docker run -d --name cass3 poklet/cassandra start $CASS1_IP

Finally we can launch the example container:

    $ sudo docker run -d -P --link cass1:cas1 --link cass2:cas2 --link cass3:cas3 --name spray-ba-cass dnvriend/spray-ba-cass

To view the log output:

    $ sudo docker logs -f spray-ba-cass

Check which local port has been mapped to the Vagrant VM:
    
    $ sudo docker ps spray-ba-cass
    
And note the entries in the PORTS column eg:

    ONTAINER ID        IMAGE                           COMMAND                CREATED             STATUS              PORTS                                                                           NAMES
    76406c7bc77d        dnvriend/spray-ba-cass:latest   /bin/sh -c java -jar   57 seconds ago      Up 56 seconds       0.0.0.0:49161->8080/tcp                                                         spray-ba-cass
    77d9d5fca591        poklet/cassandra:latest         start 172.17.0.2       9 minutes ago       Up 9 minutes        22/tcp, 61621/tcp, 7000/tcp, 7001/tcp, 7199/tcp, 8012/tcp, 9042/tcp, 9160/tcp   cass3,spray-ba-cass/cas3
    a2743fc6db4f        poklet/cassandra:latest         start 172.17.0.2       10 minutes ago      Up 10 minutes       22/tcp, 61621/tcp, 7000/tcp, 7001/tcp, 7199/tcp, 8012/tcp, 9042/tcp, 9160/tcp   cass2,spray-ba-cass/cas2,stoic_feynman/cas2
    9032f04406e5        poklet/cassandra:latest         start                  14 minutes ago      Up 14 minutes       22/tcp, 61621/tcp, 7000/tcp, 7001/tcp, 7199/tcp, 8012/tcp, 9042/tcp, 9160/tcp   cass1,spray-ba-cass/cas1,stoic_feynman/cas1

In this example, the local port of my Vagrant VM has been mapped to port 49161 to the port of the example application, and that is 8080. 
Point the browser to the following url (change the port to your mapped port):

    http://192.168.99.99:49161/web/index.html    
    
To stop the container:

    $ sudo docker stop spray-ba-cass
    
To remove the image from your computer:
    
    $ sudo docker rm -f cass1
    $ sudo docker rm -f cass2
    $ sudo docker rm -f cass3
    $ sudo docker rm -f spray-ba-cass

## Viewing the journal/snapshot data
The other nodes, most likely are 172.17.0.2 (cass1) 172.17.0.3 (cass2) and 172.17.0.4 (cass3). To verify that the data
is stored on all the nodes, we should query them: 

    $ sudo docker run -ti poklet/cassandra /bin/bash

Then type:
    
    bash-4.1# cqlsh 172.17.0.2
    Connected to Test Cluster at 172.17.0.2:9160.
    [cqlsh 4.1.1 | Cassandra 2.0.6 | CQL spec 3.1.1 | Thrift protocol 19.39.0]
    Use HELP for help.
    cqlsh>

Alternatively you  could also run cqlsh directly:

    $ sudo docker run -ti poklet/cassandra cqlsh $CASS1_I
    
or
    
    $ sudo docker run -ti poklet/cassandra cqlsh 172.17.0.3

By default, the journal messages are stored in the keyspace 'akka' and in the table 'messages':
    
    cqlsh> select * from akka.messages;

By default, snapshots are stored in the keyspace 'akka_snapshot' and in the table 'snapshots':
    
    cqlsh> select * from akka_snapshot.snapshots;

Login to all the nodes, and verify the data can be resolved.
    
# Httpie
We will use the *great* tool [httpie](https://github.com/jakubroztocil/httpie), so please install it:

# REST API
## Getting a list of users
        
    $ http http://localhost:8080/api/users
    
## Get a user by name
The username has been encoded in the path
    
    $ http http://localhost:8080/api/users/foo
    
## Adding or updating users

    $ http PUT http://localhost:8080/api/users username="foo" password="bar"
    
## Deleting a user
The username has been encoded in the path

    $ http DELETE http://localhost:8080/api/users/foo
    
# The secured resource
The resource that has been secured for us is in the /secure context

## No authentication

    $ http http://localhost:8080/secure

## Basic Authentication
    
    $ http -a foo:bar http://localhost:8080/secure
    
# User Credentials CRUD Interface
The application should automatically launch your browser and show the CRUD screen, but if it doesn't:

    http://localhost:8080/web/index.html
        
Have fun!