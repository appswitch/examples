===========================================
Single Node Cluster - Basic AppSwitch Usage
===========================================

Bring up the VM with
::

   $ vagrant up

Log into the VM with
::

   $ vagrant ssh

The AppSwitch binary (``ax``) is distributed as an image via Docker Hub.
Login to docker to be able to pull it.
::

   $ docker login

(This requires having an account on Docker Hub.)


Playing with AppSwitch
======================

You can now play with AppSwitch (ax) using this single node cluster.  The
docker-compose file includes a service to start the appswitch daemon and
also a service to start a python web server, start them with
::

   $ docker-compose up -d appswitch
   $ docker-compose up -d test

You can see the services running with
::

   $ docker-compose ps
   Name                       Command               State   Ports
   ----------------------------------------------------------------------
   appswitch_appswitch_1   /entrypoint.sh                   Up
   appswitch_test_1        /usr/bin/ax run --name tes ...   Up


You can then use AppSwitch to run ``curl``
::

   $ sudo ax run -- curl -I test:8000
   HTTP/1.0 200 OK
   Server: SimpleHTTP/0.6 Python/3.6.5
   Date: Wed, 06 Jun 2018 01:37:55 GMT
   Content-type: text/html; charset=utf-8
   Content-Length: 882


Run ``ax`` client manually
--------------------------

Let's start another web server manually.  We will assign it an IP address
of 1.1.1.1 and give it a DNS resolvable name 'webserver'.
::

   $ sudo ax run --ip 1.1.1.1 --name webserver python -m SimpleHTTPServer 8000

This is the same way the ``test`` service was started by
``docker-compose`` (except we are using python 2 here).  Again we can curl
the webserver using either the IP address or the name that we have assigned.
::

   $ sudo ax run -- curl -I webserver:8000

or::

   $ sudo ax run -- curl -I 1.1.1.1:8000

AppSwitch includes commands to view current resources.  (See ``ax get --help``.)
::

   $ ax get apps
   NAME        APPID    NODEID   CLUSTER       APPIP       DRIVER     LABELS          ZONES
   -----------------------------------------------------------------------------------------------
   webserver  f00001e0  host    appswitch  1.1.1.1         user    zone=default  [zone==default]
   test       f00001a4  host    appswitch  10.209.224.143  user    zone=default  [zone==default]

You can see from the output that ax assigned an IP address to the ``test``
service since it was started (by ``docker-compose``) using only the
``--name`` option.  The ``webserver`` service shows the IP we assigned to
it.  If we start a service without assigning it a name ax will generate a
unique identifier for use as the service name.


Connecting to services without AppSwitch
----------------------------------------

Services run by AppSwitch can be externally exposed with the ``--ports``
options. Another way to do the same after the application is already
brought up is to through ``vservice``.
::

   $ sudo ax run --ip 1.1.1.1 --name webserver --ports 8000:10.0.2.15:8000 python -m SimpleHTTPServer 8000

We can now connect to the web server using curl without wrapping the
command in ``ax``
::

   $ curl -I 10.0.2.15:8000
   HTTP/1.0 200 OK
   Server: SimpleHTTP/0.6 Python/2.7.5
   Date: Wed, 06 Jun 2018 03:18:02 GMT
   Content-type: text/html; charset=UTF-8
   Content-Length: 274

When starting the web server you can see we have explicitly used the IP
address of the interface we wish to use.  We could also have used the
wildcard address ``0.0.0.0`` to expose the service on all interfaces.

If you start two web server processes both with the same name and IP then
AppSwitch will load balance when a connection to that name/IP is made.

Virtual Services
~~~~~~~~~~~~~~~~

If we start the web server without an IP
::

   $ sudo ax run -- python -m SimpleHTTPServer 8000

And view the app
::

   $ ax get apps
                      NAME                    APPID    NODEID   CLUSTER     APPIP    DRIVER     LABELS          ZONES       
   -----------------------------------------------------------------------------------------------------------------------
     <9142a421-00e8-483e-83d0-eea9716c849a>  f000015d  host    appswitch  10.0.2.15  user    zone=default  [zone==default] 

We can associate an IP address with this app by creating a virtual service.
::

   $ ax create vservice --ip 1.1.1.1 --backends 10.0.2.15  --ports 8000 myvsvc
   Service 'myvsvc' created successfully with IP '1.1.1.1'.
   $ ax get vservices
     VSNAME  VSTYPE   VSIP       VSPORTS     VSBACKENDIPS  
   ------------------------------------------------------
     myvsvc  Random  1.1.1.1  [{8000 8000}]  [10.0.2.15]

Now we can curl to the virtual IP or the virtual name.  This feature
enables multiple IPs for the same server since the server is still
available at the IP assigned it by ax.  Furthermore, if we start more than
one server we can add them all as backends for the virtual service and
AppSwitch will load balance when connecting to the virtual name or IP.


