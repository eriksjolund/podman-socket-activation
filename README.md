# Podman socket activation

Running a web server container is one of the more common uses for Podman. Normally you
would need to publish the ports that need to be open by providing the option `--publish` (`-p`) to `podman run`.
When running rootless Podman you also need to be aware that the network traffic is processed
by the user space application __slirp4netns__ which comes with a performance penalty.

You might be surprised to hear that it's now possible to run a web server container with rootless Podman and
get native network throughput speed! Even more sur   prising is that the __--network=none__ option can be given to disable the network.
There is also no need to publish ports.

The new way to run a containerized network server is to use socket activation provided by systemd.

Socket activation conceptually works by having systemd create a socket (e.g. TCP, UDP or Unix socket). As soon as
a client connects to the socket, systemd will start the systemd service that is configured for the socket.
The newly started program inherits the open file descriptor of the socket and can accept the incoming connection.
The new feature is that Podman can now pass such a socket to the container.

Not all software daemons support socket activation but it's getting more popular.
For instance Apache HTTPD, MariaDB, Gunicorn, Pipewire, CUPS, DBUS all support socket activation support.

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
$ export DOCKER_HOST=$XDG_RUNTIME_DIR/podman/podman.sock
$ docker-compose up
```

Podman has supported socket activation of its API service for a long time.
More recently, in version 3.4.0, Podman received support for another type of socket activation, namely, socket action
of containers. Such socket activation can be used in the systemd services that are generated with
the command `podman generate systemd --new --name CTR`.

I created a container image [__ghcr.io/eriksjolund/socket-activate-echo__](https://github.com/eriksjolund/socket-activate-echo/pkgs/container/socket-activate-echo)
that contains a simple echo server that supports socket activation. (The echo server currently has a bit limited functionality. It was written for the
sole purpose of demonstrating socket activation. Source code can found in the [GitHub repo](https://github.com/eriksjolund/socket-activate-echo/))

Here is an example. Start the echo server sockets

```
git clone https://github.com/eriksjolund/socket-activate-echo.git
mkdir -p ~/.config/systemd/user
cp -r socket-activate-echo/systemd/echo* ~/.config/systemd/user
systemctl --user daemon-reload
systemctl --user start echo@demo.socket
```

List the listening sockets. In this example they all listen on port 3000.

```
$ ss -lnp | grep 3000
udp   UNCONN 0      0                                       127.0.0.1:3000             0.0.0.0:*    users:(("systemd",pid=2516,fd=33))
udp   UNCONN 0      0                                           [::1]:3000                [::]:*    users:(("systemd",pid=2516,fd=35))
tcp   LISTEN 0      4096                                    127.0.0.1:3000             0.0.0.0:*    users:(("systemd",pid=2516,fd=28))
tcp   LISTEN 0      4096                                        [::1]:3000                [::]:*    users:(("systemd",pid=2516,fd=34))
v_str LISTEN 0      0                                               *:3000                   *:*    users:(("systemd",pid=2516,fd=36))
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

An echo server does not need the ability to establish outgoing connections. It just needs to accept incoming connections from clients.
The command-line option __--network=none__ could therefore be used to prevent the container from establishing outgoing connections.

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

Try establishing an outgoing connection

```
$ podman exec -t echo-demo curl https://podman.io
curl: (6) Could not resolve host: podman.io
$
```

A good security practice is to run programs with as few privileges as possible. In case the program would get hacked, the intruder would only
gain access to the privileges at hand.

Note: in case a network server needs to establish outgoing connections, remove the __--network=none__ option.

The communication in the "socket-activation" socket has native network throughput speed. Other network traffic needs to pass through slirp4netns and gets the performance penalty that comes with it.

Note: if your machine is running SELinux, you need to have __container-selinux  2.183.0__ or newer installed.
If you are using an older version of container-selinux and it does not work, add `--security-opt label=disable` to `podman run` as a work around.

More examples of socket-activated containers

* Apache HTTPD, see https://github.com/eriksjolund/socket-activate-httpd
* MariaDB, see https://github.com/eriksjolund/mariadb-podman-socket-activation
