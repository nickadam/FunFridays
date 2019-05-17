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