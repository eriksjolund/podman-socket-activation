# Currently a draft

## Podman socket activation

Socket activation conceptually works by having systemd create a socket (e.g. TCP, UDP or Unix
socket). As soon as a client connects to the socket, systemd will start the systemd service that is
configured for the socket. The newly started program inherits the file descriptor of the socket
and can then accept the incoming connection. This is the default way of how things work (i.e. when
[Accept=no](https://www.freedesktop.org/software/systemd/man/systemd.socket.html#Accept=)).

Podman supports two forms of socket activation:

* Socket activation of the API service
* Socket activation of containers

### Socket activation of the API service

Podman has supported socket activation of its API service for a long time.
The architecture is simple because the socket is used by Podman itself:

``` mermaid
stateDiagram-v2
    [*] --> systemd: client connects
    systemd --> podman: socket inherited via fork/exec
```

The file _/usr/lib/systemd/user/podman.socket_ on a Fedora system defines the Podman API socket for
rootless users:

```
$ cat /usr/lib/systemd/user/podman.socket
[Unit]
Description=Podman API Socket
Documentation=man:podman-system-service(1)

[Socket]
ListenStream=%t/podman/podman.sock
SocketMode=0660

[Install]
WantedBy=sockets.target
```

The socket is configured to be a Unix socket and can be started like this

```
$ systemctl --user start podman.socket
$ ls $XDG_RUNTIME_DIR/podman/podman.sock
/run/user/1000/podman/podman.sock
$
```
The socket can later be used by for instance __docker-compose__ that needs a Docker-compatible API

```
$ export DOCKER_HOST=unix://$XDG_RUNTIME_DIR/podman/podman.sock
$ docker-compose up
```

### Socket activation of containers

Since version 3.4.0 Podman supports socket activation of containers, i.e.,  passing
a socket-activated socket to the container. Thanks to the fork/exec model of Podman, the socket will be first
inherited by conmon and then by the OCI runtime and finally by the container
as can be seen in the following diagram:


``` mermaid
stateDiagram-v2
    [*] --> systemd: client connects
    systemd --> podman: socket inherited via fork/exec
    state "OCI runtime" as s2
    podman --> conmon: socket inherited via double fork/exec
    conmon --> s2: socket inherited via fork/exec
    s2 --> container: socket inherited via exec
```

This type of socket activation can be used in the systemd services that are generated with the command
[`podman generate systemd`](https://docs.podman.io/en/latest/markdown/podman-generate-systemd.1.html).
The container must also support socket activation. Not all software daemons support socket activation
but it's getting more popular. For instance Apache HTTP server, MariaDB, DBUS, PipeWire, Gunicorn, CUPS
all have socket activation support.

#### Example: socket-activated echo server container in a systemd service

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

A socket activated service also requires a systemd socket unit.
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

# VMADDR_CID_ANY (-1U) = 2^32 -1 = 4294967295
# See "man vsock"
ListenStream=vsock:4294967295:3000

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
$ echo hello | socat - VSOCK-CONNECT:1:3000
hello
```

The echo server works as expected. It replies _"hello"_ when receiving the text _"hello"_.

### Socket activate an Apache HTTP server with systemd-socket-activate

Instead of setting up a systemd service to test out socket activation, an alternative is to use the command-line
tool [__systemd-socket-activate__](https://www.freedesktop.org/software/systemd/man/systemd-socket-activate.html#).

As an example let us use the container image
[ghcr.io/eriksjolund/socket-activate-httpd](https://github.com/eriksjolund/socket-activate-httpd/pkgs/container/socket-activate-httpd)
that contains an Apache HTTP server.

In one shell, start __systemd-socket-activate__.

```
$ systemd-socket-activate -l 8080 podman run --rm --network=none ghcr.io/eriksjolund/socket-activate-httpd
```

The TCP port number 8080 is given as an option to __systemd-socket-activate__. The  __--publish__ (__-p__)
option for `podman run` is not used.

In another shell, fetch a web page from _localhost:8080_

```
$ curl -s localhost:8080 | head -6
<!doctype html>
<html>
  <head>
<meta charset='utf-8'>
<meta name='viewport' content='width=device-width, initial-scale=1'>
<title>Test Page for the HTTP Server on Fedora</title>
$
```

### Network throughput and latency

Using _socket activation_ comes with another advantage. The communication in the socket-activated
socket has native network throughput. When running rootless Podman, other network traffic needs to pass
through slirp4netns and gets the performance penalty that comes with it.

Unfortunately, using _socket activation_ also comes with a disadvantage. The very first connection
to a socket-activated container will have more latency due to container startup.
To minimize this latency, consider adding the __podman run__ option __--pull=never__ and
instead pull the container image beforehand.

### Note about SELinux

> **Note**
> If your computer is running SELinux, you need to have [__container-selinux 2.186.0__](https://github.com/containers/container-selinux)
> or newer installed. If container socket activation via Podman does not work and you are using an older version of
> container-selinux, add `--security-opt label=disable` to `podman run` as a work around.

### Note about running systemd in container

> **Note** There is currently no way to pass a socket-activated socket into a container when the container is running
> systemd. In other words, the executable __/usr/lib/systemd/systemd__ does not support socket activation.
> (There is a feature request related to it https://github.com/systemd/systemd/issues/17764)
