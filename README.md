# Looking to start a shell inside a Docker container?

Starting from Docker 1.3 you can use [Docker exec](https://docs.docker.com/reference/commandline/cli/#exec) to enter a Docker container. Example:

    docker exec -it CONTAINER_NAME /bin/bash

There are differences between nsenter and docker exec; namely, nsenter doesn't enter the cgroups, and therefore evades resource limitations. The potential benefit of this would be debugging and external audit, but  for remote access, **docker exec is the current recommended approach**.

**Important notice:** this repository was useful in the early days of Docker, because `nsenter` was missing from major distributions back then. `nsenter` was written in early 2013, and included in `util-linux` release 2.23. If we look at Ubuntu LTS releases, `trusty` (14.04) shipped `util-linux` 2.20, and `xenial` (16.04) shipped 2.27. In other words, if you were using Ubuntu LTS, you had to wait until 2016 to get `nsenter` through the main, official packages. That being said, all modern distros now ship with `nsenter`, and this repository is no longer useful, except for historical or curiosity purposes. **It is no longer maintained.**


## nsenter in a can

This is a small Docker recipe to build `nsenter` easily and install it in your
system.


## What is `nsenter`?

It is a small tool allowing to `enter` into `n`ame`s`paces. Technically,
it can enter existing namespaces, or spawn a process into a new set of
namespaces. "What are those namespaces you're blabbering about?"
We are talking about [container namespaces].

`nsenter` can do many useful things, but the main reason why I'm so
excited about it is because it lets you [enter into a Docker container].


## Why build `nsenter` in a container?

This is because my preferred distros (Debian and Ubuntu) ship with an
outdated version of `util-linux` (the package that should contain `nsenter`).
Therefore, if you need `nsenter` on those distros, you have to juggle with
APT repository, or compile from source, or… Ain't nobody got time for that.

I'm going to make a very bold assumption: if you landed here, it's because
you want to enter a Docker container. Therefore, you won't mind if my
method to build `nsenter` uses Docker itself.


## How do I install `nsenter` with this?

If you want to install `nsenter` into `/usr/local/bin`, just do this:

    docker run --rm -v /usr/local/bin:/target jpetazzo/nsenter

The `jpetazzo/nsenter` container will detect that `/target` is a
mountpoint, and it will copy the `nsenter` binary into it.

If you don't trust me, and prefer to extract the `nsenter` binary,
rather than allowing my container to potentially wreak havoc into
your system's `$PATH`, you can also do this:

    docker run --rm jpetazzo/nsenter cat /nsenter > /tmp/nsenter && chmod +x /tmp/nsenter

Then do whatever you want with the binary in `/tmp/nsenter`.


##  How do I *use* `nsenter`?

First, figure out the PID of the container you want to enter:

    PID=$(docker inspect --format {{.State.Pid}} <container_name_or_ID>)

Then enter the container:

    nsenter --target $PID --mount --uts --ipc --net --pid


## What's that docker-enter thing?

It's just a small shell script that wraps up the steps described above into
a tiny helper. It takes the name or ID of a container and optionally the name
of a program to execute inside the namespace. If no command is specified a
shell will be invoked instead.

    # list the root filesystem
    docker-enter my_awesome_container ls -la


## Docker toolbox usage for OS X or Windows user


### SSH to the Docker Toolbox virtual machine

    docker-machine ssh default


### Install nsenter, docker-enter, and importenv into the VM

    docker run --rm -v /usr/local/bin:/target jpetazzo/nsenter

You can also install `nsenter` to another folder. In that case, you will
need to specify the full path of `nsenter` to run it.

    docker run --rm -v /tmp:/target jpetazzo/nsenter


### Using nsenter

List running containers:

    docker ps

Identify the ID of the container that you want to get into; and retrieve
its associated PID:

    PID=$(docker inspect --format {{.State.Pid}} 08a2a025e05f)

Enter the container:

    sudo nsenter --target $PID --mount --uts --ipc --net --pid

Remember to run those commands in the Docker Toolbox virtual machine; not
in your host environment.


### Using docker-enter

With `docker-enter`, you don't need to lookup the container PID.

You can get a shell inside the container:

    docker-enter 08a2a025e05f

Or run commands directly:

    docker-enter 08a2a025e05f ls /var/log
    docker-enter 08a2a025e05f df -h


## docker-enter with boot2docker

If you are using boot2docker, you can use the function below, to:

- install `nsenter` and `docker-enter` into boot2docker's /var/lib/boot2docker/ directory,
  so they survive restarts.
- execute `docker-enter` inside of boot2docker combined with ssh

```
docker-enter() {
  boot2docker ssh '[ -f /var/lib/boot2docker/nsenter ] || docker run --rm -v /var/lib/boot2docker/:/target jpetazzo/nsenter'
  boot2docker ssh -t sudo /var/lib/boot2docker/docker-enter "$@"
}
```

You can use it directly from your host (OS X/Windows), no need to ssh into boot2docker.


## Caveats

- This only works on Intel 64 bits platforms. It should be relatively
  easy to adapt to other architectures, though.
- `nsenter` still needs to run from the host; it cannot run inside a
  container (yet).


[container namespaces]: https://www.youtube.com/watch?v=sK5i-N34im8
[enter into a Docker container]: http://jpetazzo.github.io/2014/03/23/lxc-attach-nsinit-nsenter-docker-0-9/
[Debugging a Docker container]: http://blog.loof.fr/2014/06/debugging-docker-container.html
[Nicolas De Loof]: https://twitter.com/ndeloof
