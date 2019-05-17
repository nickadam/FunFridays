# HAProxy on docker

The docker team maintains an haproxy official image. Documentation for the image is available here.

https://docs.docker.com/samples/library/haproxy/

Unfortunately this image does not come with a sample config file so we can try and generate one by using alpine's installer package.

```
docker run alpine apk add haproxy
docker ps -a # get the container name
docker cp {container name}:/etc/haproxy/haproxy.cfg .
```

This sample config file does a great job of communicating what haproxy can do. There is a listener setup with two different backends and acls to steer traffic to the appropriate backend.

To clean this up a bit so there is only one load balanced backend we can remove everthing that isn't associated with default_backend.

