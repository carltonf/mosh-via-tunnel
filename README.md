# Mosh over TCP

*Subtitle:* How to forward UDP traffic over TCP tunnel.

At my home, the Internet connections to hosts abroad may be unstable. I
usually uses a proxy for most sites, however ssh connections over proxy get
lost quite often and I have to manually reconnect. Mosh is a good tool in such
scenario but the UDP traffic can't be tunnelled directly through proxies. This
custom wrapper aims to workaround this problem. [^safety]


# Prerequisites

## mosh-client

There is no native Windows mosh port, other than `Cygwin` port is available.
I'm using the mosh-client from `MobaXterm`, a `Cygwin` derivative.

## Relay Tools

I've used `socat`[^fn_socat_win], ssh tunnel with `autossh` and eventually
settles down on `gost`. If one is using mosh-client solely on Linux hosts, no
special relay tools are [needed][sca55].

`gost` are more versatile and its syntax is modern. [^fn_goproxy]

## CLI Template Tool

This is optional and only needed when I want to put my code into source
control without also committing secrets about my proxy and remote hosts. 

After some search, I settle down on Mustache.

# How Mosh works

The name `mosh` actually only refers to a help script, which mainly does the
following two steps:

1. ssh into the remote machine to start mosh-server. mosh-server UDP port and
   session key is returned. This ssh connection will close *immediately*.
2. Start local mosh-client using the session key to establish a UDP connection
   to the remote. 

The step 1 is the normal ssh connection, i.e. a TCP connection. The second one
is a UDP connection, which this wrapper aims to forward.

It worths mentioning that mosh-server will only allow **the same** client to
reconnect. So if corresponding mosh-clients are gone, there is no hope to
reuse old mosh-servers and one has to explicitly *kill* them off. (*TIP* Use
combinations of `pgrep` and `netstat` to look for ports' owner process.)

# How this script `mosh-gost` works

**Read the code ;P**[^todo]

## Server side

We need to set up server to help forward UDP traffic. The following uses a
`systemd` service and will set up an Internet-facing `Shadowsocks` service and
an loopback device only relay service.

Start a gost port relay service (`/usr/local/lib/systemd/system/gost@.service`):

```ini
[Unit]
Description=Gost for %I
After=network.target
Wants=network.target

[Service]
Type=simple
CapabilityBoundingSet=CAP_NET_BIND_SERVICE
ExecStart=/usr/local/bin/gost -C /usr/local/etc/gost/%i.json
Restart=on-failure
RestartPreventExitStatus=1

[Install]
WantedBy=multi-user.target
```

Gost configuration files are stored at `/usr/local/etc/gost/`, here is the
relay configuration (in Mustache template form)[^fn_mustache]:

```json
{
    "Debug": true,
    "Retries": 0,
    "ServeNodes": [
        "ss://{{encryption}}:{{& password}}@{{public_ip}}:1080",
        "relay://127.0.0.1:1081"
    ],
    "ChainNodes": [ ]
}
```

*NOTE on 'ss:'* the extra `Shadowsocks` service is not functionally necessary,
I set this up for security reason, not wanting an unguarded relay service
exposed to the Internet. The explicit IP settings are for similar reasons.

# What else I have tried or thought about

## TAP device

Adding extra tap/tun devices requires admin privilege. However, if more UDP
connections are required, this might be the only elegant solution.[^android]

## autossh

I've used autossh for quite a while (mainly for reverse ssh tunnel) and
tweaks many options. However there are always some cases when autossh fails to
reconnect automatically.[^autossh_opts]

## Mosh alternatives

I searched for quite a while but the only one I find similar to mosh is
[EternalTerminal][mgiE]. `EternalTerminal` uses TCP as default connection type, so probably works better with proxy.

However it's not ubiquitously available like mosh, requiring extra repos and dependence. Also from some comments, it's less responsive than mosh.


[//]: # (------------------ Refs and Footnotes ----------------------------)

[^safety]: UDP-over-TCP might not be safe
([quote](https://superuser.com/a/53109/504415)): " TCP streams are not
guaranteed to preserve message boundaries, so a single UDP datagram may be
split in parts, breaking any protocol." While this may lead to many malformed
UDP packets and even complete failure of forwarding, but this hasn't been a
problem in my use case.

[sca55]: https://superuser.com/a/53109/504415

[^android]: Later I've successfully setup an Android phone as a host for
mosh-client with proxy. An SSH Server (Termux) is started on the phone and I
later ssh into the phone, install mosh and use the mosh normally. The proxy
app (Clash for Android) has an option for routing specific applications. 
Termux (or other ssh server app) and all programs running within it will be
routed through the proxy. In fact, the proxy app has set a tun device.

[^fn_goproxy]: Also [goproxy](https://github.com/snail007/goproxy)

[^fn_socat_win]: socat has good [Windows
port](https://sourceforge.net/projects/unix-utils/files/socat/1.7.3.2/). I
found this [doc](http://zarb.org/~gc/html/udp-in-ssh-tunneling.html) useful.
At the time of writing, I already forget the commands I use to make socat
UDP-over-TCP working.

[^todo]: write it if we get time.

[^autossh_opts]: These options helped in maintaining long-lived ssh tunnel
`ExitOnForwardFailure` and `ServerAliveCountInterval` and
`ServerAliveCountMax`. However these don't seem enough.

[mgiE]: https://mistertea.github.io/EternalTerminal/ 

[^fn_mustache]: Simply replace the form `{{...}}` with your own settings.