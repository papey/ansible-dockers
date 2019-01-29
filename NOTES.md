# Some notes about this experiment

Notes i will used to make a blog post.

## Brutal way

The most brutal way is to isolate mandatory stuff for each daemon :

- bridge : containers will be attached to this specific bridge
- data-dir : where container images are stored
- exec-dir : where container states are stored
- socket : where docker listen for command and events controlling containers (start, stop, restart, status...)
- pidfile : file containing pid of running process (here it's the docker deamon)

The cheapest way to test this is **systemd**.

### Bridge

For the bridge part, systemd-networkd is the best option. In order to create
a bridge, two files are needed :

- a netdev file, containing interface description (name, kind)
- a network file, containing interface configuration (ip, gateway, dns)

#### Netdev

The _awesomebridge.netdev_ file, placed under _/etc/systemd/network_

```
[NetDev]
Name=awesomebridge
Kind=bridge
```

#### Network

And the _awesomebridge.network_ file, placed under _/etc/systemd/network_

```
[Match]
Name=awesomebridge.network

[Network]
Address=172.18.0.1/16
```

After that, trigger a systemd-networkd restart

```shell
systemctl restart systemd-networkd
```

New bridge appears !

### Data-dir, exec-dir, socket & pidfile

All those arguments are files or directories created by the docker daemon if
needed. Passing specific arguments for each docker deamons is quite easy using
the 'argument service' type offered by systemd.

```
...
[Service]
Type=notify
# the default is not to use systemd for cgroups because the delegate issues still
# exists and systemd currently does not support the cgroup feature set required
# for containers run by docker
ExecStart=/usr/bin/dockerd \
    --bridge=%i \
    --data-root=/var/lib/docker/%i.data \
    --exec-root=/var/lib/docker/%i.exec \
    --host=unix:///var/run/docker-%i.sock \
    --pidfile=/var/run/docker-%i.pid
ExecReload=/bin/kill -s HUP $MAINPID
TimeoutSec=0
RestartSec=2
Restart=always
```

%i will be replaced on the fly by systemd with the argument passed to the
unit. Example when calling :

```
systemctl start docker@alice.service
```

The actual command will be :

```
/usr/bin/dockerd \
    --bridge=alice \
    --data-root=/var/lib/docker/alice.data \
    --exec-root=/var/lib/docker/alice.exec \
    --host=unix:///var/run/docker-alice.sock \
    --pidfile=/var/run/docker-alice.pid
```

### Now, how can I interact with one or another deamon ?

Just by using the desired socket.

For example, using Bob's deamon

```shell
docker -H unix:///var/run/docker-bob.sock image pull alpine
```

```shell
docker -H unix:///var/run/docker-bob.sock image ls
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
alpine              latest              3f53bb00af94        5 weeks ago         4.41MB
```

```shell
docker -H unix:///var/run/docker-bob.sock run -i -t alpine echo hello
hello
```

```shell
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS                      PORTS               NAMES
d376d05271ac        alpine              "echo hello"        47 seconds ago      Exited (0) 47 seconds ago                       epic_torvalds
```

What about Alice then ?

```
docker -H unix:///var/run/docker-alice.sock image ls
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
```

It's empty, yay !

### Let's dig, just for fun

#### Networks

Bob runs a web server :

```shell
docker -H unix:///var/run/docker-bob.sock run --rm -d -p 80 nginx
6e9aed2cee9857eaa060829449e2d8b3b7acd5f7edaedb893b0ff56cc908e460
```

```shell
docker -H unix:///var/run/docker-bob.sock container ls
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS                   NAMES
6e9aed2cee98        nginx               "nginx -g 'daemon of…"   20 seconds ago      Up 19 seconds       0.0.0.0:32769->80/tcp   upbeat_babbage
```

Is it working ? Yes

```shell
curl -I http://127.0.0.1:32769
HTTP/1.1 200 OK
Server: nginx/1.15.8
Date: Mon, 28 Jan 2019 15:56:04 GMT
Content-Type: text/html
Content-Length: 612
Last-Modified: Tue, 25 Dec 2018 09:56:47 GMT
Connection: keep-alive
ETag: "5c21fedf-264"
Accept-Ranges: bytes
```

And by direct IP access

```shell
docker -H unix:///var/run/docker-bob.sock container inspect --format='{{range .NetworkSettings.Networks }}{{ .IPAddress }}{{ end }}' upbeat_babbage
172.18.0.2
```

```shell
curl -I 172.18.0.2:80
HTTP/1.1 200 OK
Server: nginx/1.15.8
Date: Mon, 28 Jan 2019 16:04:21 GMT
Content-Type: text/html
Content-Length: 612
Last-Modified: Tue, 25 Dec 2018 09:56:47 GMT
Connection: keep-alive
ETag: "5c21fedf-264"
Accept-Ranges: bytes
```

Can Alice access Bob's nginx ?

```shell
/ # apk add curl
fetch http://dl-cdn.alpinelinux.org/alpine/v3.8/main/x86_64/APKINDEX.tar.gz
fetch http://dl-cdn.alpinelinux.org/alpine/v3.8/community/x86_64/APKINDEX.tar.gz
(1/5) Installing ca-certificates (20171114-r3)
(2/5) Installing nghttp2-libs (1.32.0-r0)
(3/5) Installing libssh2 (1.8.0-r3)
(4/5) Installing libcurl (7.61.1-r1)
(5/5) Installing curl (7.61.1-r1)
Executing busybox-1.28.4-r2.trigger
Executing ca-certificates-20171114-r3.trigger
OK: 6 MiB in 18 packages
/ # curl -I http://172.18.0.2:80
HTTP/1.1 200 OK
Server: nginx/1.15.8
Date: Mon, 28 Jan 2019 16:06:51 GMT
Content-Type: text/html
Content-Length: 612
Last-Modified: Tue, 25 Dec 2018 09:56:47 GMT
Connection: keep-alive
ETag: "5c21fedf-264"
Accept-Ranges: bytes
```

Is this behavior ok ? Let's look at the route table

```shell
ip route show
default via 10.0.2.2 dev eth0
10.0.2.0/24 dev eth0 proto kernel scope link src 10.0.2.15
172.18.0.0/16 dev bob proto kernel scope link src 172.18.0.1
192.142.42.0/24 dev eth1 proto kernel scope link src 192.142.42.42
```

When curling from Alice container, kernel sees a packet to the destination
172.18.0.2, 172.18.0.0/16 route is in the routing table, without firewalling
rules in the forward table, everything from Alice to Bob's subnet will be
redirected to Bob's bridge and then the destination container.

TL;DR : this is the correct default behavior.

#### Sharing data

What about sharing the same data-dir ? What is Bob and Alice shares the same data-root ?

With Bob's daemon running :

```shell
Error starting daemon: error while opening volume store metadata database: timeout
```

Without Bob's daemon running :

```shell
docker -H unix:///var/run/docker-alice.sock image ls -a
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
nginx               latest              42b4762643dc        5 days ago          109MB
alpine              latest              3f53bb00af94        5 weeks ago         4.41MB
```

Looks like there is some lockup mechanism.

Let's dig inside docker-ce source code (git grep <3). In the file
_components/engine/volume/service/store.go_ :

```go
		var err error
		vs.db, err = bolt.Open(filepath.Join(volPath, "metadata.db"), 0600, &bolt.Options{Timeout: 1 * time.Second})
		if err != nil {
			return nil, errors.Wrap(err, "error while opening volume store metadata database")
		}
```

bolt, wtf is this ?

```go
	bolt "go.etcd.io/bbolt"
```

##### Bbolt

[Bbolt](https://github.com/etcd-io/bbolt) is a key/value store engine.
Usefull when a full database is not needed.

And here [comes the lock](https://github.com/etcd-io/bbolt#opening-a-database) !

```
Please note that Bolt obtains a file lock on the data file so multiple
processes cannot open the same database at the same time. Opening an already
open Bolt database will cause it to hang until the other process closes it.
To prevent an indefinite wait you can pass a timeout option to the Open()
function:
```

```go
db, err := bolt.Open("my.db", 0600, &bolt.Options{Timeout: 1 * time.Second})
```

And inside docker-ce code :

```go
		vs.db, err = bolt.Open(filepath.Join(volPath, "metadata.db"), 0600, &bolt.Options{Timeout: 1 * time.Second})
```

This explains why it's not possible to use the same data-root for multiple
docker deamons. This a not like a relational DB, lock mechanisms are
mandatory to ensure integrity of every piece of data.