## DIY Demo for User Namespace Remapping in Docker Engine 1.12


Let's use a pristine machine for experimenting.  Using docker-machine will be better than using Docker for Mac or Docker for Windows for this lab since we'll need to ssh into the virtual machine.


```
pvn@dewgong /Users/pvn> docker-machine create lab0
Running pre-create checks...
Creating machine...
...
Checking connection to Docker...
Docker is up and running!
To see how to connect your Docker Client to the Docker Engine running on this virtual machine, run: docker-machine env lab0
pvn@dewgong /Users/pvn> eval $(docker-machine env lab0)
pvn@dewgong /Users/pvn> docker version
Client:
 Version:      1.12.0
 API version:  1.24
 Go version:   go1.6.3
 Git commit:   8eab29e
 Built:        Thu Jul 28 21:04:48 2016
 OS/Arch:      darwin/amd64
 Experimental: true

Server:
 Version:      1.12.1
 API version:  1.24
 Go version:   go1.6.3
 Git commit:   23cf638
 Built:        Thu Aug 18 17:52:38 2016
 OS/Arch:      linux/amd64
pvn@dewgong /Users/pvn> docker-machine ssh lab0
                        ##         .
                  ## ## ##        ==
               ## ## ## ## ##    ===
           /"""""""""""""""""\___/ ===
      ~~~ {~~ ~~~~ ~~~ ~~~~ ~~~ ~ /  ===- ~~~
           \______ o           __/
             \    \         __/
              \____\_______/
 _                 _   ____     _            _
| |__   ___   ___ | |_|___ \ __| | ___   ___| | _____ _ __
| '_ \ / _ \ / _ \| __| __) / _` |/ _ \ / __| |/ / _ \ '__|
| |_) | (_) | (_) | |_ / __/ (_| | (_) | (__|   <  __/ |
|_.__/ \___/ \___/ \__|_____\__,_|\___/ \___|_|\_\___|_|
Boot2Docker version 1.12.1, build HEAD : ef7d0b4 - Thu Aug 18 21:18:06 UTC 2016
Docker version 1.12.1, build 23cf638
docker@boot2docker:~$
```

OK, the first thing we'll do is start a registry. Since we aren't going to bother with certificates, we'll need to restart the engine with the --insecure-registry option.

```
docker@boot2docker:~$ sudo /etc/init.d/docker stop
docker@boot2docker:~$ sudo dockerd --insecure-registry 127.0.0.1:5000 &
INFO[0000] libcontainerd: new containerd process, pid: 2544
...
INFO[0001] Daemon has completed initialization
INFO[0001] Docker daemon                                 commit=23cf638 graphdriver=aufs version=1.12.1
INFO[0001] API listen on /var/run/docker.sock

docker@boot2docker:~$ docker run -d -p 5000:5000 --name=registry-standard registry:2
Unable to find image 'registry:2' locally
2: Pulling from library/registry

e110a4a17941: Pull complete
2ee5ed28ffa7: Pull complete
d1562c23a8aa: Pull complete
06ba8e23299f: Pull complete
802d2a9c64e8: Pull complete
Digest: sha256:1b68f0d54837c356e353efb04472bc0c9a60ae1c8178c9ce076b01d2930bcc5d
Status: Downloaded newer image for registry:2
9b0d599d86d64c77819b41788446bc3cade995be17215b12d18448156419775e
```

Now, let's take a look inside the container and look at the processes:

```
docker@boot2docker:~$ docker ps
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS                    NAMES
9b0d599d86d6        registry:2          "/entrypoint.sh /etc/"   4 minutes ago       Up 4 minutes        0.0.0.0:5000->5000/tcp   registry-standard
docker@boot2docker:~$ docker exec registry-standard ps -ef
PID   USER     TIME   COMMAND
    1 root       0:00 registry serve /etc/docker/registry/config.yml
   14 root       0:00 ps -ef
```

Note that the PID is 1 and the user is root.  Outside the container things look different:

```
docker@boot2docker:~$ ps -ef | egrep "[r]egistry|PID"
UID        PID  PPID  C STIME TTY          TIME CMD
root      2445  2431  0 22:17 ?        00:00:00 registry serve /etc/docker/registry/config.yml
```

From the host, the PID is different, though our registry is still running as root.  Before we remap the UID, let's quickly observe that just running the container with the --user option isn't viable.

```
docker@boot2docker:~$ docker stop registry-standard
registry-standard
docker@boot2docker:~$ docker run -d -p 5000:5000 --name registry-user -u 1000 registry:2
7ddb4f7081422f06ff397b4c8bb3030d2cb78e9b84b421287f56d47ad71cb749
docker@boot2docker:~$ docker ps
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS                    NAMES
7ddb4f708142        registry:2          "/entrypoint.sh /etc/"   4 seconds ago       Up 4 seconds        0.0.0.0:5000->5000/tcp   registry-user
docker@boot2docker:~$ docker exec registry-user ps -ef
PID   USER     TIME   COMMAND
    1 1000       0:00 registry serve /etc/docker/registry/config.yml
    9 1000       0:00 ps -ef

docker@boot2docker:~$ ps -ef | egrep "[r]egistry|PID"
UID        PID  PPID  C STIME TTY          TIME CMD
root      2540  2353  0 22:34 pts/0    00:00:00 dockerd --insecure-registry 127.0.0.1:5000
docker    2724  2710  0 22:38 ?        00:00:00 registry serve /etc/docker/registry/config.yml
docker    2768  2353  0 22:39 pts/0    00:00:00 egrep [r]egistry|PID
```

So now we have the registry process running as UID 1000 (the docker user).  Let's tag an image and push into this registry:

```
docker@boot2docker:~$ docker pull hello-world:latest
latest: Pulling from library/hello-world

c04b14da8d14: Pull complete
Digest: sha256:0256e8a36e2070f7bf2d0b0763dbabdd67798512411de4cdcf9431a1feb60fd9
Status: Downloaded newer image for hello-world:latest

docker@boot2docker:~$ docker tag hello-world:latest 127.0.0.1:5000/hello:user

docker@boot2docker:~$ docker push 127.0.0.1:5000/hello:user
The push refers to a repository [127.0.0.1:5000/hello]
ERRO[0429] Attempting next endpoint for push after error: Get https://127.0.0.1:5000/v2/: http: server gave HTTP response to HTTPS client
a02596fdd012: Preparing
a02596fdd012: Retrying in 2 seconds
^CERRO[0432] Not continuing with push after error: context canceled
```

If we look in the logs, we can see the cause of the failure, the process doesn't have permissions to write in /var/lib/registry:

```
docker@boot2docker:~$ docker logs registry-user | tail -2
time="2016-08-22T22:41:36Z" level=error msg="response completed with error" err.code=unknown err.detail="filesystem: mkdir /var/lib/registry/docker: permission denied" ...
...
docker@boot2docker:~$
```

OK, let's try remapping the user namespace.


```
docker@boot2docker:~$ sudo /etc/init.d/docker stop
INFO[1914] Processing signal 'terminated'
INFO[1915] stopping containerd after receiving terminated
[1]+  Done                       sudo dockerd --insecure-registry 127.0.0.1:5000
docker@boot2docker:~$ cat /etc/subuid
dockremap:165536:65536
docker@boot2docker:~$ sudo dockerd --insecure-registry 127.0.0.1:5000 --userns-remap=dockremap &
INFO[0000] libcontainerd: new containerd process, pid: 2820
...
INFO[0001] Daemon has completed initialization
INFO[0001] Docker daemon                                 commit=23cf638 graphdriver=aufs version=1.12.1
INFO[0001] API listen on /var/run/docker.sock

```

OK, now we've got that up and running.  Before we fire up the registry again, let's take a look around.  You'll notice our containers from earlier (registry-standard and registry-user) are gone, as are all the images we had downloaded earlier.


```
docker@boot2docker:~$ docker ps -a
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS               NAMES
docker@boot2docker:~$ docker images
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
```

If we investigate in /var/lib/docker, we'll see that in addition to the normal subdirectories we would expect, such as containers, volumes, image, and so on, we also have a new directory (in this case, 165536.165536).  Inside that directory is effectively a clean /var/lib/docker.


```
root@boot2docker:~# ls /var/lib/docker/165536.165536/
aufs        containers  image       network     swarm       tmp         trust       volumes
```

The 165536.165536 directory is simply the first UID and GID in the range we are remapping to.  These are determined by the entires in the /etc/subuid and /etc/subgid files corresponding to the username we passed with the --userns-remap option (in this case, "dockremap").  OK, now that we have our bearings, let's get a new registry pulled down and running.

```
docker@boot2docker:~$ docker run -d -p 5000:5000 --name=registry-userns registry:2
ERRO[1103] Handler for POST /v1.24/containers/create returned error: No such image: registry:2
Unable to find image 'registry:2' locally
2: Pulling from library/registry

e110a4a17941: Pull complete
2ee5ed28ffa7: Pull complete
d1562c23a8aa: Pull complete
06ba8e23299f: Pull complete
802d2a9c64e8: Pull complete
Digest: sha256:1b68f0d54837c356e353efb04472bc0c9a60ae1c8178c9ce076b01d2930bcc5d
Status: Downloaded newer image for registry:2
9493c7d78d0c9643c0b46c4a9869a2113cc1a3443b311d04c09b9389fb25e273
```

Now notice that the registry is running as root inside the container, but as our 165536 UID outside of the container:

```
docker@boot2docker:~$ docker exec registry-userns ps -ef
PID   USER     TIME   COMMAND
    1 root       0:00 registry serve /etc/docker/registry/config.yml
   10 root       0:00 ps -ef

docker@boot2docker:~$ ps -ef | egrep "[r]egistry|PID"
UID        PID  PPID  C STIME TTY          TIME CMD
root      2816  2353  0 23:07 pts/0    00:00:03 dockerd --insecure-registry 127.0.0.1:5000 --userns-remap=dockremap
165536    2967  2952  0 23:28 ?        00:00:00 registry serve /etc/docker/registry/config.yml
docker    3009  2353  0 23:35 pts/0    00:00:00 egrep [r]egistry|PID
```

So now we have both UID and PID remapped.  Now for the real test, will the push work?

```
docker@boot2docker:~$ docker pull hello-world:latest
latest: Pulling from library/hello-world
c04b14da8d14: Pull complete
Digest: sha256:0256e8a36e2070f7bf2d0b0763dbabdd67798512411de4cdcf9431a1feb60fd9
Status: Downloaded newer image for hello-world:latest
docker@boot2docker:~$ docker tag hello-world:latest 127.0.0.1:5000/hello:userns
docker@boot2docker:~$ docker push 127.0.0.1:5000/hello:userns
The push refers to a repository [127.0.0.1:5000/hello]
a02596fdd012: Pushed
userns: digest: sha256:a18ed77532f6d6781500db650194e0f9396ba5f05f8b50d4046b294ae5f83aa4 size: 524
```

Bonus topics:

1. If you want to start your engine with user namespace remapping active at boot time, you can simply add the --userns-remap option to your DOCKER_OPTS, whether that be in /var/lib/boot2docker/profile, /etc/default/docker, /etc/systemd/system/docker.service.d/docker.conf, or wherever your distribution configures these things.

```
docker@boot2docker:~$ cat /var/lib/boot2docker/profile

EXTRA_ARGS='
--label provider=xhyve
--userns-remap=dockremap

'
CACERT=/var/lib/boot2docker/ca.pem
DOCKER_HOST='-H tcp://0.0.0.0:2376'
DOCKER_STORAGE=aufs
DOCKER_TLS=auto
SERVERKEY=/var/lib/boot2docker/server-key.pem
SERVERCERT=/var/lib/boot2docker/server.pem
```

2. There's nothing special about the dockremap user.  We could just as easily use any other username on the host system, define a subuid and subgid range for that user, and remap to that.  

```
docker@boot2docker:~$ sudo adduser -u 5000 bozo
Changing password for bozo
New password:
Bad password: too short
Retype password:
Password for bozo changed by root

docker@boot2docker:~$ sudo vi /etc/passwd

docker@boot2docker:~$ grep bozo /etc/passwd
bozo:x:5000:5000:Linux User,,,:/home/bozo:/bin/false

docker@boot2docker:~$ sudo vi /etc/subuid
docker@boot2docker:~$ sudo vi /etc/subgid
docker@boot2docker:~$ cat /etc/subuid /etc/subgid
dockremap:165536:65536
bozo:5000:10000
dockremap:165536:65536
bozo:5000:10000

docker@boot2docker:~$ sudo /etc/init.d/docker stop
INFO[2843] Processing signal 'terminated'
INFO[2844] stopping containerd after receiving terminated
[2]+  Done                       sudo dockerd --insecure-registry 127.0.0.1:5000 --userns-remap=dockremap

docker@boot2docker:~$ sudo dockerd --userns-remap=bozo &

INFO[0000] libcontainerd: new containerd process, pid: 3052
...
INFO[0001] Daemon has completed initialization
INFO[0001] Docker daemon                                 commit=23cf638 graphdriver=aufs version=1.12.1
INFO[0001] API listen on /var/run/docker.sock

docker@boot2docker:~$ docker run -d pvnovarese/clock
Unable to find image 'pvnovarese/clock:latest' locally
latest: Pulling from pvnovarese/clock
d7e8ec85c5ab: Pull complete
a3ed95caeb02: Pull complete
Digest: sha256:3c59ef664f011e1e9fa175b735cc99a5f3e4693fecccfabba5632b4dd34655a3
Status: Downloaded newer image for pvnovarese/clock:latest
97e42496a652f718e79c720161fe3d5e021664133f78dc26b0e1cef4282fde18

docker@boot2docker:~$ ps -ef | egrep "while|PID"
UID        PID  PPID  C STIME TTY          TIME CMD
bozo      3145  3131  0 23:57 ?        00:00:00 /bin/sh -c while : \ ; do date; sleep 1; done
docker    3199  2353  0 23:57 pts/0    00:00:00 egrep while|PID
```

So in this example we have actually remapped to a live user's UID instead of to a dummy range.  This is probably a terrible idea, because if this process does escape it's container isolation, it would immediately have all privledges of that user outside of the container.  There is extremely limited upside to this and a lot of downside.  The dockremap user is specifically used in these cases precisely because it owns nothing and has no rights.  Remapping to a regular user's UID can be informative but it's best avoided outside of limited lab experimentation.



