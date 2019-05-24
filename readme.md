---
title: Traefik Enterprise Edition Production Best Practices Solution Brief for Docker Enterprise and Swarm
summary: This Solution Brief documents how to deploy a Highly-Available Traefik =EE ingress solution for Docker Swarm
type: guide
author: adamancini
campaign: docker-certified-infrastructure
product:
- ee
testedon:
- EE 2.0
- EE 2.1
- ee-17.06.2-ee-17
- ee-18.09.00-ee
- ucp-3.0.6
- ucp-3.1.6
- dtr-2.6.0
- dtr-2.6.3
platform:
- linux
tags:
- solution-brief
- security
- networking
- swarm
- ingress
- load-balancing
---

# Traefik Enterprise Edition on Docker Enterprise Edition with Docker Swarm

Traefik Enterprise Edition (Traefik EE) is a production-grade, distributed, and highly-available routing solution built on top of [Traefik](https://traefik.io/).

Containous aims at simplifying the life of today’s DevOps and Site Reliability Engineers (SREs) with an easy-to-install, robust and secure edge router. Our cloud-native solution enables users to address all the routing (simple to very complex), load balancing, tracing and observability, and governance needs they may have with a small footprint. Containous is a cloud-agnostic and legacy friendly routing solution.
Traefik EE is an enterprise-grade ingress controller built upon the acclaimed open source edge router 'Traefik'. It benefits from one of the most vibrant and supportive communities and caters to all the 'wiring' needs of your microservices projects, whether you are starting from scratch (greenfield) or transitioning away from a different infrastructure.

![Traefik Architecture](./images/traefikee-1.png "Traefik Architecture")

To learn more about Traefik EE concepts, see [Concepts of the documentation](https://docs.containo.us/learning/concepts/).

For production environments, Docker recommends using deploying Traefik EE data nodes on Docker EE workers dedidicated to load balancing.  Our goal is to achieve an environment similar to our [Layer 7 Routing with Interlock](https://docs.docker.com/ee/ucp/interlock/deploy/production/) documentation.

##### todo <insert architecture diagram of our goal: 3 control node, 2 data node, nodetype==loadbalancer, host-mode publishing.

## Prerequisites

The 'Traefik Enterprise Edition Production Best Practices Solution Brief for Docker Enterprise and Swarm' solution guide has been tested using the following environment:

- Docker EE 2.1
- TraefikEE v1.0.1, with [the command-line tool `traefikeectl`](https://docs.containo.us/installation-guides/getting-started/#install-traefikeectl)
- A [valid TraefikEE license](https://containo.us/traefikee/) stored in the environment variable `$TRAEFIKEE_LICENSE_KEY`
- An external layer 4 load-balancer and associated DNS A record

This guide assumes you are familiar with the basic architecture of Traefik outlined in the [Solution Guide](https://docs.containo.us/solutions/dockeree/)

#### A Note on Terminology

This guide often refers to Traefik EE "control" and "data" nodes.  There are various ways of deploying the components of Traefik EE and the number of containers may not necessarily be equal to the number of nodes that Traefik is deployed on.  For the purposes of this guide a Traefik control node or Traefik data node refers to a single Traefik EE container deployed in a given cluster or the collective Traefik EE containers, not the underlying Docker EE infrastructure.

## Ingress Planning Requirements

### Name Resolution

To reach Traefik EE data nodes from an external network, we recommend a wildcard DNS record that resolves to an external load-balancer distributing requests to your Docker EE workers hosting Traefik EE data nodes.

If you intend to administrate Traefik EE with `traefikeectl` on your local workstation, you may also need to adjust your UCP loadbalancer to allow traffic through the `traefikeectl` API port (default: `55055`).

In this guide, we make the assumption that there is an A record `*.tr.domain.com` that resolves to a load balancer, distributing requests to Traefik EE data nodes.  We assume that there is a separate A record and load balancer for UCP, such as `ucp.domain.com`.

### Ports

Operating Traefik EE requires the use of certain cluster ports:

- 2 available ports on the Docker EE worker nodes that will be dedicated for load balancing: Let's say `9080` and `9443`
    - Configure your external load balancer to forward port `80` -> `9080` targeting your load balancer nodes, and repeat for `443` -> `9443`
- The Control API port, used by `traefikeectl` to communicate with control nodes (default: 55055)
- The dashboard port, where the dashboard is served (default: 8080) on the control nodes

The Control API port, with Traefik EE, is mTLS encrypted.  See the related documents on [Security](https://docs.containo.us/learning/concepts/#security).

### Scheduling

Traefik EE control nodes require access to Docker APIs.  Traefik EE 1.0.1, at the time this guide was written, is not able to use TLS certificates to authenticate against the UCP APIs, therefore we must bind Docker's control socket on the manager nodes.  Refer to our [Engine Security](https://docs.docker.com/engine/security/https/) documentation for more information on protecting the Docker socket.  In a future version, Traefik EE will add support for TLS authentication so that it may communicate with the Docker APIs via UCP client certificates.

## Installation

You can install Traefik EE in HA Mode using
[Advanced Installation of Traefik Enterprise Edition on Swarm with Compose Files](https://docs.containo.us/installation-guides/swarm/compose/n-cn/),

#### Minimum Requirements
- The `traefikeectl` tool installed on a manager node or on your workstation
- A Docker Swarm (swarm mode) cluster:
  - Version: >= 1.13 (minimum API version 1.25)
  - At least 3 manager nodes, and at least 2 worker nodes
- Docker client
  - Version: >= 1.13 (minimum API version 1.25)
  - Either:
    - Configured to communicate with your swarm cluster by sourcing a [UCP Client Bundle]()
    - Available locally on one of the manager nodes
- Ingress ports are available and reachable
- `traefikeectl` API port is reachable (default: `55055`)

#### Install Prerequisites
```
# obtain `traefikeectl`
mkdir traefikee && cd traefikee
wget https://s3.amazonaws.com/traefikee/binaries/v1.0.1/traefikeectl/traefikeectl_v1.0.1_linux_amd64.tar.gz
mv traefikeectl /usr/bin/.

# obtain the compose files
curl -sSL \
  https://s3.amazonaws.com/traefikee/examples/v1.0.1/swarm/traefikee-swarm-v1.0.1.tar.gz | tar xvz

# create the Traefik EE License Secret
read -sp 'Traefik EE License Key: ' TRAEFIKEE_LICENSE_KEY
echo -n ${TRAEFIKEE_LICENSE_KEY} | docker secret create <^>traefikee-license<^^> -
```

> Note: If you are installing TraefikEE from behind a corporate proxy, you will need to add the following environment
variables to each deployment YAML so that TraefikEE can make its upstream license check:
```
services:
  control-node: # or bootstrap-node or data-node.
    # [...]
    environment:
      HTTP_PROXY: "http://127.0.0.1:3129"
      HTTPS_PROXY: "http://127.0.0.1:3129"
```

#### Bootstrap the Traefik EE Control plane

##### Create the Traefik EE control plane network

```
docker network create --driver=overlay <^>traefikee-control<^^>
```

##### Create the `bootstrap-node` service

Edit bootstrap-node.yml, and set the name of the control plane network
```
networks:
  traefikee-net:
    name: <^>traefikee-control<^^>
    external: true
```

Set the name of our secret containing our license key
```
secrets:
  traefikee-license:
    external: true
    name: <^>traefikee-license<^^>
```

Under the `command:` key, replace the rest of the variable prompts with appropriate values for your environment and change `--timeout=120` to `--timeout=600`
```
    command:
      - "bootstrap"
      - "--swarmmode"
      - "--swarmmode.network=<^>traefik-control<^^>
      - "--swarmmode.licensesecret=traefikee-license"
      - "--traefikeelog.traefik=<^>debug<^^>
      - "--clustername=<^>traefikee-swarm<^^>"
      - "--api"
      - "--timeout=600"
      - "--controlNodes=<^>3<^^>"
```

Alternatively, wherever there is a `${VARIABLE}` in a compose file, you may export those variables in your shell and "docker stack deploy" will use those values instead
```
export TRAEFIKEE_LICENSE_SECRET=<^>traefikee-license<^^>
export TRAEFIKEE_SWARM_NETWORK=<^>traefikee-control<^^>
export TRAEFIKEE_EXPECTED_CONTROL_NODES=<^>3<^^>
export TRAEFIKEE_LOG_LEVEL=<^>debug<^^>
export TRAEFIKEE_CLUSTER_NAME=<^>traefikee-swarm<^^>
```

Save the file.  Completed, it should look similar to the following:
```
---
version: '3.6'
networks:
  traefikee-net:
    name: traefikee-control
    external: true
secrets:
  traefikee-license:
    name: traefikee-license
    external: true
services:
  bootstrap-node:
    image: containous/traefikee:v1.0.1
    deploy:
      restart_policy:
        condition: on-failure
      placement:
        constraints:
          - node.role == manager
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
    secrets:
      - traefikee-license
    networks:
      - traefikee-net
    command:
      - "bootstrap"
      - "--swarmmode"
      - "--swarmmode.network=traefikee-control"
      - "--swarmmode.licensesecret=traefikee-license"
      - "--traefikeelog.traefik=debug"
      - "--clustername=traefikee-swarm"
      - "--api"
      - "--timeout=600"
      - "--controlNodes=3"
    labels:
      - "traefikee=bootstrap-node"
```

Finally, deploy the `bootstrap-node` service and verify that it is scheduled
```
docker stack deploy -c bootstrap-node.yml <^^>traefikee-swarm<^>
docker service ps <^>traefikee-swarm<^^>_bootstrap-node
docker service logs -f <^>traefikee-swarm<^^>_bootstrap_node
```

##### Create the `control-node` service

Edit `control-node.yaml` and set compose API version to '`3.7`'
```
version: '3.7'
```

Next, configure the name of the Traefik EE control plane network
```
networks:
  traefikee-net:
    name: <^>traefikee-control<^^>
    external: true
```

Next, set the name of the Traefike EE control node join token
```
# the name of the secret will be cluster-name-control-node-join token
# you can use `docker secret ls` to discover it

docker secret ls
> ID                          NAME                                        DRIVER              CREATED              UPDATED
> iwh6gkt7vdksecv9dg1d1ayy8   traefikee-swarm-control-node-join-token                       About a minute ago   About a minute ago
> ktpk6p88nf31ldo8byrtnobg5   traefikee-swarm-data-node-join-token                          About a minute ago   About a minute ago
> ld6mvz4lh1ox0u7iofqjiwdl1   traefikee-license                                               7 minutes ago        7 minutes ago
> i550mhwctoryyyeujttx0bwr9   ucp-auth-key                                                    4 days ago           4 days ago

# in our case <^>traefikee-swarm<^^>-control-node-join-token
# in control-node.yaml:

secrets:
  traefikee-control-node-join-token:
    external: true
    name: <^>traefikee-swarm<^^>-control-node-join-token
```

Continue editing `control-node.yaml` and add `update_config`, `rollback_config`, and `restart_policy` parameters

> Note:  these timers should be generous to ensure that when the control plane is updated with "docker service update", we do not shut down more than 1 control node at a time in order to maintain a quorum.  The given settings provide about ~90s of delay in between task restarts to allow for the control plane to converge.  These timers may be adjusted for your environment.


```yaml
services:
  control-node:
    image: containous/traefikee:v1.0.1
    deploy:
      mode: replicated
      replicas: ${TRAEFIKEE_CONTROL_NODE_REPLICAS_COUNT}
      restart_policy:
        condition: any
        delay: 5s
        max_attempts: 3
        window: 60s
      update_config:
        parallelism: 1
        delay: 60s
        failure_action: rollback
        max_failure_ratio: .25
        order: start-first
      rollback_config:
        parallelism: 1
        failure_action: pause
        order: start-first
      placement:
        constraints:
          - node.role == manager
        ...
```

Then, configure the Traefik EE dashboard and `traefikeectl` API ports, if desired.  These will default to `8080` and `55055`, respectively.
```yaml
    ports:
      - ${TRAEFIKEE_DASHBOARD_PORT:-8080}:8080
      - ${TRAEFIKEE_CTLAPI_PORT:-55055}:55055
```

Then, configure the `command:` key and replace the `${TRAEFIKEE_LOG_LEVEL}` and `${TRAEFIKEE_SWARM_NETWORK}` variables
```yaml
    command:
      ...
      - "--traefikeelog.traefik=<^>info<^^>"
      - "--swarmmode"
      - "--swarmmode.network=<^>traefikee-control<^^>"
      - "--swarmmode.jointokensecret=traefikee-control-node-join-token"
```

Finally, replace the `${TRAEFIKEE_PEER_ADDRESSES}` variable with the address of our `bootstrap-node` service:
```yaml
    command:
      - "--peeraddresses=<^>traefikee-swarm<^^>_bootstrap-node:4242"
```

And save the file.  The complete `control-node.yaml` file should look similar to the following:
```yaml
---
version: '3.7'

networks:
  traefikee-net:
    name: traefikee-control
    external: true

secrets:
  traefikee-control-node-join-token:
    external: true
    name: traefikee-swarm-control-node-join-token

services:
  control-node:
    image: containous/traefikee:v1.0.1
    deploy:
      mode: replicated
      replicas: 3
      restart_policy:
        condition: on-failure
        delay: 5s
        max_attempts: 3
        window: 30s
      update_config:
        parallelism: 1
        delay: 180s
        failure_action: rollback
        max_failure_ratio: .25
        order: start-first
      rollback_config:
        parallelism: 0
        order: start-first
      placement:
        constraints:
          - node.role == manager
    stop_grace_period: 60s
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
    secrets:
      - traefikee-control-node-join-token
    networks:
      - traefikee-net
    ports:
      - ${TRAEFIKEE_DASHBOARD_PORT:-8080}:8080
      - ${TRAEFIKEE_CTLAPI_PORT:-55055}:55055
    command:
      - "start-control-node"
      - "--peeraddresses=traefikee-swarm_bootstrap-node:4242"
      - "--traefikeelog.traefik=info"
      - "--swarmmode"
      - "--swarmmode.network=traefikee-control"
      - "--swarmmode.jointokensecret=traefikee-control-node-join-token"
    labels:
      - "traefikee=control-node"
```

###### If you wish to configure Traefik to proxy the Traefik EE dashboard

Generate a username & password combination to authenticate against the dashboard, using the `htpasswd` utility (or similar) for HTTP Simple Auth.
We need to tell the YAML parser to escape each '`$`' in our resulting password string with a `$` character (replacing `$` with `$$`) to use it directly in docker-compose.yml file:

```sh
read -sp 'Traefik EE Dashboard Password: ' TRAEFIKEE_DASHBOARD_PASSWORD
echo $(htpasswd -nbB <^>user<^^> "${TRAEFIKEE_DASHBOARD_PASSWORD}") | sed -e s/\\$/\\$\\$/g

> <^>user<^^>:$$apr1$$ryHGa8yK$$5lRELezhgkUtJxiJ.XTfZ.
```

- This is not necessary if the value is exported to your shell:
    ```sh
    read -sp 'Traefik EE Dashboard Password: ' TRAEFIKEE_DASHBOARD_PASSWORD
    export TRAEFIKEE_DASHBOARD_PASSWORD=$(htpasswd -nbB <^>user<^^> "${TRAEFIKEE_DASHBOARD_PASSWORD}")
    ```

> More advanced authentication methods can be configured using Traefik's `frontend.auth` labels.  See[authentication](https://docs.traefik.io/configuration/api/#authentication) for more details.

Add the following deployment labels to your control-node.yml file.
> Note the documentation on the correct placement of the [deploy](https://docs.docker.com/compose/compose-file/#deploy) key

```yaml
    deploy:
      mode: replicated
      replicas: <^>3<^^>
      labels:
        - "traefik.docker.network=<^^>traefikee-control<^>"
        - "traefik.enable=true"
        - "traefik.basic.frontend.rule=Host:<^>dashboard.tr.domain.com<^^>"
        - "traefik.frontend.auth.basic=<^>user:$$apr1$$ryHGa8yK$$5lRELezhgkUtJxiJ.XTfZ.<^^>"
        - "traefik.basic.port=8080"
        - "traefik.basic.protocol=http"
```

Deploy the `control-node` service and validate
```sh
docker stack deploy -c control-node.yml <^>traefikee-swarm<^^>
docker service ps <^>traefikee-swarm<^^>_control-node
docker service logs -f <^>traefikee-swarm<^^>_control-node
```

#### Create the `data-node` service

```
```

#### Configure Traefik EE entrypoint

When the installation is complete, check your cluster nodes and logs using `traefikeectl`:

```sh
traefikeectl list-nodes --clustername=traefikee-swarm
traefikeectl logs --clustername=traefikee-swarm
...
```

- Deploy a customized [routing configuration](https://docs.containo.us/references/configs/routing/#configure-routing-in-traefikee)
  to create the [entrypoints](https://docs.traefik.io/configuration/entrypoints/).
  Please note that Traefik EE uses the `80` and `443` port internally,
  hence these values for the entrypoints:

    ```shell
    traefikeectl deploy --clustername=traefikee-swarm \
        --docker.swarmmode \
        --entryPoints='Name:http Address::80' \
        --entryPoints='Name:https Address::443 TLS' \
        --defaultentrypoints=https,http
    ```

#### Deploy Test Application

You can start deploying applications in Docker Swarm with
[labels configured](https://docs.traefik.io/configuration/backends/docker/#using-docker-with-swarm-mode):

- Start by creating the following Docker YAML Compose file named `whoami.yaml`, given an DNS record of `*.tr.domain.com` for our Traefik data nodes

```yaml
version: '3.7'
networks:
  traefikee-ingress:
    external: true

services:
  whoami:
    image: containous/whoami
    deploy:
      mode: replicated
      replicas: 2
      labels:
        - "traefik.backend=whoami"
        - "traefik.enable=true"
        - "traefik.frontend.rule=Host:whoami.tr.domain.com"
        - "traefik.port=80"
    networks:
     - traefikee-ingress
  whoami2:
    image: containous/whoami
    deploy:
      mode: replicated
      replicas: 2
      labels:
        - "traefik.backend=whoami"
        - "traefik.enable=true"
        - "traefik.frontend.rule=Host:whoami.tr.domain.com"
        - "traefik.port=80"
        - "traefik.weight=10"
    networks:
     - traefikee-ingress
```

- Deploy your application with the following command:

    ```shell
    docker stack deploy --compose-file=./whoami-stack.yaml whoami
    ```

- Check the application deployment status, with `2/2` replicas ready:

    ```shell
    docker stack ps whoami
    ```

- Verify that the requests are routed by TraefikEE to the "whoami" application:

    ```shell
    curl http://public.cluster.dns.org:9080
    ```

- Cleanup the "whoami" application if everything is alright:

    ```shell
    docker stack rm whoami
    ```


#### Move an application from Interlock to Traefik EE
