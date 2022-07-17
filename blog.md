# Currently a draft
## Podman socket activation

Title: "_Use socket activation with Podman to get improved security_"

Subtitle: "_Learn how to restrict network access for a containerized network server_"

--------------------------------

Network services that are facing the public internet are exposed to the risk of being attacked. There are different ways of how we could better protect them.
For instance, using strong passwords and keeping the systems up to date with the latest security updates, will reduce the risk of an intrusion.

Another advice is trying to minimize the potential damages caused by a compromised network daemon.
Assuming the compromise is not caused by a kernel bug, the intruder will not be able to gain any more privileges
than those of the running network daemon. A good strategy is therefore to run the network daemon with as few privileges as possible.
There is a new feature in Podman that opens up the possibility to run a network daemon with more limited access to the internet.
Since version 3.4.0 Podman supports socket activation of containers, i.e., passing a socket-activated socket to the container.
Interestingly, it's possible for a container to use such a socket-activated socket even when the network is disabled, (i.e., when the option __--network=none__ is given to `podman run`).

Not all software daemons support socket activation but it's getting more popular, e.g., Apache HTTP server, MariaDB, DBUS, PipeWire, Gunicorn, CUPS all have socket activation support.

### An echo server example

Let's try out [socket-activate-echo](https://github.com/eriksjolund/socket-activate-echo/pkgs/container/socket-activate-echo), a simple echo server container that supports socket activation.
For this we need to use a Linux system because socket activation is provided by systemd.

Create the container

```
$ podman create --rm --name echo --network none ghcr.io/eriksjolund/socket-activate-echo
```

Generate the systemd service unit

```
$ mkdir -p ~/.config/systemd/user
$ podman generate systemd --name --new echo > ~/.config/systemd/user/echo.service
```

A socket-activated service also requires a systemd socket unit.
Create the file _~/.config/systemd/user/echo.socket_ where we define the
sockets that the container should use

```
[Unit]
Description=echo server

[Socket]
ListenStream=127.0.0.1:3000
ListenDatagram=127.0.0.1:3000
ListenStream=[::1]:3000
ListenDatagram=[::1]:3000
ListenStream=%h/echo_stream_sock

[Install]
WantedBy=default.target
```

`%h` is a systemd specifier that expands to the user's home directory.

After editing the unit files, systemd needs to reload it's configuration

```
$ systemctl --user daemon-reload
```

Start the socket unit

```
$ systemctl --user start echo.socket
```

Test the echo server with the program __socat__

```
$ echo hello | socat - tcp4:127.0.0.1:3000
hello
$ echo hello | socat - tcp6:[::1]:3000
hello
$ echo hello | socat - udp4:127.0.0.1:3000
hello
$ echo hello | socat - udp6:[::1]:3000
hello
$ echo hello | socat - unix:$HOME/echo_stream_sock
hello
```

The echo server works as expected. It replies _"hello"_ after receiving the text _"hello"_.

### Improved security by using --network=none

In case the echo server would get compromised due to a security vulnerability, the container might be used to
launch attacks against other PCs or devices on the network. An echo server does not need the ability to
establish outgoing connections. It just needs to accept incoming connections on the socket-activated socket it
inherited. Luckily, the command-line option
[__--network=none__](https://docs.podman.io/en/latest/markdown/podman-run.1.html#network-mode-net), given to
`podman run` in the service unit file, provides those restrictions.

```
$ grep -B8 -- --network=none ~/.config/systemd/user/echo.service
ExecStart=/usr/bin/podman run \
	--cidfile=%t/%n.ctr-id \
	--cgroups=no-conmon \
	--rm \
	--sdnotify=conmon \
	-d \
	--replace \
	--name echo \
	--network=none ghcr.io/eriksjolund/socket-activate-echo
```

Assume an intruder has shell access in the container. The situation can be simulated by executing
commands with `podman exec`.

```
$ podman exec -ti echo /bin/bash -c "ip -brief addr"
lo               UNKNOWN        127.0.0.1/8 ::1/128
```

Only the loopback interface is available.

```
$ podman exec -ti echo /bin/bash -c "curl https://podman.io"
curl: (6) Could not resolve host: podman.io
```

__curl__ is not able to download any web page. The network interface _tap0_ that rootless
Podman normally uses to access the internet is not available.

If we instead remove the option __--network=none__ and run the same commands,
we see that the network interface _tap0_ is available

```
$ podman exec -ti echo /bin/bash -c "ip -brief addr"
lo               UNKNOWN        127.0.0.1/8 ::1/128
tap0             UNKNOWN        10.0.2.100/24 fd00::9847:3aff:fe5d:97ea/64 fe80::9847:3aff:fe5d:97ea/64
```

and that __curl__ is able to download the web page.

```
$ podman exec -ti echo /bin/bash -c "curl https://podman.io" | head -2
<!doctype html>
<html lang="en-US">
```

By using the option __--network=none__, we thus limit the possibilities for an intruder to use the
compromised container as a starting point for attacks on other PCs.

See also [Podman socket activation tutorial](https://github.com/containers/podman/blob/main/docs/tutorials/socket_activation.md).

### Note about SELinux

> **Note**
> If your computer is running SELinux, you need to have
> [__container-selinux 2.183.0__](https://github.com/containers/container-selinux)
> or newer installed to run the examples as specified above. If you are using an older version of
> container-selinux, add `--security-opt label=disable` to `podman run` as a work around.
