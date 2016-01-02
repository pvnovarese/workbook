```
pvn@gyarados /home/pvn> docker run -it --rm --security-opt seccomp:deny:nanosleep alpine /bin/sh
Unable to find image 'alpine:latest' locally
latest: Pulling from library/alpine
8f4ec95ceaee: Pull complete
Digest: sha256:78a756d480bcbc35db6dcc05b08228a39b32c2b2c7e02336a2dcaa196547a41d
Status: Downloaded newer image for alpine:latest
docker: Error response from daemon: Container command not found or does not exist..

pvn@gyarados /home/pvn> docker run -it --rm alpine /bin/sh
/ #
```

need to figure out syntax for seccomp security-opt.  What I tried above was from https://www.youtube.com/watch?v=sw3NjVMMXz8 which is really old.

need to review

https://github.com/docker/docker/blob/master/docs/security/seccomp.md

https://github.com/docker/docker/issues/17142

https://www.youtube.com/watch?v=sw3NjVMMXz8

https://github.com/jfrazelle/dotfiles/blob/master/etc/docker/seccomp/chrome.json
