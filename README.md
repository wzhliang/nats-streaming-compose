Create a docker compose file to cluster three instances of nats-streaming server.  We use a docker custom network, which allows containers to refer to each other
by hostname.

Put this in a file named *docker-compose.yaml*:

```
version: '2'
networks:
  nats-net:
    ipam:
      config:
      - subnet: 172.25.0.0/16
services:
  nats-1:
    image: nats-streaming
    command: --config /server.cfg --cluster_id nats-1
    volumes:
    - ./seed.cfg:/server.cfg
    networks:
    - nats-net
    ports:
    - 4222:4222
    - 8222:8222
  nats-2:
    image: nats-streaming
    command: --config /server.cfg --cluster_id nats-2
    volumes:
    - ./notseed.cfg:/server.cfg
    networks:
    - nats-net
    ports:
    - 4223:4222
    - 8223:8222
  nats-3:
    image: nats-streaming
    command: --config /server.cfg --cluster_id nats-3
    volumes: 
    - ./notseed.cfg:/server.cfg 
    networks: 
    - nats-net 
    ports:
    - 4224:4222 
    - 8224:8222
```

Define a seed config, seed.cfg:

```
# Cluster seed node

listen: 0.0.0.0:4222
http: 8222

cluster {
  port 4248
}
```

Define a not-seed.cfg, notseed.cfg:

```
# Cluster not-a-seed node

listen: 0.0.0.0:4222
http: 8222

cluster {
  port 4248
  routes = [
    nats-route://nats-1:4248
  ]
}
```

Start the cluster, detaching from the terminal.


```bash
$ docker-compose -f docker-compose.yaml up -d
```

and have a look at the running containers


```bash
$ docker ps
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS                                            NAMES
dbd08f184c3e        nats-streaming      "/nats-streaming-serv"   About an hour ago   Up About an hour    0.0.0.0:4223->4222/tcp, 0.0.0.0:8223->8222/tcp   clustering_nats-2_1
c7cbb32a6977        nats-streaming      "/nats-streaming-serv"   About an hour ago   Up About an hour    0.0.0.0:4224->4222/tcp, 0.0.0.0:8224->8222/tcp   clustering_nats-3_1
11aa05775a03        nats-streaming      "/nats-streaming-serv"   About an hour ago   Up About an hour    0.0.0.0:4222->4222/tcp, 0.0.0.0:8222->8222/tcp   clustering_nats-1_1
```

Grab the container IPs.  You need [jq](https://stedolan.github.io/jq/) for this.

```bash
$ for i in $(docker  ps -q); do docker inspect $i | jq  -r '.[0].NetworkSettings | .Networks | .["clustering_nats-net"].IPAddress'; done 
172.25.0.3
172.25.0.4
172.25.0.2
```

Get logs on one of the containers (looks ok to me)

```bash
$ docker logs dbd
[1] 2016/12/20 21:59:51.714233 [INF] Starting nats-streaming-server[nats-2] version 0.3.4
[1] 2016/12/20 21:59:51.714439 [INF] Starting nats-server version 0.9.5
[1] 2016/12/20 21:59:51.714480 [INF] Starting http monitor on 0.0.0.0:8222
[1] 2016/12/20 21:59:51.714600 [INF] Listening for client connections on 0.0.0.0:4222
[1] 2016/12/20 21:59:51.714639 [INF] Server is ready
[1] 2016/12/20 21:59:51.715066 [INF] Listening for route connections on 0.0.0.0:4248
[1] 2016/12/20 21:59:51.994123 [INF] STAN: Message store is MEMORY
[1] 2016/12/20 21:59:51.994154 [INF] STAN: --------- Store Limits ---------
[1] 2016/12/20 21:59:51.994161 [INF] STAN: Channels:                  100 *
[1] 2016/12/20 21:59:51.994164 [INF] STAN: -------- channels limits -------
[1] 2016/12/20 21:59:51.994168 [INF] STAN:   Subscriptions:          1000 *
[1] 2016/12/20 21:59:51.994171 [INF] STAN:   Messages     :       1000000 *
[1] 2016/12/20 21:59:51.994230 [INF] STAN:   Bytes        :     976.56 MB *
[1] 2016/12/20 21:59:51.994235 [INF] STAN:   Age          :     unlimited *
[1] 2016/12/20 21:59:51.994237 [INF] STAN: --------------------------------
```

So are the instances really clustered?  How to know?  Try querying each admin server for connect_urls, which seems to show the cluster is assembled.  
Is it?  

```
$ for i in 2 3 4 ; do curl -s http://localhost:822${i}/varz | jq .connect_urls ; done
[
  "172.25.0.3:4222",
  "172.25.0.4:4222"
]
[
  "172.25.0.2:4222",
  "172.25.0.4:4222"
]
[
  "172.25.0.2:4222",
  "172.25.0.3:4222"
]
```
