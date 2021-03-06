# HAProxy on docker

## 1. Get a sample config file from alpine

https://github.com/nickadam/FunFridays/tree/1243e333787d04cd0dd9cea3000161fe47a7c101

The docker team maintains an haproxy official image. Documentation for the image is available here.

https://docs.docker.com/samples/library/haproxy/

Unfortunately this image does not come with a sample config file so we can try and generate one by using alpine's installer package.

```
docker run alpine apk add haproxy
docker ps -a # get the container name
docker cp {container name}:/etc/haproxy/haproxy.cfg .
```

This sample config file does a great job of communicating what haproxy can do. There is a listener setup with two different backends and acls to steer traffic to the appropriate backend.

## 2. Modify sample config file for our purposes

https://github.com/nickadam/FunFridays/tree/177fd6c6d0aee0ebb2ea4be484873b387d157f3b

To clean this up a bit so there is only one load balanced backend we can remove everthing that isn't associated with default_backend.

Now we have a listener on 5000 and for backend server that roundrobin between localhost ports 5001-5004, not very useful since there isn't anything running there. Change these to something useful, or just pick some random internet sites to load balance.

## 3. Run the config file in the official haproxy image

https://github.com/nickadam/FunFridays/tree/5704613c5976f7493006738bbd5c34a7f1c79374/haproxy

We can follow the steps at https://docs.docker.com/samples/library/haproxy/ to create our Dockerfile

```
FROM haproxy:1.7
COPY haproxy.cfg /usr/local/etc/haproxy/haproxy.cfg
```

Build the image

```
docker build -t my-haproxy .
```

Test the configuration file

```
docker run -it --rm --name haproxy-syntax-check my-haproxy haproxy -c -f /usr/local/etc/haproxy/haproxy.cfg
```

Oh bummer we have some errors in our config, lets fix those.

```
$ docker run -it --rm --name haproxy-syntax-check my-haproxy haproxy -c -f /usr/local/etc/haproxy/haproxy.cfg
<7>haproxy-systemd-wrapper: executing /usr/local/sbin/haproxy -p /run/haproxy.pid -db -c -f /usr/local/etc/haproxy/haproxy.cfg -Ds
[ALERT] 136/143629 (10) : parsing [/usr/local/etc/haproxy/haproxy.cfg:31] : cannot find user id for 'haproxy' (0:Success)
[ALERT] 136/143629 (10) : parsing [/usr/local/etc/haproxy/haproxy.cfg:32] : cannot find group id for 'haproxy' (0:Success)
[ALERT] 136/143629 (10) : Error(s) found in configuration file : /usr/local/etc/haproxy/haproxy.cfg
[ALERT] 136/143629 (10) : Fatal errors found in configuration.
<5>haproxy-systemd-wrapper: exit, haproxy RC=1
```

Let's change the user id and group id to root and try again.

```
docker build -t my-haproxy .
docker run -it --rm --name haproxy-syntax-check my-haproxy haproxy -c -f /usr/local/etc/haproxy/haproxy.cfg
```

Yaay config file is valid!
```
$ docker run -it --rm --name haproxy-syntax-check my-haproxy haproxy -c -f /usr/local/etc/haproxy/haproxy.cfg
<7>haproxy-systemd-wrapper: executing /usr/local/sbin/haproxy -p /run/haproxy.pid -db -c -f /usr/local/etc/haproxy/haproxy.cfg -Ds
Configuration file is valid
<5>haproxy-systemd-wrapper: exit, haproxy RC=0
```

Run a container from your new image

```
docker run -p 8080:5000 --name my-running-haproxy my-haproxy
```

Bummer another error, lets get rid of stats and chroot and run again.

Works! No errors and it didn't exit.

```
$ docker run --name my-running-haproxy my-haproxy
<7>haproxy-systemd-wrapper: executing /usr/local/sbin/haproxy -p /run/haproxy.pid -db -f /usr/local/etc/haproxy/haproxy.cfg -Ds
```

Now lets curl it a few times and see what we get

```
$ curl 127.0.0.1:8080
<HTML><HEAD><meta http-equiv="content-type" content="text/html;charset=utf-8">
<TITLE>301 Moved</TITLE></HEAD><BODY>
<H1>301 Moved</H1>
The document has moved
<A HREF="http://www.google.com:8080/">here</A>.
</BODY></HTML>
```

```
$ curl 127.0.0.1:8080
<!DOCTYPE html>
<html lang="en-us">
  <head>
    <meta http-equiv="content-type" content="text/html; charset=UTF-8">
    <meta charset="utf-8">
    <title>Yahoo</title>
...
```

```
$ curl 127.0.0.1:8080
<h2>Our services aren't available right now</h2><p>We're working to restore
```

```
$ curl 127.0.0.1:8080
<!DOCTYPE html>
<!--[if IEMobile 7 ]> <html lang="en_US" class="no-js iem7"> <![endif]-->
<!--[if lt IE 7]> <html class="ie6 lt-ie10 lt-ie9 lt-ie8 lt-ie7 no-js" lang="en_US"> <![endif]-->
<!--[if IE 7]>    <html class="ie7 lt-ie10 lt-ie9 lt-ie8 no-js" lang="en_US"> <![endif]-->
<!--[if IE 8]>    <html class="ie8 lt-ie10 lt-ie9 no-js" lang="en_US"> <![endif]-->
<!--[if IE 9]>    <html class="ie9 lt-ie10 no-js" lang="en_US"> <![endif]-->
<!--[if (gte IE 9)|(gt IEMobile 7)|!(IEMobile)|!(IE)]><!--><html class="no-js" lang="en_US"><!--<![endif]-->
```

Looks like it's working, each request is being round robin to the backend servers.

## 3. Push image to docker hub, or the registry of your choice

```
docker tag my-haproxy nickadam/funfridayhaproxy
docker push nickadam/funfridayhaproxy
```

Now our image is available from anywhere on the internet.

## 4. Make a service with docker-compose

https://github.com/nickadam/FunFridays/tree/5cfd99126e9e7c3ddf90a40825c3ca6e378a7282/haproxy

Rather than define all our command arguments to run this service in the command line, let's put them in a docker-compose.yml file.

The reference for docker-compose is available here, https://docs.docker.com/compose/compose-file/

Now we can launch the service using docker-compose.

```
docker-compose up -d
```

And we can stop it with docker-compose too

```
docker-compose down
```

## 5. Prepare docker-compose to run in a swarm

https://github.com/nickadam/FunFridays/tree/6b1cd723b73b47471f4060c64973a6b1d42355e9/haproxy

To move our config to production we can add a few tidbits to it that are specific to our swarm. If you don't have a uniform cluster or your fastidious about where processes are running you can contrain your service to specific cluster nodes using labels.

If you don't have a swarm, you can easily make a single node swarm with just one command.
```
docker swarm init
```

Add a label to your node and modify the docker-compose file to only run on the nodes with the label. https://docs.docker.com/engine/swarm/manage-nodes/#add-or-remove-label-metadata 

```
docker node ls # get the node hostname
docker node update --label-add my-haproxy=true {node hostname}
```

## 6. Deploy to docker swarm

We can deploy our "stack" of services to docker swarm by referencing our docker-compose file
```
docker stack deploy -c docker-compose.yml haproxy
```

We can see the service is running 

```
$ docker service ls
ID             NAME                MODE    REPLICAS  IMAGE                             PORTS
6l5rky29cdxl   haproxy_my-haproxy  global  1/1       nickadam/funfridayhaproxy:latest  *:8080->5000/tcp
```

Since the image is accessible from the registry all that's needed for production is the compose file. The production servers will pull down the appropriate image and run it in the cluster.
