============================
Multi-Node AppSwitch Cluster
============================

This example will assume you have played with the single node cluster
example.  We will not repeat docker/docker-compose information based on
this assumption.

We can play with a multi-node AppSwitch cluster using two virtual machine
nodes.  The nodes are called ``host0`` and ``host1``.

Bring up the VMs with
::

   $ vagrant up

Open up two terminals and in each log into a VM
::

   $ vagrant ssh host0
   $ vagrant ssh host1


The nodes have the following network interfaces configured

- host0: 192.168.56.10
- host1: 192.168.56.11


Configure and Start the Daemon
==============================

There are two configuration files provided, one for each node.  Direct
docker-compose to the correct config file by defining the environment
variable ``$AX_CONFIG_FILE``.  On each node run
::

   $ export AX_CONFIG_FILE="$(hostname)-ax.config"

A quick look at the config files will show that we are passing the daemon
the node IP and a list of neighbors, in this case the IP addresses of both
nodes.  Bring up the daemon on each node
::

   $ docker-compose up -d appswitch

(Remember to log into Docker Hub.)


Playing with AppSwitch
======================

Now we can have a play with these two nodes.  First let's start a web
server on host0
::

   $ sudo ax run --ip 1.1.1.1 --name webserver python -m SimpleHTTPServer 8000

We can then try curl'ing this server from host1 (actually we can reach this
service from either node).
::

   $ sudo ax run -- curl -I webserver:8000
   HTTP/1.0 200 OK
   Server: SimpleHTTP/0.6 Python/2.7.5
   Date: Wed, 06 Jun 2018 03:18:02 GMT
   Content-type: text/html; charset=UTF-8
   Content-Length: 274


Similar to the single node example we can also expose the service.  If we
use the wildcard address ``0.0.0.0`` then the service will be exposed on
both nodes.
::

    $ sudo ax run --ip 1.1.1.1 --name webserver --ports 8000:0.0.0.0:8000 python -m SimpleHTTPServer 8000

From host0 we can then directly curl the web server
::

   $ curl -I 192.168.56.11:8000

And on host1 we can hit it also
::

   $ curl -I 192.168.56.10:8000
