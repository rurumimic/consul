# Consul

- [Learn](https://learn.hashicorp.com/consul)

## Development mode

```bash
brew tap hashicorp/tap
brew install hashicorp/tap/consul
brew upgrade hashicorp/tap/consul
```

### Start the agent

```bash
consul agent -dev
```

> **Note for OS X Users**
> Consul uses your hostname as the default node name. 
> If your hostname contains periods, DNS queries to that node will not work with Consul. 
> To avoid this, explicitly set the name of your node with the `-node` flag.

#### Discover datacenter members

```bash
consul members

Node        Address         Status  Type    Build  Protocol  DC   Segment
<Hostname>  127.0.0.1:8301  alive   server  1.9.1  2         dc1  <all>
```

DNS lookups:

```bash
dig @127.0.0.1 -p 8600 <Hostname>.node.consul
```

#### Stop the agent

```bash
consul leave

Graceful leave complete
```

---

## Service Discovery

### Define a service

```bash
mkdir ./consul.d
```

```bash
echo '{
  "service": {
    "name": "web",
    "tags": [
      "rails"
    ],
    "port": 80
  }
}' > ./consul.d/web.json
```

Restart the agent:

```bash
consul agent -dev -enable-script-checks -config-dir=./consul.d

2021-01-20T16:00:04.508+0900 [DEBUG] agent: Node info in sync
2021-01-20T16:00:04.508+0900 [DEBUG] agent: Service in sync: service=web
```

### Query services

#### DNS interface

```bash
dig @127.0.0.1 -p 8600 web.service.consul # NAME.service.consul
dig @127.0.0.1 -p 8600 web.service.consul SRV # entire address/port pair
dig @127.0.0.1 -p 8600 rails.web.service.consul # TAG.NAME.service.consul
```

#### HTTP API

```bash
curl http://localhost:8500/v1/catalog/service/web
curl 'http://localhost:8500/v1/health/service/web?passing'
```

### Update services

```bash
echo '{
  "service": {
    "name": "web",
    "tags": [
      "rails"
    ],
    "port": 80,
    "check": {
      "args": [
        "curl",
        "localhost"
      ],
      "interval": "10s"
    }
  }
}' > ./consul.d/web.json
```

```bash
consul reload

Check is now critical: check=service:web
```

```bash
dig @127.0.0.1 -p 8600 web.service.consul
```

---

## Service Mesh

### Start a Consul-unaware service

```bash
brew install socat
```

`socat` act as the "upstream" service.

```bash
socat -v tcp-l:8181,fork exec:"/bin/cat"
```

Verify it is working by using `nc` (netcat) to connect directly to the echo service on the correct port.

```bash
nc 127.0.0.1 8181
hello
hello
testing 123
testing 123
```

### Register the service and proxy with Consul

```bash
echo '{
  "service": {
    "name": "socat",
    "port": 8181,
    "connect": {
      "sidecar_service": {}
    }
  }
}' > ./consul.d/socat.json
```

```bash
consul reload
```

in another terminal, start the proxy:

```bash
consul connect proxy -sidecar-for socat
```

### Register a dependent service and proxy

```bash
echo '{
  "service": {
    "name": "web",
    "connect": {
      "sidecar_service": {
        "proxy": {
          "upstreams": [
            {
              "destination_name": "socat",
              "local_bind_port": 9191
            }
          ]
        }
      }
    }
  }
}' > ./consul.d/web.json
```

```bash
consul reload
```

Verify that you aren't able to connect to the socat service on port 9191:

```bash
nc 127.0.0.1 9191
```

Start the web proxy:

```bash
consul connect proxy -sidecar-for web
```

Try connecting to socat again on port 9191:

```bash
nc 127.0.0.1 9191
hello
hello
```

### Control communication with intentions

Create an intention to deny access from web to socat:

```bash
consul intention create -deny web socat
```

Fail:

```bash
nc 127.0.0.1 9191
```

Delete the intention:

```bash
consul intention delete web socat
```

Succeed:

```bash
nc 127.0.0.1 9191
hello
hello
```

---

## Consul KV

### Add data

```bash
consul kv put redis/config/minconns 1
consul kv put redis/config/maxconns 25
```

```bash
consul kv put -flags=42 redis/config/users/admin abcd1234
```

### Query data

```bash
consul kv get redis/config/minconns # 1
consul kv get -detailed redis/config/users/admin
```

```bash
consul kv get -recurse

redis/config/maxconns:25
redis/config/minconns:1
redis/config/users/admin:abcd1234
```

### Delete data

```bash
consul kv delete redis/config/minconns
consul kv delete -recurse redis
```

### Modify existing data

```bash
consul kv put foo bar
consul kv get foo # bar
consul kv put foo zip
consul kv get foo # zip
```

## Consul UI

In development mode, the UI is automatically enabled.

[http://localhost:8500/ui](http://localhost:8500/ui)

```bash
consul leave
```

---

## Local Consul Datacenter

### Set up the environment

```bash
vagrant up
```

```bash
vagrant ssh n1
vagrant ssh n2
```

### Start the agents

#### n1

```bash
consul agent \
  -server \
  -bootstrap-expect=1 \
  -node=agent-one \
  -bind=172.20.20.10 \
  -data-dir=/tmp/consul \
  -config-dir=/etc/consul.d
```

```bash
consul members

Node       Address            Status  Type    Build  Protocol  DC   Segment
agent-one  172.20.20.10:8301  alive   server  1.9.1  2         dc1  <all>
```

#### n2

```bash
consul agent \
  -node=agent-two \
  -bind=172.20.20.11 \
  -enable-script-checks=true \
  -data-dir=/tmp/consul \
  -config-dir=/etc/consul.d
```

```bash
consul members

Node       Address            Status  Type    Build  Protocol  DC   Segment
agent-two  172.20.20.11:8301  alive   client  1.9.1  2         dc1  <default>
```

### Join the agents

n1:

```bash
consul join 172.20.20.11
```

```bash
consul members

Node       Address            Status  Type    Build  Protocol  DC   Segment
agent-one  172.20.20.10:8301  alive   server  1.9.1  2         dc1  <all>
agent-two  172.20.20.11:8301  alive   client  1.9.1  2         dc1  <default>
```

### Query a node

n1:

```bash
dig @127.0.0.1 -p 8600 agent-two.node.consul

;; ANSWER SECTION:
agent-two.node.consul.  0       IN      A       172.20.20.11
;; ADDITIONAL SECTION:
agent-two.node.consul.  0       IN      TXT     "consul-network-segment="
```

### Stop the agents

`Ctrl-c` or `consul leave`

### Clean up the environment

```bash
vagrant destroy -f
```
