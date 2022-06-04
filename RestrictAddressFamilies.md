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

Let's see how we could use __RestrictAddressFamilies__  for the socket-activated echo server.
If the `--pull=never` option is added to `podman run`, the echo server container will continue to work even with
the very restricted setting

```
RestrictAddressFamilies=AF_UNIX AF_NETLINK
```

All use of the system call `socket()` is then disallowed except for AF_UNIX sockets and AF_NETLINK
sockets.

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
a user directly after a reboot.  If lingering is enabled for a user and the user is not logged in,
the first started Podman systemd user service will notice that the Podman user namespace is missing
and will thus try to create it. This normally succeeds, but when RestrictAddressFamilies is used it fails.

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
$ grep podman-usernamespace.service echo-restrict.service
After=podman-usernamepsace.service
BindTo=podman-usernamespace.service
```

The service _podman-usernamepsace.service_ is a `Type=oneshot` service that executes `podman unshare /bin/true`. That
command is normally used for other things, but a side effect of the command is that it sets up the user namespace.

Instead of using _podman-usernamespace.service_, another solution could have been to create a dependency on
a systemd user service that performs a container image pull
(i.e `ExecStart=/usr/bin/podman pull ghcr.io/eriksjolund/socket-activate-echo:latest`)

> **Note**
> Currently __runc__ supports `RestrictAddressFamilies=AF_UNIX AF_NETLINK`, but the number of socket-activated
> sockets are limited to max 2 (see bug: https://github.com/opencontainers/runc/issues/3488)

> **Note**
> At the time of this writing, __crun__ does not support `RestrictAddressFamilies=AF_UNIX AF_NETLINK`
> (see feature request: https://github.com/containers/crun/issues/929)

Verifying that the Podman restriction `RestrictAddressFamilies=AF_UNIX AF_NETLINK` works as expected:
If we would use `--pull=always` instead of `--pull=never` in _echo-restrict.service_, the service fails as
expected because Podman is blocked from establishing connections to the container registry.

__journalctl__ would then show such error messages

```
$ journalctl --user -xe -u echo.service | grep -A2 "Trying to pull" | tail -3
May 26 10:09:54 asus podman[28272]: Trying to pull ghcr.io/eriksjolund/socket-activate-echo:latest...
May 26 10:09:54 asus podman[28272]: Error: initializing source docker://ghcr.io/eriksjolund/socket-activate-echo:latest: pinging container registry ghcr.io: Get "https://ghcr.io/v2/": dial tcp 140.82.121.34:443: socket: address family not supported by protocol
May 26 10:09:54 asus systemd[10686]: test.service: Main process exited, code=exited, status=125/n/a
$
```