*NOTE: This guide was written with 1.10.0-dev (experimental).  The official 1.10 release has some slightly different behavior, so this guide is obsolete but I'm archiving it for completeness.  Latest version can be found in pvnovarese/workbook.*

Current configuration:

```
pvn@gyarados /home/pvn> uname -a
Linux gyarados 4.2.5-300.fc23.x86_64 #1 SMP Tue Oct 27 04:29:56 UTC 2015 x86_64 x86_64 x86_64 GNU/Linux
pvn@gyarados /home/pvn> docker -v
Docker version 1.10.0-dev, build 8537501, experimental
pvn@gyarados /home/pvn> docker version
Client:
 Version:      1.10.0-dev
 API version:  1.22
 Go version:   go1.5.2
 Git commit:   8537501
 Built:        Mon Dec 21 21:05:49 2015
 OS/Arch:      linux/amd64
 Experimental: true

Server:
 Version:      1.10.0-dev
 API version:  1.22
 Go version:   go1.5.2
 Git commit:   8537501
 Built:        Mon Dec 21 21:05:49 2015
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
pvn@gyarados /home/pvn> grep dockremap /etc/passwd
dockremap:x:10000:10000::/home/dockremap:/bin/false
pvn@gyarados /home/pvn> cat /etc/subuid
dockremap:200000:65536
pvn@gyarados /home/pvn> cat /etc/subgid
dockremap:200000:65536

pvn@gyarados /home/pvn> sudo docker daemon --userns-remap=dockremap &
WARN[0000] Running experimental build
INFO[0000] User namespaces: ID ranges will be mapped to subuid/subgid ranges of: dockremap:dockremap
WARN[0000] devmapper: Usage of loopback devices is strongly discouraged for production use. Please use `--storage-opt dm.thinpooldev` or use `man docker` to refer to dm.thinpooldev section.
WARN[0000] devmapper: Base device already exists and has filesystem xfs on it. User specified filesystem  will be ignored.
INFO[0000] [graphdriver] using prior storage driver "devicemapper"
INFO[0000] Firewalld running: true
INFO[0000] Default bridge (docker0) is assigned with an IP address 172.17.0.1/16. Daemon option --bip can be used to set a preferred IP address
INFO[0001] Loading containers: start.

INFO[0001] Loading containers: done.
INFO[0001] Daemon has completed initialization
INFO[0001] Docker daemon                                 commit=8537501 execdriver=native-0.2 graphdriver=devicemapper version=1.10.0-dev
INFO[0001] API listen on /var/run/docker.sock
```

OK, looks like it started.  First thing I notice, all my images are gone.

```
pvn@gyarados /home/pvn> docker images
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
pvn@gyarados /home/pvn>
```

OK, lets see what's going on there.

```
pvn@gyarados /home/pvn> ls /var/lib/docker/
0.0/  200000.200000/  network/
```

OK, so this remapped engine will basically operate in a new environment.  OK, let's pull something and fire it up.

```
pvn@gyarados /home/pvn> docker pull pvnovarese/mprime
Using default tag: latest
latest: Pulling from pvnovarese/mprime
a3ed95caeb02: Pull complete
546e579918ed: Pull complete
Digest: sha256:0b315a681a6b9f14f93ab34f3c744fd547bda30a03b55263d93861671fa33b00
Status: Downloaded newer image for pvnovarese/mprime:latest

pvn@gyarados /home/pvn> docker run -d --name=mprime0 pvnovarese/mprime:latest
d460aeea507417388cafef7ffab1dc2267d0a6f4953215387dbdd1f4174ad669

pvn@gyarados /home/pvn> ps aux | grep [m]prime
200000   27395 99.1  0.0  15224 11328 ?        RNs  09:41   0:59 /mprime -t
```

sweet, it's working.

However, if we run multiple containers...

```
pvn@gyarados /home/pvn> docker run -d --name=mprime1 pvnovarese/mprime:latest
0488c83798c901225d4947c5d13dba30a259cda2fdee14341d529bdd0e3f3674

pvn@gyarados /home/pvn> docker ps
CONTAINER ID        IMAGE                      COMMAND             CREATED             STATUS              PORTS               NAMES
0488c83798c9        pvnovarese/mprime:latest   "/mprime -t"        13 minutes ago      Up 13 minutes                           mprime1
d460aeea5074        pvnovarese/mprime:latest   "/mprime -t"        13 minutes ago      Up 13 minutes                           mprime0

pvn@gyarados /home/pvn> ps aux | grep [m]prime
200000   14719 99.8  0.0  15224 11604 ?        RNs  10:03  13:44 /mprime -t
200000   14908 99.7  0.0  15224 11632 ?        RNs  10:03  13:20 /mprime -t
```

Note processes in both containers are using the same UID.  



notes:

http://integratedcode.us/2015/10/13/user-namespaces-have-arrived-in-docker/
