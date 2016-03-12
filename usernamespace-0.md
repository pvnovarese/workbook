Current configuration:

```
pvn@gyarados /home/pvn> uname -a
Linux gyarados 4.3.5-300.fc23.x86_64 #1 SMP Mon Feb 1 03:18:41 UTC 2016 x86_64 x86_64 x86_64 GNU/Linuxi
pvn@gyarados /home/pvn> docker version
Client:
 Version:      1.11.0-dev
 API version:  1.23
 Go version:   go1.5.3
 Git commit:   79edcc5
 Built:        Sat Feb 13 00:36:00 2016
 OS/Arch:      linux/amd64
 Experimental: true

Server:
 Version:      1.11.0-dev
 API version:  1.23
 Go version:   go1.5.3
 Git commit:   79edcc5
 Built:        Sat Feb 13 00:36:00 2016
 OS/Arch:      linux/amd64
 Experimental: true
```

and I have a few images ready to dork around with:

```
pvn@gyarados /home/pvn> docker images
REPOSITORY               TAG                 IMAGE ID            CREATED             SIZE
pvnovarese/mprime        latest              459769dbc7a1        12 days ago         4.461 MB
pvnovarese/clock         latest              f568fa0096f6        12 days ago         1.113 MB
pvnovarese/debug         latest              1b8cc940d2c6        2 weeks ago         8.038 MB
sysdig/sysdig            latest              7deee9c45f7f        2 weeks ago         552.6 MB
busybox                  latest              d9551b4026f0        3 weeks ago         1.113 MB
alpine                   latest              558af09712a4        3 months ago        5.244 MB
```

OK, let's start the engine with user namespaces active.  First thing is to make sure we've got entries in passwd and the /etc/sub*id files:

```
pvn@gyarados /home/pvn> grep bozo /etc/passwd
bozo:x:5000:5000::/home/bozo:/bin/bash
pvn@gyarados /home/pvn> grep bozo /etc/group
bozo:x:5000:
pvn@gyarados /home/pvn> cat /etc/subuid
bozo:5000:65536
pvn@gyarados /home/pvn> cat /etc/subgid
bozo:5000:65536
```

Note here, the UID/GID we are actually remapping to does not have to match the UID/GID of the username in /etc/passwd.  Whatever is in the subuid/subgid files is what will actually own the processes we start.  Despite this, you do actually have to match the user name itself in the passwd and subuid files with the name you pass on the command line to the engine in the --userns-remap flag.  I don't (yet) understand why this works this way.

Anyway, let's fire up the engine with the --userns-remap flag:

```
pvn@gyarados /home/pvn> sudo docker daemon --userns-remap=bozo &
[1] 659
pvn@gyarados /home/pvn> WARN[0000] Running experimental build
INFO[0000] User namespaces: ID ranges will be mapped to subuid/subgid ranges of: bozo:bozo
WARN[0000] devmapper: Usage of loopback devices is strongly discouraged for production use. Please use `--storage-opt dm.thinpooldev` or use `man docker` to refer to dm.thinpooldev section.
INFO[0000] devmapper: Creating filesystem xfs on device docker-8:19-30671130-base
INFO[0000] devmapper: Successfully created filesystem xfs on device docker-8:19-30671130-base
INFO[0001] Graph migration to content-addressability took 0.00 seconds
INFO[0001] Firewalld running: true
INFO[0001] Default bridge (docker0) is assigned with an IP address 172.17.0.0/16. Daemon option --bip can be used to set a preferred IP address
INFO[0002] Loading containers: start.

INFO[0002] Loading containers: done.
INFO[0002] Daemon has completed initialization
INFO[0002] Docker daemon                                 commit=79edcc5 execdriver=native-0.2 graphdriver=devicemapper version=1.11.0-dev
INFO[0002] API listen on /var/run/docker.sock

pvn@gyarados /home/pvn>
```

OK, looks like it started.  First thing I notice, all my images are gone.

```
pvn@gyarados /home/pvn> docker images
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
pvn@gyarados /home/pvn>
```

OK, lets see what's going on there.

```
pvn@gyarados /home/pvn> sudo ls -lF /var/lib/docker/
total 80
drwx------.  9 bozo   bozo   4096 Mar 11 20:03 5000.5000/
drwx------. 15 root   root   4096 Mar 11 09:35 containers/
drwx------.  5 root   root   4096 Jul  4  2015 devicemapper/
drwxr-xr-x.  2 root   root   4096 Feb  4 08:25 discovery_certs/
drwx------. 66 root   root  16384 Dec 23 11:31 graph/
drwx------.  3 root   root   4096 Dec 23 21:00 image/
drwx------.  2 root   root   4096 Jul  4  2015 init/
-rw-r--r--.  1 root   root  13312 Mar 11 09:26 linkgraph.db
drwxr-x---.  3 root   root   4096 Oct 15 16:52 network/
-rw-------.  1 root   root   1257 Dec 23 11:31 repositories-devicemapper
drwx------.  6 root   root   4096 Mar 11 20:02 tmp/
drwx------.  2 root   root   4096 Jul  4  2015 trust/
drwx------. 17 root   root   4096 Mar 11 09:25 volumes/
pvn@gyarados /home/pvn>
```

OK, so this remapped engine will basically operate in a new environment (in the 5000.5000 directory). We can look in there and see it's essentially a new /var/lib/docker.

```
pvn@gyarados /home/pvn> sudo su -
[root@gyarados ~]# cd /var/lib/docker/5000.5000/
[root@gyarados 5000.5000]# ls
containers/  devicemapper/  image/  network/  tmp/  trust/  volumes/
[root@gyarados 5000.5000]#
```

OK, let's pull something and fire it up.

```
pvn@gyarados /home/pvn> docker pull pvnovarese/mprime
Using default tag: latest
latest: Pulling from pvnovarese/mprime

a3ed95caeb02: Pull complete
546e579918ed: Pull complete
Digest: sha256:21561b776f6e3f30044d09e40f31d696425354e4a1885da10c153eb5bb707237
Status: Downloaded newer image for pvnovarese/mprime:latest
pvn@gyarados /home/pvn> docker run -d --name=mprime0 pvnovarese/mprime:latest
7f47d752ba9d110c162acfcac7d0ed696495b60b8677b1556de771b382429c2c
pvn@gyarados /home/pvn> ps aux | grep [m]prime
bozo      1518 91.0  0.0  15224 11652 ?        RNs  20:12   0:07 /mprime -t
```

sweet, it's working.

However, if we run multiple containers...

```
pvn@gyarados /home/pvn> docker run -d --name=mprime1 pvnovarese/mprime:latest
e087aee0a9ce5a65285db081159f676ae8a5eecb62e019175721368d8d83f653
pvn@gyarados /home/pvn> docker ps
CONTAINER ID        IMAGE                      COMMAND             CREATED              STATUS              PORTS               NAMES
e087aee0a9ce        pvnovarese/mprime:latest   "/mprime -t"        11 seconds ago       Up 8 seconds                            mprime1
7f47d752ba9d        pvnovarese/mprime:latest   "/mprime -t"        About a minute ago   Up 59 seconds                           mprime0
pvn@gyarados /home/pvn> ps aux | grep [m]prime
bozo      1518 99.5  0.0  15224 11652 ?        RNs  20:12   1:09 /mprime -t
bozo      1657 97.9  0.0  15224 11648 ?        RNs  20:13   0:19 /mprime -t
pvn@gyarados /home/pvn>
```

Note processes in both containers are using the same UID.



notes:

http://integratedcode.us/2015/10/13/user-namespaces-have-arrived-in-docker/

