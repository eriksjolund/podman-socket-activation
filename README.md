# Currently a draft
## Podman socket activation

Title: "_Use socket activation with Podman to get improved security and native network throughput_"

Subtitle: "_Learn how to restrict network access for a containerized network server_"

Socket activation conceptually works by having systemd create a socket (e.g. TCP, UDP or Unix socket). As soon as
a client connects to the socket, systemd will start the systemd service that is configured for the socket.
The newly started program inherits the open file descriptor of the socket and accepts the incoming connection.
This is default way how things work (i.e. when [Accept=no](https://www.freedesktop.org/software/systemd/man/systemd.socket.html#Accept=)).

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

This type of socket activation can be used in the systemd services that are generated with the command [`podman generate systemd`](https://docs.podman.io/en/latest/markdown/podman-generate-systemd.1.html). The container must also support socket activation. Not all software daemons support socket activation but it's getting more popular.
For instance Apache HTTP server, MariaDB, DBUS, PipeWire, Gunicorn, CUPS all have socket activation support.

#### Example: socket-activated echo server container in a systemd service

The container image [__ghcr.io/eriksjolund/socket-activate-echo__](https://github.com/eriksjolund/socket-activate-echo/pkgs/container/socket-activate-echo)
contains an echo server that supports socket activation. Source code is available in the GitHub repo [eriksjolund/socket-activate-echo](https://github.com/eriksjolund/socket-activate-echo/)
where also more examples can be found.

To try it out, clone the GitHub repository and install the systemd units. Then start the echo server sockets

```
git clone https://github.com/eriksjolund/socket-activate-echo.git
mkdir -p ~/.config/systemd/user
cp -r socket-activate-echo/systemd/* ~/.config/systemd/user
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

In case the echo server would get compromised due to a security vulnerability, the container might be used to launch attacks against other PCs or devices on the network.
An echo server does not need the ability to establish outgoing connections. It just needs to accept incoming connections on the socket-activated socket it inherited.
Luckily, the command-line option __--network=none__, given to `podman run`  in the service unit file, provides those restrictions.

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

Only the loopback interface is available

```
$ podman exec -ti echo-demo /bin/bash -c "ip -brief addr"
lo               UNKNOWN        127.0.0.1/8 ::1/128
```

__curl__ is not able to download any web page

```
$ podman exec -ti echo-demo /bin/bash -c "curl https://podman.io"
curl: (6) Could not resolve host: podman.io
$
```

If we instead remove the option  __--network=none__ and run the same commands
we see that the network interface _tap0_ is also available

```
$ podman exec -ti echo-demo /bin/bash -c "ip -brief addr"
lo               UNKNOWN        127.0.0.1/8 ::1/128
tap0             UNKNOWN        10.0.2.100/24 fd00::9847:3aff:fe5d:97ea/64 fe80::9847:3aff:fe5d:97ea/64
$
```

and that __curl__ is able to download the web page.

```
$ podman exec -ti echo-demo /bin/bash -c "curl https://podman.io" | head -2
<!doctype html>
<html lang="en-US">
$
```

By using the option  __--network=none__, we thus limit the possibilities for an intruder to use the
compromised container as a starting point for attacks on other PCs.

### Network throughput and latency

Using _socket activation_ comes with another advantage. The communication in the socket-activated
socket has native network throughput. Other network traffic needs to pass through slirp4netns and
gets the performance penalty that comes with it.

Unfortunately, using _socket activation_ also comes with a disadvantage. The very first connection
to a socket-activated container will have more latency due to container startup.
To minimize this latency, consider adding the __podman run__ option __--pull=never__ and
instead pull the container image beforehand.

### Restrict Podman with _RestrictAddressFamilies_

When using Podman in a systemd service, the __systemd__ directive [__RestrictAddressFamilies__](https://www.freedesktop.org/software/systemd/man/systemd.exec.html#RestrictAddressFamilies=)
can be used to restrict Podman's access to sockets. The restriction only concerns the use of the system call __socket()__, which means that socket-activated sockets are unaffected by the directive.

Containers that only need internet access via socket-activated sockets can still be run by Podman when
systemd is configured to restrict Podman's ability to use the system call `socket()` for the AF_INET and AF_INET6
socket families. Of course, Podman would then be blocked from pulling down any container image so the container image needs to be present beforehand.

Let's see how we could use __RestrictAddressFamilies__  for the socket-activated echo server.
If the `--pull=never` option is added to `podman run`, the echo server container will continue to work even with the very restricted setting

```
RestrictAddressFamilies=AF_UNIX AF_NETLINK
```

All use of the system call `socket()` is then disallowed except for the socket families AF_UNIX sockets and AF_NETLINK sockets.

In case there would be a security vulnerability in Podman, conmon or runc, this configuration limits
the possibilities an intruder has to launch attacks on other PCs on the network.

The example _echo-restrict.service_ is configured to use `RestrictAddressFamilies=AF_UNIX AF_NETLINK`.
The service is activated by _echo-restrict.socket_.

```
$ grep Listen ~/.config/systemd/user/echo-restrict.socket
ListenStream=127.0.0.1:9000
```

To try it out, start the socket

```
$ systemctl --user start echo-restrict.socket
```

and test the echo server with __socat__.

```
$ echo hello | socat - tcp4:127.0.0.1:9000
hello
$
```

### The need for a separate service for creating the user namespace

Let us consider the situaition when systemd starts the systemd user services for
a user directly after a reboot.  If lingering is enabled for a user and the user is not logged,
the first started Podman systemd user service will notice that the Podman user namespace is missing
and will thus try to create it. This normally succeeds, but when RestrictAddressFamilies is used it fails.

The reason is that using RestrictAddressFamilies in an unprivileged systemd user service
 implies [`NoNewPrivileges=yes`](https://www.freedesktop.org/software/systemd/man/systemd.exec.html#NoNewPrivileges=), which prevents __/usr/bin/newuidmap__ and __/usr/bin/newgidmap__
from running with elevated privileges. Podman executes __newuidmap__ and __newgidmap__  to set up the user namespace.

The executables normally run with elevated privileges, as they need to
perform operations not available to an unprivileged user. These capabalities are

```
$ getcap /usr/bin/newuidmap
/usr/bin/newuidmap cap_setuid=ep
$ getcap /usr/bin/newgidmap
/usr/bin/newgidmap cap_setgid=ep
$
```

Setting up the user namespace only needs to be done once after each reboot because the user namespace will be reused for all other invocations of Podman.

This means that a service with RestrictAddressFamilies or   NoNewPrivileges=yes must not be the first started
Podman systemd user service for a user.

Any service using  `RestrictAddressFamilie` or `NoNewPrivileges=yes` must therefore depend on some other service that is responsible for setting up the user namespace.
The unit _echo-restrict.service_ depends on _podman-usernamespace.service_:

```
$ grep podman-usernamepsace.servic echo-restrict.service
After=podman-usernamepsace.service
BindTo=podman-usernamespace.service
```

The service _podman-usernamepsace.service_ is a Type=oneshot service that executes `podman unshare /bin/true`. The command is normally used for other
things, but a side effect of the command is that it sets up the user namespace.

Instead of using  _podman-usernamespace.service_, another solution could have been to create a dependency on a systemd user service that performs a container image pull
(i.e `ExecStart=/usr/bin/podman pull ghcr.io/eriksjolund/socket-activate-echo:latest`)

> **Note**
> Currently __runc__ supports `RestrictAddressFamilies=AF_UNIX AF_NETLINK`, but the number of socket-activated sockets are limited to max 2 (see bug: https://github.com/opencontainers/runc/issues/3488)

> **Note**
> At the time of this writing, __crun__ does not support `RestrictAddressFamilies=AF_UNIX AF_NETLINK` (see feature request: https://github.com/containers/crun/issues/929)

Verifying that the Podman restriction `RestrictAddressFamilies=AF_UNIX AF_NETLINK` works as expected:
If we would use `--pull=always` instead of `--pull=never` in _echo-restrict.service_, the service fails as expected because
Podman is blocked from establishing connections to the container registry.

__journalctl__ would then show such error messages

```
$ journalctl --user -xe -u echo.service | grep -A2 "Trying to pull" | tail -3
May 26 10:09:54 asus podman[28272]: Trying to pull ghcr.io/eriksjolund/socket-activate-echo:latest...
May 26 10:09:54 asus podman[28272]: Error: initializing source docker://ghcr.io/eriksjolund/socket-activate-echo:latest: pinging container registry ghcr.io: Get "https://ghcr.io/v2/": dial tcp 140.82.121.34:443: socket: address family not supported by protocol
May 26 10:09:54 asus systemd[10686]: test.service: Main process exited, code=exited, status=125/n/a
$
```

### Socket activate an Apache HTTP server with systemd-socket-activate

Instead of setting up a systemd service to test out socket activation, an alternative is to use the command-line tool [__systemd-socket-activate__](https://www.freedesktop.org/software/systemd/man/systemd-socket-activate.html#).

As an example let us use the container image [ghcr.io/eriksjolund/socket-activate-httpd](https://github.com/eriksjolund/socket-activate-httpd/pkgs/container/socket-activate-httpd)
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

### Note about SELinux

> **Note**
> If your computer is running SELinux, you need to have [__container-selinux 2.186.0__](https://github.com/containers/container-selinux) or newer installed.
> If container socket activation via Podman does not work and you are using an older version of
> container-selinux, add `--security-opt label=disable` to `podman run` as a work around.

### Note about running systemd in container

> **Note** There is currently no way to pass a socket-activated socket into a container when the container is running systemd.
> In other words, the executable __/usr/lib/systemd/systemd__ does not support socket activation.
> (There is a feature request related to it https://github.com/systemd/systemd/issues/17764)

### Note about Windows and macOS

> **Note** Socket activation via the Podman API service is not supported. If you are using Podman on Windows or macOS
> you will not be able to use socket activation for sockets on your host because the Podman on the host is using
> a Podman API service from a Linux VM.

### Left over blog text

Running a web server container is one of the more common uses for Podman. Normally you
would need to publish the ports that need to be open by providing the option `--publish` (`-p`) to `podman run`.
When running rootless Podman you also need to be aware that the network traffic is processed
by the user space application __slirp4netns__ which comes with a performance penalty.

You might be surprised to hear that it's now possible to run a web server container with rootless Podman and
get native network throughput! Even more surprising is that the __--network=none__ option can be given to disable the network.
There is also no need to publish ports.

The new way to run a network server container with Podman is to use socket activation provided by systemd.
Not all software daemons support socket activation but it's getting more popular.
For instance Apache HTTP server, MariaDB, DBUS, PipeWire, Gunicorn, CUPS all have socket activation support.
