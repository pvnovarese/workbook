# experimenting with Notary and Docker Content Trust

## What is Docker Content Trust?

Docker Content Trust is an implementation of the open-source Notary project, which is used to sign and verify arbitrary blobs of data.  In this case, Notary is used to sign and verify Docker images.  Notary is baked into Docker Engine starting with version 1.8 and Notary server is already fully integrated with Docker Hub, and also available to integrate with Docker Trusted Registry (this is an experimental feature in DTR 1.4). 

## Related information

For more in-depth examples and advanced concepts refer to the Docker documentation:

- [Content Trust Concepts](https://docs.docker.com/security/trust/content_trust/)
- [Automation Systems](https://docs.docker.com/security/trust/trust_automation/)
- [Understanding and Managing Keys](https://docs.docker.com/security/trust/trust_key_mng/)
- [Personal Sandbox](https://docs.docker.com/security/trust/trust_sandbox/)
	•	https://www.docker.com/docker-security 
	•	https://github.com/docker/notary
	•	http://blog.docker.com/2015/08/content-trust-docker-1-8/


## Prerequisites

* An existing Docker Hub account
* Docker Engine 1.8 or newer
* Logged into Docker Hub with the Docker client



## Phase 1: get familiar with DCT and verifying pulls

First, enable DCT by setting the environment variable:

```
pvn@gyarados /home/pvn> export DOCKER_CONTENT_TRUST=1

``` 

Try to pull unsigned content:

```
pvn@gyarados /home/pvn> docker pull jpetazzo/clock
Using default tag: latest
no trust data available
```

Operations without trust data will fail. You can temporarily disable content trust to pull unsigned content:
```
pvn@gyarados /home/pvn> docker pull --disable-content-trust jpetazzo/clock
Using default tag: latest
latest: Pulling from jpetazzo/clock
a3ed95caeb02: Pull complete
1db09adb5ddd: Pull complete
Digest: sha256:446edaa1594798d89ee2a93f660161b265db91b026491e4671c14371eff5eea0
Status: Downloaded newer image for jpetazzo/clock:latest
``` 
Enable debug mode to compare default and signed pulls

```
pvn@gyarados /home/pvn> docker -D pull --disable-content-trust hello-world
Using default tag: latest
latest: Pulling from library/hello-world
03f4658f8b78: Already exists
a3ed95caeb02: Already exists
Digest: sha256:8be990ef2aeb16dbcb9271ddfe2610fa6658d13f6dfb8bc72074cc1ca36966a7
Status: Downloaded newer image for hello-world:latest

pvn@gyarados /home/pvn> docker rmi hello-world
Untagged: hello-world:latest
Deleted: sha256:690ed74de00f99a7d00a98a5ad855ac4febd66412be132438f9b8dbd300a937d

pvn@gyarados /home/pvn> docker -D pull hello-world
Using default tag: latest
 …
 [debug output snipped]
 …
DEBU[0002] successfully verified targets
Pull (1 of 1): hello-world:latest@sha256:8be990ef2aeb16dbcb9271ddfe2610fa6658d13f6dfb8bc72074cc1ca36966a7
sha256:8be990ef2aeb16dbcb9271ddfe2610fa6658d13f6dfb8bc72074cc1ca36966a7: Pulling from library/hello-world
03f4658f8b78: Already exists
a3ed95caeb02: Already exists
Digest: sha256:8be990ef2aeb16dbcb9271ddfe2610fa6658d13f6dfb8bc72074cc1ca36966a7
Status: Downloaded newer image for hello-world@sha256:8be990ef2aeb16dbcb9271ddfe2610fa6658d13f6dfb8bc72074cc1ca36966a7
Tagging hello-world@sha256:8be990ef2aeb16dbcb9271ddfe2610fa6658d13f6dfb8bc72074cc1ca36966a7 as hello-world:latest
```

At this point you’ll notice your docker images output looks a little different than usual.
```
pvn@gyarados /home/pvn> docker images
REPOSITORY               TAG                 IMAGE ID            CREATED             SIZE
hello-world              latest              690ed74de00f        12 weeks ago        960 B
hello-world              <none>              690ed74de00f        12 weeks ago        960 B
jpetazzo/clock           latest              12068b93616f        10 months ago       2.43 MB
```

In this case, the reference with the “<none>” tag is the signed reference.  You can see this more explicitly with the “digests=true” option.
```
pvn@gyarados /home/pvn> docker images --digests=true
REPOSITORY               TAG                 DIGEST                                                                    IMAGE ID            CREATED             SIZE
hello-world              latest              <none>                                                                    690ed74de00f        12 weeks ago        960 B
hello-world              <none>              sha256:8be990ef2aeb16dbcb9271ddfe2610fa6658d13f6dfb8bc72074cc1ca36966a7   690ed74de00f        12 weeks ago        960 B
jpetazzo/clock           latest              <none>                                                                    12068b93616f        10 months ago       2.43 MB
```


## Phase 2: Registry operations with Docker Content Trust


We are going to use the hello-world image we pulled in the previous exercise to compare signed and unsigned pushes into our user namespace in Docker Hub.


Login to Hub with Engine 1.8 or newer.

```
pvn@gyarados /home/pvn> docker login
Username: <username>
Password: <password>
Email: <email>
WARNING: login credentials saved in /home/ubuntu/.docker/config.json
Login Succeeded 
```

Make sure content trust is disabled, then re-tag the hello-world image so we can push it into our own namespace.

```
pvn@gyarados /home/pvn>  unset DOCKER_CONTENT_TRUST
pvn@gyarados /home/pvn>  docker tag hello-world <username>/trust-test:latest
```

When we push the image, note that the -D flag will enable debug output if you want to see more details on what’s happening.  For unsigned pushes this is not going to reveal much, but when we do signed pushes we’ll see a lot more detail.

```
pvn@gyarados /home/pvn> docker -D push <username>/trust-test:latest
```

Now, let’s re-enable Content Trust and pull the image we just pushed.

```
pvn@gyarados /home/pvn> export DOCKER_CONTENT_TRUST=1
pvn@gyarados /home/pvn> docker pull <username>/trust-test:latest
no trust data available
```

Again, this pull fails because the image is not signed.

However, now that we have Content Trust enabled, we can re-push the image and Engine will create the trust data for us (again, use the -D flag if you want to see additional detail).  

```
pvn@gyarados /home/pvn> docker tag hello-world <username>/trust-test:signed
pvn@gyarados /home/pvn> docker push <username>/trust-test:signed
```

Example output:

        pvn@gyarados /home/pvn> docker push pvnovarese/trust-test:signed
        The push refers to a repository [docker.io/pvnovarese/trust-test] (len: 1)
        2c5ac3f849df: Image already exists
        ab2b8a86ca6c: Image already exists
        signed: digest: sha256:b33669e38aef420ff10d4787c07afe11a679564f65c31bde39572c9196f3e143 size: 3214
        Signing and pushing trust metadata
        You are about to create a new root signing key passphrase. This passphrase
        will be used to protect the most sensitive key in your signing system. Please
        choose a long, complex passphrase and be careful to keep the password and the
        key file itself secure and backed up. It is highly recommended that you use a
        password manager to generate the passphrase and keep it safe. There will be no
        way to recover this key. You can find the key in your config directory.
        Enter passphrase for new root key with id 020aa4f:
        Repeat passphrase for new root key with id 020aa4f:
        Enter passphrase for new repository key with id docker.io/pvnovarese/trust-test (ab6b057):
        Repeat passphrase for new repository key with id docker.io/pvnovarese/trust-test (ab6b057):
        Finished initializing "docker.io/pvnovarese/trust-test”

At this point in your Docker hub repository you have two different images/tags:

| Image               | Description                                                                      |
|---------------------|----------------------------------------------------------------------------------|
| `trust-test:latest` | This image is still unsigned and cannot be pulled when content trust is enabled. |
| `trust-test:signed` | This image is signed and can be pulled if content trust is enabled or disabled.  |

