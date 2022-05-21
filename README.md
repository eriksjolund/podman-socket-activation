# Currently a draft
## Podman socket activation

Title suggestions:

* "_How socket activation achieves improved security and native network throughput speed for a container_"

* "_How socket activation lets Podman run a network server even with the container network disabled_"

* "_Improve security and get native network throughput speed by using socket activation_"

* "_Improve security and get native network throughput speed by using rootless Podman and socket activation_"

Running a web server container is one of the more common uses for Podman. Normally you
would need to publish the ports that need to be open by providing the option `--publish` (`-p`) to `podman run`.
When running rootless Podman you also need to be aware that the network traffic is processed
by the user space application __slirp4netns__ which comes with a performance penalty.

You might be surprised to hear that it's now possible to run a web server container with rootless Podman and
get native network throughput speed! Even more surprising is that the __--network=none__ option can be given to disable the network.
There is also no need to publish ports.

The new way to run a containerized network server is to use socket activation provided by systemd.

Socket activation conceptually works by having systemd create a socket (e.g. TCP, UDP or Unix socket). As soon as
a client connects to the socket, systemd will start the systemd service that is configured for the socket.
The newly started program inherits the open file descriptor of the socket and can accept the incoming connection.
The new feature is that Podman can now pass such a socket to the container.

Not all software daemons support socket activation but it's getting more popular.
For instance Apache HTTP server, MariaDB, DBUS, Pipewire, Gunicorn, CUPS all support socket activation support.

### Podman's socket-activated API service

On my Fedora laptop, I can find many systemd unit files that are defining sockets for socket activation:

```
$  find /usr/lib -name '*.socket' | wc -l
94
```

One of the files is _/usr/lib/systemd/user/podman.socket_

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

This Unix socket can be started by my regular user on the laptop

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

### Socket-activated echo server container in a systemd service

Podman has supported socket activation of its API service for a long time.
More recently, in version 3.4.0, Podman received support for another type of socket activation, namely, socket action
of containers. Such socket activation can be used in the systemd services that are generated with
the command `podman generate systemd --new --name CTR`.

I created a container image [__ghcr.io/eriksjolund/socket-activate-echo__](https://github.com/eriksjolund/socket-activate-echo/pkgs/container/socket-activate-echo)
that contains a simple echo server that supports socket activation. (The echo server currently has a bit limited functionality. It was written for the
sole purpose of demonstrating socket activation. Source code can found in the GitHub repo [eriksjolund/socket-activate-echo](https://github.com/eriksjolund/socket-activate-echo/)
where also more examples can be found).

Let's try it out. Start the echo server sockets

```
git clone https://github.com/eriksjolund/socket-activate-echo.git
mkdir -p ~/.config/systemd/user
cp -r socket-activate-echo/systemd/echo* ~/.config/systemd/user
systemctl --user daemon-reload
systemctl --user start echo@demo.socket
```

List the listening sockets that we will connect to

```
$ ss -lnp | grep 3000
udp   UNCONN 0      0                                       127.0.0.1:3000             0.0.0.0:*    users:(("systemd",pid=2516,fd=33))
udp   UNCONN 0      0                                           [::1]:3000                [::]:*    users:(("systemd",pid=2516,fd=35))
tcp   LISTEN 0      4096                                    127.0.0.1:3000             0.0.0.0:*    users:(("systemd",pid=2516,fd=28))
tcp   LISTEN 0      4096                                        [::1]:3000                [::]:*    users:(("systemd",pid=2516,fd=34))
v_str LISTEN 0      0                                               *:3000                   *:*    users:(("systemd",pid=2516,fd=36))
$ ss -lx | grep echo | grep u_str
u_str LISTEN 0      4096          /home/eriksjolund/echo_stream_sock.demo 49486            * 0
$
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
$ echo hello | socat - unix:$HOME/echo_stream_sock.demo
hello
$ echo hello | socat - VSOCK-CONNECT:1:3000
hello
```

### Improve security by disabling the network

In case an echo server had a security vulnerability and got compromised it might be used to launch attacks against other PCs or devices on the network.
An echo server does not need the ability to establish outgoing connections. It just needs to accept incoming connections on the socket-activated socket it inehrited.
Luckily, the command-line option __--network=none__, given to `podman run`  in the service unit file, achieves exactly what we need.

```
$ grep -A 9 ExecStart= ~/.config/systemd/user/echo@.service
ExecStart=/usr/bin/podman run \
  --cidfile=%t/%n.ctr-id \
  --cgroups=no-conmon \
  --rm \
  --sdnotify=conmon \
  --replace \
  --name echo-%i \
  --detach \
  --network none \
    ghcr.io/eriksjolund/socket-activate-echo
```

Assume an intruder has shell access in the container. The situation can be simulated by executing commands with `podman exec`.

```
$ podman exec -ti echo-demo /bin/bash -c "ip -brief addr"
lo               UNKNOWN        127.0.0.1/8 ::1/128
$ podman exec -ti echo /bin/bash -c "curl https://podman.io"
curl: (6) Could not resolve host: podman.io
$
```
Only the loopback interface is available and curl is not able to download any web page.

If we instead would remove the option  __--network=none__, the same commands show that the _tap0_ network interface is also available and that
curl is able to download a web page:
```
$ podman exec -ti echo-demo /bin/bash -c "ip -brief addr"
lo               UNKNOWN        127.0.0.1/8 ::1/128
tap0             UNKNOWN        10.0.2.100/24 fd00::9847:3aff:fe5d:97ea/64 fe80::9847:3aff:fe5d:97ea/64
$
$ podman exec -ti echo-demo /bin/bash -c "curl https://podman.io" | head -2
<!doctype html>
<html lang="en-US">
```

### Speed advantage

Using _socket activation_ comes with another advantage.

The communication in the "socket-activation" socket has native network throughput speed.
Other network traffic needs to pass through slirp4netns and gets the performance penalty that comes with it.

### Socket activate an Apache HTTP server with systemd-socket-activate

Instead of setting up a systemd service to test out socket activation, an alternative is to use the command-line tool __systemd-socket-activate__.

As an example let us use the container image [ghcr.io/eriksjolund/socket-activate-httpd](https://github.com/eriksjolund/socket-activate-httpd/pkgs/container/socket-activate-httpd)
that contains an Apache HTTP server.

In one shell, launch the container with __systemd-socket-activate__  and __podman run__.

```
$ systemd-socket-activate -l 8080 podman run --rm --network=none ghcr.io/eriksjolund/socket-activate-httpd
```

The TCP port number 8080 is only given as an option to __systemd-socket-activate__. The  __--publish__ (__-p__)
option for `podman run` is not used. As long as no client has connected, only __systemd-socket-activate__ is running.

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

Quite a lot of things happened during this web page fetch:

1. __curl__ connects to localhost TCP port 8080 and a TCP connection is established.
2. __systemd-socket-activate__ starts `podman run --rm --network=none ghcr.io/eriksjolund/socket-activate-httpd`. Podman inherits the socket.
3. Podman pulls the container image if needed.
4. Podman starts the Apache HTTP server (__httpd__) via fork/exec of conmon and the OCI runtime. The socket is inherited by __httpd__.
5. __httpd__ calls `accept()` to accept the connection.
6. __httpd__ returns the web page to __curl__. The TCP connection is then closed.
7. __httpd__ keeps running and is ready to handle any client connections that it may receive on the listening socket.

### Note about SElinux

If your computer is running SELinux, you need to have __container-selinux  2.183.0__ or newer installed.
If container socket activation via Podman does not work and you are using an older version of
container-selinux, add `--security-opt label=disable` to `podman run` as a work around.
