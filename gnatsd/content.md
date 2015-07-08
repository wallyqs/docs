# Supported tags and respective `Dockerfile` links

-	[`0.5.6`, `0.6.0`, `latest` (*gnatsd/Dockerfile*)](https://github.com/nats-io/gnatsd/blob/v0.6.0/docker/Dockerfile)

For more information about this image and its history, please see the [relevant manifest file (`library/gnatsd`)](https://github.com/docker-library/official-images/blob/master/library/gnatsd) in the [`docker-library/official-images` GitHub repo](https://github.com/docker-library/official-images).


# What is NATS?

NATS is an open source, high-performance, lightweight, always up cloud messaging system. Gnatsd is the high-performance server written in the Go language.

%%LOGO%%

NATS is comprised of the server (Gnatsd, which is written in Go), and a wide variety of high-fidelity clients for many languages. These can all be found on the NATS.io Github repo:

>[NATS.io Github] (https://github.com/nats-io)

NATS was created by Derek Collison, founder and CEO of Apcera, who has spent 20+ years designing, building, and using publish-subscribe messaging systems. 

Unlike traditional Enterprise Messaging Systems (EMS), NATS has an always on ‘dial tone’, and does whatever is required to remain available and up. This provides a great basis for building modern, reliable, and scalable cloud and distributed systems. 

NATS is designed to be highly scalable, and is capable of sending over 6 Million messages per second, providing brokered throughput well in excess of other messaging projects.

> [NATS.io](http://www.nats.io)

# How to use this image

```
docker run --name my_gnatsd -d apcera/gnatsd
```

This image includes `EXPOSE 4222 8333` (the client and monitor port respectively), so standard container linking will make it automatically available to the linked containers. You can also expose this via the `-p` or `-P` flags to `docker run`.

## Example of NATS Cluster usage

### Configuration for setting up a cluster on 3 different servers

#### gnatsd A

```
# Cluster Server A

port: 7222

cluster {
  host: '0.0.0.0'
  port: 7244

  routes = [
    nats-route://192.168.59.103:7246
    nats-route://192.168.59.103:7248
  ]
}
```

#### gnatsd B

```
# Cluster Server B

port: 8222

cluster {
  host: '0.0.0.0'
  port: 7246

  routes = [
    nats-route://192.168.59.103:7244
    nats-route://192.168.59.103:7248
  ]
}
```

#### gnatsd C

```
# Cluster Server C

port: 9222

cluster {
  host: '0.0.0.0'
  port: 7248

  routes = [
    nats-route://192.168.59.103:7244
    nats-route://192.168.59.103:7246
  ]
}
```

#### Starting the containers

Then on each one of your servers, you should be able to start the gnatsd image as follows:

```
docker run -it -p 0.0.0.0:7222:7222 -p 0.0.0.0:7244:7244 --rm -v $(pwd)/conf/gnatsd-A.conf:/tmp/cluster.conf apcera/gnatsd -c /tmp/cluster.conf -p 7222 -D -V
```

```
docker run -it -p 0.0.0.0:8222:8222 -p 0.0.0.0:7246:7246 --rm -v $(pwd)/conf/gnatsd-B.conf:/tmp/cluster.conf apcera/gnatsd -c /tmp/cluster.conf -p 8222 -D -V
```

```
docker run -it -p 0.0.0.0:9222:9222 -p 0.0.0.0:7248:7248 --rm -v $(pwd)/conf/gnatsd-C.conf:/tmp/cluster.conf apcera/gnatsd -c /tmp/cluster.conf -p 9222 -D -V
```

### Alternative cluster setup

In this scenario:

- We bring up A and get its IP Address (`nats-route://192.168.59.103:7244`)
- Then create B and then use address of A in its configuration.
- Get the address of B `nats-route://192.168.59.104:7246` and create C and use the addresses of A and B.

First, we create the Node A and start up a gnatsd server with the following config:

```
# Cluster Server A

port: 4222

cluster {
  host: '0.0.0.0'
  port: 7244

}
```

Then run:

```
docker run -it -p 0.0.0.0:4222:4222 -p 0.0.0.0:7244:7244 --rm -v $(pwd)/conf/gnatsd-A.conf:/tmp/cluster.conf apcera/gnatsd -c /tmp/cluster.conf -p 4222 -D -V
```

Next, it is time to create the next node.  Notice that the first node has ip:port as `192.168.59.103:7244` so we add this to the `routes` configuration as follows:

```
# Cluster Server B

port: 4222

cluster {
  host: '0.0.0.0'
  port: 7244

  routes = [
    nats-route://192.168.59.103:7244
  ]
}
```

Then start server B:

```
docker run -it -p 0.0.0.0:4222:4222 -p 0.0.0.0:7244:7244 --rm -v $(pwd)/conf/gnatsd-A.conf:/tmp/cluster.conf apcera/gnatsd -c /tmp/cluster.conf -p 4222 -D -V
```

Finally, we create another Node C.  We now know the routes of A and B
so we can add those to the configuration:

```
# Cluster Server C

port: 4222

cluster {
  host: '0.0.0.0'
  port: 7244

  routes = [
    nats-route://192.168.59.103:7244
    nats-route://192.168.59.104:7244
  ]
}
```

Then start it:

```
docker run -it -p 0.0.0.0:4222:4222 -p 0.0.0.0:7244:7244 --rm -v $(pwd)/conf/gnatsd-C.conf:/tmp/cluster.conf apcera/gnatsd -c /tmp/cluster.conf -p 9222 -D -V
```

Now, the following should work:  make a subscription to Node A then publish to Node C.
You should be able to to receive the message without any issues.
