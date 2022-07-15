# Currently a draft
## Restrict Podman with _RestrictAddressFamilies_

When using Podman in a systemd service, the __systemd__ directive
[__RestrictAddressFamilies__](https://www.freedesktop.org/software/systemd/man/systemd.exec.html#RestrictAddressFamilies=)
can be used to restrict Podman's access to sockets. The restriction only concerns the use of the system call __socket()__,
which means that socket-activated sockets are unaffected by the directive.

Containers that only need internet access via socket-activated sockets can still be run by Podman when
systemd is configured to restrict Podman's ability to use the system call `socket()` for the AF_INET and AF_INET6
socket families. Of course, Podman would then be blocked from pulling down any container images so the container
image needs to be present beforehand.

Let's see how we could use __RestrictAddressFamilies__  for a socket-activated echo server.
If the `--pull=never` option is added to `podman run`, the echo server container will continue to work even with
the very restricted setting

```
RestrictAddressFamilies=AF_UNIX AF_NETLINK
```

All use of the system call `socket()` is then disallowed except for AF_UNIX sockets and AF_NETLINK
sockets.

In case there would be a security vulnerability in Podman, conmon or runc, this configuration limits
the possibilities an intruder has to launch attacks on other PCs on the network.

Let's try out [socket-activate-echo](https://github.com/eriksjolund/socket-activate-echo/pkgs/container/socket-activate-echo),
a simple echo server container that supports socket activation.

### Create the Systemd unit files

Create the container

```
$ podman pull -q ghcr.io/eriksjolund/socket-activate-echo
$ podman create --rm --name echo --network none --pull=never ghcr.io/eriksjolund/socket-activate-echo
```

Generate the systemd service unit

```
$ mkdir -p ~/.config/systemd/user
$ podman generate systemd --name --new echo > ~/.config/systemd/user/restricted-echo.service
```

Add the two lines
```
RestrictAddressFamilies=AF_UNIX AF_NETLINK
NoNewPrivileges=yes
```
under the line `[Service]` with the program __sed__ (or just use an editor)

```
$ sed -i '/\[Service\]/a \
RestrictAddressFamilies=AF_UNIX AF_NETLINK\
NoNewPrivileges=yes' ~/.config/systemd/user/echo.service
```

Create the file _~/.config/systemd/user/restricted-echo.socket_ with
this file contents

```
[Unit]
Description=restricted echo server
[Socket]
ListenStream=127.0.0.1:9000

[Install]
WantedBy=default.target
```

Add the two lines
```
After=podman-usernamespace.service
BindTo=podman-usernamespace.service
```
under the line `[Unit]` with the program __sed__ (or just use an editor)

```
$ sed -i '/\[Unit\]/a \
After=podman-usernamespace.service
BindTo=podman-usernamespace.service' ~/.config/systemd/user/restricted-echo.service
```

Create the file _~/.config/systemd/user/podman-usernamespace.service_ with this file contents

```
[Unit]
Description=podman-usernamespace.service

[Service]
Type=oneshot
Restart=on-failure
TimeoutStopSec=70
ExecStart=/usr/bin/podman unshare /bin/true
RemainAfterExit=yes

[Install]
WantedBy=default.target
```

After editing the unit files, systemd needs to reload it's configuration

```
$ systemctl --user daemon-reload
```

Start the socket unit

```
$ systemctl --user start restricted-echo.socket
```

Test the echo server with the program __socat__

```
$ echo hello | socat - tcp4:127.0.0.1:9000
hello
$
```

The echo server works as expected! It replies _"hello"_ after receiving the text _"hello"_.

Currently Podman does not pull any container when the container is started.

Modify the service unit so that Podman always pulls the container image

```
$ grep -- --pull= .config/systemd/user/echo.service
	--pull=never ghcr.io/eriksjolund/socket-activate-echo
$ sed -i s/pull=never/pull=always/ .config/systemd/user/echo.service
$ grep -- --pull= .config/systemd/user/echo.service
	--pull=always ghcr.io/eriksjolund/socket-activate-echo
```

After editing the unit file, systemd needs to reload it's configuration

```
$ systemctl --user daemon-reload
```

Stop the service

```
$ systemctl --user stop restricted-echo.service
```

Test the echo server with the program __socat__

```
$ echo hello | socat - tcp4:127.0.0.1:9000
```

As expected the service fails because Podman is blocked from establishing connections to the container registry.

__journalctl__ shows such this error  message

```
$ journalctl --user -xe -u echo.service | grep -A2 "Trying to pull" | tail -3
May 26 10:09:54 asus podman[28272]: Trying to pull ghcr.io/eriksjolund/socket-activate-echo:latest...
May 26 10:09:54 asus podman[28272]: Error: initializing source docker://ghcr.io/eriksjolund/socket-activate-echo:latest: pinging container registry ghcr.io: Get "https://ghcr.io/v2/": dial tcp 140.82.121.34:443: socket: address family not supported by protocol
May 26 10:09:54 asus systemd[10686]: test.service: Main process exited, code=exited, status=125/n/a
$
```

### The need for a separate service for creating the user namespace

Let us consider the situaition when systemd starts the systemd user services for
a user directly after a reboot.  If lingering is enabled for the user and the user is not logged in,
the first started Podman systemd user service will notice that the Podman user namespace is missing
and will thus try to create it. This normally succeeds, but when RestrictAddressFamilies is used
together with rootless Podman it fails.

The reason is that using RestrictAddressFamilies in an unprivileged systemd user service
 implies [`NoNewPrivileges=yes`](https://www.freedesktop.org/software/systemd/man/systemd.exec.html#NoNewPrivileges=),
which prevents __/usr/bin/newuidmap__ and __/usr/bin/newgidmap__ from running with elevated privileges.
Podman executes __newuidmap__ and __newgidmap__  to set up the user namespace. Both executables normally
run with elevated privileges, as they need to perform operations not available to an unprivileged user.
These capabalities are

```
$ getcap /usr/bin/newuidmap
/usr/bin/newuidmap cap_setuid=ep
$ getcap /usr/bin/newgidmap
/usr/bin/newgidmap cap_setgid=ep
$
```

Setting up the user namespace only needs to be done once because the created user namespace will be
reused for all other invocations of Podman. Services using `RestrictAddressFamilies` or `NoNewPrivileges=yes` can
be made to work by configuring them to start after a systemd user service that is responsible for setting
up the user namespace.

For instance, the unit _echo-restrict.service_ depends on _podman-usernamespace.service_:

```
$ grep podman-usernamespace.service ~/.config/systemd/user/echo-restrict.service
After=podman-usernamespace.service
BindTo=podman-usernamespace.service
```

The service _podman-usernamespace.service_ is a `Type=oneshot` service that executes `podman unshare /bin/true`. That
command is normally used for other things, but a side effect of the command is that it sets up the user namespace.

Instead of using _podman-usernamespace.service_, another solution could have been to create a dependency on
a systemd user service that performs a container image pull
(i.e `ExecStart=/usr/bin/podman pull ghcr.io/eriksjolund/socket-activate-echo:latest`)

> **Note**
> __runc__ supports `RestrictAddressFamilies=AF_UNIX AF_NETLINK` but before version 1.1.3,
> runc had a bug that limited the number of socket-activated sockets to max 2.

> **Note**
> At the time of this writing, __crun__ only has support for `RestrictAddressFamilies=AF_UNIX AF_NETLINK`
> in the main Git branch. The latest crun release 1.4.5 does not have the functionality.
