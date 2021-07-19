# Workshop non-root containers

Many containers are running as root which is quite dangerous and considered a bad practice. It does not really matter if this container is running on a Docker host or on a Kubernetes cluster, this workshop is about containers.

## Prequisites

In order to experience the behaviour of resource limits the following setup is expected:

* minikube running local with 4 Cores and 8GiB of memory. (minikube start --memory=8G --cpus=4)
* minikube metrics-server enabled (minikube addons enable metrics-server)

network connectivity to download images

Create the minikube cluster if you haven't done it yet. Once it is started open a terminal and enter the command `kubectl get events --watch` to receive constant updates on what is happening inside the cluster.

Besides minikube this workshop also expects you to have docker installed.

## Why non root containers?

Imagine what would happen if you execute the following command: `docker run -it -v /:/host ubuntu` and that you would navigate to the `/host` directory inside the container. It could look like this:

```bash
❯ docker run --rm --hostname container-host -it -v /:/host ubuntu
root@container-host:/# cd /host
root@container-host:/host# ls
Users  bin  dev  etc  home  init  lib  lib64  linuxrc  mnt  opt  proc  root  run  sbin  squashfs.tgz  sys  tmp  usr  var
root@container-host:/host# id
uid=0(root) gid=0(root) groups=0(root)
root@container-host:/host# exit
❯ 
```

As you can see this gives us priviliged access to the filesystem on which the container runtime is running. Even if we do not have these priliveges ourselves we get elevated permissions. Needless to say that this is an unwanted scenario. It is best practice in the Linux world to have non-privileged users to limit what can be done in a filesystem.

Something else we would like to avoid is the ability to install additional software as much as possible. Images now-adays are build with an absolute minimum of executables present. Preferably the only executable present are the ones required to run an application.
But if we have root privileges we can easily install additional software or tools that help us to download and install additional software such as `curl`, `git` and `wget`.

There are several solutions that we can use to mitigate this behaviour. Both docker and Kubernetes have the ability to change the user-id of the user running inside a container at deployment time.
This is usefull for existing images.

It is also possible to alter the user running inside a container during the image build process. Both methods will be explained during this workshop.

## Changing the UID during deployment

It is possible to do this during deployment of a container. The first example shows how this can be done with docker, the second example is the equivalent in Kubernetes: (don't forget to type exit inside the container)

```bash
# Docker first
❯ docker run --rm --hostname container-host -it --user 4242 -v /:/host ubuntu
I have no name!@container-host:/$ id
uid=4242 gid=0(root) groups=0(root)
I have no name!@container-host:/$ exit
exit
❯
# Kubernetes next
❯ kubectl apply -f pod-1.yaml
pod/non-root-demo created
❯ kubectl exec -it non-root-demo -- bash
groups: cannot find name for group ID 3000
groups: cannot find name for group ID 2000
I have no name!@non-root-demo:/$ cd /host
I have no name!@non-root-demo:/host$ ls
Users  bin  data  dev  etc  home  init	lib  lib64  libexec  linuxrc  media  mnt  opt  proc  root  run	sbin  srv  sys	tmp  usr  var
I have no name!@non-root-demo:/host$ id
uid=4242 gid=3000 groups=3000,2000
I have no name!@non-root-demo:/host$ exit
exit
❯ kubectl delete -f pod-1.yaml
```

It's simple isn't it?

## Limitations when changing the UID

### Network ports

When running containers without root priliveges certain restrictions might interfere. A well known limitation is the ability to open network ports with portnumbers < 1024. 
Compare the following two commands:

```bash
❯ docker run -it --rm -p 8000:80 --name aspnetcore_sample mcr.microsoft.com/dotnet/samples:aspnetapp
❯ docker run -it --rm -p 8000:80 --name aspnetcore_sample --user 4242 mcr.microsoft.com/dotnet/samples:aspnetapp
warn: Microsoft.AspNetCore.DataProtection.Repositories.EphemeralXmlRepository[50]
      Using an in-memory repository. Keys will not be persisted to storage.
warn: Microsoft.AspNetCore.DataProtection.KeyManagement.XmlKeyManager[59]
      Neither user profile nor HKLM registry available. Using an ephemeral key repository. Protected data will be unavailable when application exits.
warn: Microsoft.AspNetCore.DataProtection.KeyManagement.XmlKeyManager[35]
      No XML encryptor configured. Key {145a686f-bdd0-4db7-be08-751c0f4e55db} may be persisted to storage in unencrypted form.
crit: Microsoft.AspNetCore.Server.Kestrel[0]
      Unable to start Kestrel.
      System.Net.Sockets.SocketException (13): Permission denied
         at System.Net.Sockets.Socket.UpdateStatusAfterSocketErrorAndThrowException(SocketError error, String callerName)
         at System.Net.Sockets.Socket.DoBind(EndPoint endPointSnapshot, SocketAddress socketAddress)
         at System.Net.Sockets.Socket.Bind(EndPoint localEP)
... omitted
```

As you can see the Kestrel service fails to start with a Permission Denied.

If we provide an environment variable we can instruct the Kestrel service to listen at another port, e.g. 5000:

```bash
❯ docker run -e ASPNETCORE_URLS=http://+:5000 -it --rm -p 8000:5000 --name aspnetcore_sample --user 4242 mcr.microsoft.com/dotnet/samples:aspnetapp
warn: Microsoft.AspNetCore.DataProtection.Repositories.EphemeralXmlRepository[50]
      Using an in-memory repository. Keys will not be persisted to storage.
warn: Microsoft.AspNetCore.DataProtection.KeyManagement.XmlKeyManager[59]
      Neither user profile nor HKLM registry available. Using an ephemeral key repository. Protected data will be unavailable when application exits.
warn: Microsoft.AspNetCore.DataProtection.KeyManagement.XmlKeyManager[35]
      No XML encryptor configured. Key {4b6376fc-2aa4-4067-812f-a7e1c3b376bc} may be persisted to storage in unencrypted form.
info: Microsoft.Hosting.Lifetime[0]
      Now listening on: http://[::]:5000
info: Microsoft.Hosting.Lifetime[0]
      Application started. Press Ctrl+C to shut down.
info: Microsoft.Hosting.Lifetime[0]
      Hosting environment: Production
info: Microsoft.Hosting.Lifetime[0]
      Content root path: /app
```

Note that the service did start and that you have to press `Ctrl + C` to stop it.

The kubernetes equivalent:

```bash
❯ kubectl apply -f pod-2.yaml
pod/non-root-kestrel created
❯ kubectl logs non-root-kestrel
warn: Microsoft.AspNetCore.DataProtection.Repositories.EphemeralXmlRepository[50]
      Using an in-memory repository. Keys will not be persisted to storage.
warn: Microsoft.AspNetCore.DataProtection.KeyManagement.XmlKeyManager[59]
      Neither user profile nor HKLM registry available. Using an ephemeral key repository. Protected data will be unavailable when application exits.
warn: Microsoft.AspNetCore.DataProtection.KeyManagement.XmlKeyManager[35]
      No XML encryptor configured. Key {007dab5b-5206-4eca-815c-421e80938657} may be persisted to storage in unencrypted form.
info: Microsoft.Hosting.Lifetime[0]
      Now listening on: http://[::]:5000
info: Microsoft.Hosting.Lifetime[0]
      Application started. Press Ctrl+C to shut down.
info: Microsoft.Hosting.Lifetime[0]
      Hosting environment: Production
info: Microsoft.Hosting.Lifetime[0]
      Content root path: /app
❯ kubectl delete -f pod-2.yaml
pod/non-root-kestrel deleted
```

### Filesystem permissions

When accessing files the system will look at the user id and the group memberships to determine which permissions a process has. Often, but not always, all users are allowed to read files. Writing files is in general more restricted.

> Side note: When writing a file permissions are checked on both file-level and directory-level. file-level is checked for existing files, directory-level when creating new files.

One way to deal with these situations is by using a volume mapping. With docker this volume mapping is always to a directory on the host or a docker volume.
With Kubernetes it can also be a so called `emptyDir`. In the examples below a directory `/external` is created:

```bash
# Docker first
❯ docker run --rm --hostname container-host -it --user 4242 -v /:/host -v /tmp:/external ubuntu
I have no name!@container-host:/$ cd /external/
I have no name!@container-host:/external$ touch test
I have no name!@container-host:/external$ exit
❯ 
# Kubernetes second
❯ kubectl apply -f pod-3.yaml
pod/non-root-demo created
❯ kubectl exec -it non-root-demo -- bash
groups: cannot find name for group ID 3000
groups: cannot find name for group ID 2000
I have no name!@non-root-demo:/$ cd /external
I have no name!@non-root-demo:/external$ touch test
I have no name!@non-root-demo:/external$ exit
exit
❯
```

## Non root from images

Another (better?) way to work with non-root containers is by changing to a non privileged user during the build of an image as shown below:

```bash
❯ docker build --file Dockerfile-1 -t non-root-image .
Sending build context to Docker daemon  65.02kB
Step 1/2 : FROM ubuntu
 ---> 9873176a8ff5
Step 2/2 : USER 4242
 ---> Using cache
 ---> 9c358bab9e1e
Successfully built 9c358bab9e1e
Successfully tagged non-root-image:latest
❯ docker run --rm --hostname container-host -it non-root-image
I have no name!@container-host:/$ id
uid=4242 gid=0(root) groups=0(root)
I have no name!@container-host:/$ exit
exit
❯
```

It is possible to change user ids at will during the build process look carefull at the following build:

```bash
❯ docker build --file Dockerfile-2 -t non-root-image-2 .
Sending build context to Docker daemon   72.7kB
Step 1/6 : FROM ubuntu
 ---> 9873176a8ff5
Step 2/6 : USER 1984
 ---> Running in 5c114b65b8e5
Removing intermediate container 5c114b65b8e5
 ---> 355e41c8cdd8
Step 3/6 : RUN id
 ---> Running in 940647ef8ded
uid=1984 gid=0(root) groups=0(root)
Removing intermediate container 940647ef8ded
 ---> 426c37a4663b
Step 4/6 : USER 0
 ---> Running in 775707c8da7c
Removing intermediate container 775707c8da7c
 ---> ccfc712b412e
Step 5/6 : RUN id
 ---> Running in d24ea4b7dcff
uid=0(root) gid=0(root) groups=0(root)
Removing intermediate container d24ea4b7dcff
 ---> 45e957866834
Step 6/6 : USER 4242
 ---> Running in 11eba29f846a
Removing intermediate container 11eba29f846a
 ---> 6777cb356b39
Successfully built 6777cb356b39
Successfully tagged non-root-image:latest
```

When permissions and ports have to be fixed inside an image we need a bit more work. By default nginx containers listen on port 80 and require a bit more convincing to run non-root:

```bash
❯ docker build --file Dockerfile-3 -t non-root-nginx .
Sending build context to Docker daemon  73.73kB
Step 1/6 : FROM nginx
 ---> 4cdc5dd7eaad
Step 2/6 : USER 0
 ---> Using cache
 ---> bb92851e853e
Step 3/6 : RUN sed -i -E 's/80/5000/' /etc/nginx/conf.d/default.conf
 ---> Using cache
 ---> 8023d5440923
Step 4/6 : RUN chown -R 4242 /var/log/nginx /var/cache/nginx
 ---> Using cache
 ---> 7edd248328d0
Step 5/6 : RUN chmod 777 /var/run
 ---> Using cache
 ---> ad46ed82b9b7
Step 6/6 : USER 4242
 ---> Using cache
 ---> dced58b78e79
Successfully built dced58b78e79
Successfully tagged non-root-nginx:latest
❯ docker run --rm -p 8000:5000 --name non-root-nginx  non-root-nginx
/docker-entrypoint.sh: /docker-entrypoint.d/ is not empty, will attempt to perform configuration
/docker-entrypoint.sh: Looking for shell scripts in /docker-entrypoint.d/
/docker-entrypoint.sh: Launching /docker-entrypoint.d/10-listen-on-ipv6-by-default.sh
10-listen-on-ipv6-by-default.sh: info: can not modify /etc/nginx/conf.d/default.conf (read-only file system?)
/docker-entrypoint.sh: Launching /docker-entrypoint.d/20-envsubst-on-templates.sh
/docker-entrypoint.sh: Launching /docker-entrypoint.d/30-tune-worker-processes.sh
/docker-entrypoint.sh: Configuration complete; ready for start up
2021/07/19 12:42:23 [warn] 1#1: the "user" directive makes sense only if the master process runs with super-user privileges, ignored in /etc/nginx/nginx.conf:2
nginx: [warn] the "user" directive makes sense only if the master process runs with super-user privileges, ignored in /etc/nginx/nginx.conf:2
2021/07/19 12:42:23 [notice] 1#1: using the "epoll" event method
2021/07/19 12:42:23 [notice] 1#1: nginx/1.21.1
2021/07/19 12:42:23 [notice] 1#1: built by gcc 8.3.0 (Debian 8.3.0-6)
2021/07/19 12:42:23 [notice] 1#1: OS: Linux 4.19.130-boot2docker
2021/07/19 12:42:23 [notice] 1#1: getrlimit(RLIMIT_NOFILE): 1048576:1048576
2021/07/19 12:42:23 [notice] 1#1: start worker processes
2021/07/19 12:42:23 [notice] 1#1: start worker process 23
2021/07/19 12:42:23 [notice] 1#1: start worker process 24
^C2021/07/19 12:42:25 [notice] 1#1: signal 2 (SIGINT) received, exiting
2021/07/19 12:42:25 [notice] 24#24: exiting
2021/07/19 12:42:25 [notice] 24#24: exit
2021/07/19 12:42:25 [notice] 23#23: exiting
2021/07/19 12:42:25 [notice] 23#23: exit
2021/07/19 12:42:25 [notice] 1#1: signal 17 (SIGCHLD) received from 24
2021/07/19 12:42:25 [notice] 1#1: worker process 24 exited with code 0
2021/07/19 12:42:25 [notice] 1#1: signal 29 (SIGIO) received
2021/07/19 12:42:25 [notice] 1#1: signal 17 (SIGCHLD) received from 23
2021/07/19 12:42:25 [notice] 1#1: worker process 23 exited with code 0
2021/07/19 12:42:25 [notice] 1#1: exit
❯
```

Again you have to press `Ctl + C` to break out of this container.

These non-root images can be deployed in a Kubernetes cluster as well.
This is not easy to demo because the image resides local on your workstation and kubernetes expects the image to be present in a repository.

With minikube there is a way around this. We repeat the build process, but this time we do it inside the minikube environment:

```bash
# Switch the build to docker inside minikube
❯ eval $(minikube docker-env)
# Build the image
❯ docker build --file Dockerfile-3 -t non-root-nginx .
Sending build context to Docker daemon  74.75kB
Step 1/6 : FROM nginx
 ---> 4cdc5dd7eaad
Step 2/6 : USER 0
 ---> Running in fdf1350683d7
Removing intermediate container fdf1350683d7
 ---> 6e4d275ac724
Step 3/6 : RUN sed -i -E 's/80/5000/' /etc/nginx/conf.d/default.conf
 ---> Running in ef745433178b
Removing intermediate container ef745433178b
 ---> 5fa3266c20c0
Step 4/6 : RUN chown -R 4242 /var/log/nginx /var/cache/nginx
 ---> Running in a1a1ebc0a0b0
Removing intermediate container a1a1ebc0a0b0
 ---> 1b931fa9b73b
Step 5/6 : RUN chmod 777 /var/run
 ---> Running in adfe4c0d98b1
Removing intermediate container adfe4c0d98b1
 ---> b58a1eb857be
Step 6/6 : USER 4242
 ---> Running in dfbd5d9baf5c
Removing intermediate container dfbd5d9baf5c
 ---> 8a2182d964e2
Successfully built 8a2182d964e2
Successfully tagged non-root-nginx:latest


```

Now we can deploy this image and inspect it:

```bash
❯ kubectl apply -f pod-4.yaml
pod/non-root-nginx created
❯ kubectl logs non-root-nginx
/docker-entrypoint.sh: /docker-entrypoint.d/ is not empty, will attempt to perform configuration
/docker-entrypoint.sh: Looking for shell scripts in /docker-entrypoint.d/
/docker-entrypoint.sh: Launching /docker-entrypoint.d/10-listen-on-ipv6-by-default.sh
10-listen-on-ipv6-by-default.sh: info: can not modify /etc/nginx/conf.d/default.conf (read-only file system?)
/docker-entrypoint.sh: Launching /docker-entrypoint.d/20-envsubst-on-templates.sh
/docker-entrypoint.sh: Launching /docker-entrypoint.d/30-tune-worker-processes.sh
/docker-entrypoint.sh: Configuration complete; ready for start up
2021/07/19 13:17:40 [warn] 1#1: the "user" directive makes sense only if the master process runs with super-user privileges, ignored in /etc/nginx/nginx.conf:2
nginx: [warn] the "user" directive makes sense only if the master process runs with super-user privileges, ignored in /etc/nginx/nginx.conf:2
2021/07/19 13:17:40 [notice] 1#1: using the "epoll" event method
2021/07/19 13:17:40 [notice] 1#1: nginx/1.21.1
2021/07/19 13:17:40 [notice] 1#1: built by gcc 8.3.0 (Debian 8.3.0-6)
2021/07/19 13:17:40 [notice] 1#1: OS: Linux 4.19.182
2021/07/19 13:17:40 [notice] 1#1: getrlimit(RLIMIT_NOFILE): 1048576:1048576
2021/07/19 13:17:40 [notice] 1#1: start worker processes
2021/07/19 13:17:40 [notice] 1#1: start worker process 24
2021/07/19 13:17:40 [notice] 1#1: start worker process 25
2021/07/19 13:17:40 [notice] 1#1: start worker process 26
2021/07/19 13:17:40 [notice] 1#1: start worker process 27
❯ kubectl delete -f pod-4.yaml
```
