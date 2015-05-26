# Docker repo to spawn selenium standalone servers with Chrome and Firefox with VNC support
[![Gitter](https://badges.gitter.im/Join Chat.svg)](https://gitter.im/elgalu/docker-selenium?utm_source=badge&utm_medium=badge&utm_campaign=pr-badge&utm_content=badge)

* selenium-server-standalone
* google-chrome-stable
* firefox (stable)
* VNC access (useful for debugging the container)
* openbox (lightweight window manager using freedesktop standards)

## Note this repo evolved into SeleniumHQ/docker-selenium
See: https://github.com/SeleniumHQ/docker-selenium

However SeleniumHQ/docker-selenium project focus on building selenium grids while this one focusing on debugging via VNC and meant to be run as a standalone selenium server.

### One-liner Install & Usage

In general: add `sudo` only if needed in your environment and `--privileged` if you really need it.

    sudo docker run --privileged -p 4444:4444 -p 5900:5900 -e SCREEN_WIDTH=1550 -e SCREEN_HEIGHT=1110 -e VNC_PASSWORD=secret elgalu/selenium:v2.45.0-openbox1

If your setup is correct, privileged mode and sudo should not be necessary. Also guacamole server is now available:

    docker run --rm --name=ch -p=0.0.0.0:8081:8080 -p=0.0.0.0:2222:2222 -p=0.0.0.0:4470:4444 -p=0.0.0.0:5920:5900 -e SCREEN_WIDTH=1800 -e SCREEN_HEIGHT=1110 -e VNC_PASSWORD=hola -e SSH_PUB_KEY="$(cat ~/.ssh/id_rsa.pub)" -e WITH_GUACAMOLE=true elgalu/selenium:v2.45.0-ssh1

Then open a browser into http://localhost:8081/#/login/ and login to guacamole with user "docker" and the same password as ${VNC_PASSWORD} so you no longer need a VNC client to debug the docker instance.

You can also ssh into the machine as long as `SSH_PUB_KEY="$(cat ~/.ssh/id_rsa.pub)"` is correct.

    ssh -p 2222 -o StrictHostKeyChecking=no application@localhost

That's is useful for tunneling else you can stick with `docker exec` to get into the instance with a shell:

    docker exec -ti ch bash

### Step by step non-privileged do it yourself

#### 1. Build this image

Ensure you have the Ubuntu base image downloaded, this step is optional since docker takes care of downloading the parent base image automatically, but for the sake of curiosity:

    docker run -i -t ubuntu:14.04.2 /bin/bash

If you don't git clone this repo, you can simply build from github:

    docker build github.com/elgalu/docker-selenium
    ID=$(docker images -q | sed -n 1p)
    docker tag $ID elgalu/docker-selenium:latest

If you git clone this repo locally, i.e. cd into where the Dockerfile is, you can:

    docker build -t="elgalu/docker-selenium:local" .

If you prefer to download the final built image from docker you can pull it, personally I always prefer to build them manually except for the base images like Ubuntu 14.04.2:

    docker pull elgalu/selenium:latest

#### 2. Use this image

##### e.g. Spawn a container for Chrome testing:

    CH=$(docker run --rm --name=ch -p=127.0.0.1::4444 -p=127.0.0.1::5900 \
        -v /e2e/uploads:/e2e/uploads elgalu/docker-selenium:local)

Note `-v /e2e/uploads:/e2e/uploads` is optional in case you are testing browser uploads on your webapp you'll probably need to share a directory for this.

The `127.0.0.1::` part is to avoid binding to all network interfaces, most of the time you don't need to expose the docker container like that so just *localhost* for now.

I like to remove the containers after each e2e test with `--rm` since this docker container is not meant to preserve state, spawning a new one is less than 3 seconds. You need to think of your docker container as processes, not as running virtual machines if case you are familiar with vagrant.

A dynamic port will be binded to the container ones, i.e.

    # Obtain the selenium port you'll connect to:
    docker port $CH 4444
    #=> 127.0.0.1:49155

    # Obtain the VNC server port in case you want to look around
    docker port $CH 5900
    #=> 127.0.0.1:49160

In case you have RealVNC binary `vnc` in your path, you can always take a look, view only to avoid messing around your tests with an unintended mouse click or keyboard.

    ./bin/vncview.sh 127.0.0.1:49160

##### e.g. Spawn a container for Firefox testing:

This command line is the same as for Chrome, remember that the selenium running container is able to launch either Chrome or Firefox, the idea around having 2 separate containers, one for each browser is for convenience plus avoid certain `:focus` issues you web app may encounter during e2e automation.

    FF=$(docker run --rm --name=ff -p=127.0.0.1::4444 -p=127.0.0.1::5900 \
        -v /e2e/uploads:/e2e/uploads elgalu/docker-selenium:local)

##### Look around

    docker images
    #=>

    REPOSITORY               TAG                 IMAGE ID            CREATED             VIRTUAL SIZE
    elgalu/docker-selenium   local               eab41ff50f72        About an hour ago   931.1 MB
    ubuntu                   14.04.2             d0955f21bf24        4 weeks ago         188.3 MB

### Troubleshooting

All output is sent to stdout so it can be inspected by running:

``` bash
$ docker logs -f <container-id|container-name>
```

Container leaves a few logs files to see what happened:

    /tmp/Xvfb_headless.log
    /tmp/xmanager.log
    /tmp/x11vnc_forever.log
    /tmp/local-sel-headless.log
    /tmp/selenium-server-standalone.log
