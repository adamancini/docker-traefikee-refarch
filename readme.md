### install prerequisites
> https://docs.containo.us/installation-guides/getting-started/#install-traefikeectl
```bash
mkdir traefikee && cd traefikee
wget https://s3.amazonaws.com/traefikee/binaries/v1.0.1/traefikeectl/traefikeectl_v1.0.1_linux_amd64.tar.gz
mv traefikeectl /usr/bin/.
```

### download compose files
```bash
curl -sSL \
  https://s3.amazonaws.com/traefikee/examples/v1.0.1/swarm/traefikee-swarm-v1.0.1.tar.gz | tar xvz
```

### create the traefikee control plane network
```bash
docker network create --driver=overlay traefikee-control
```

### create the traefikee license secret
```bash
export TRAEFIKEE_LICENSE_KEY=<traefikee-license>
echo -n ${TRAEFIKEE_LICENSE_KEY} | docker secret create traefikee-license -
```

### create bootstrap node
```bash
export TRAEFIKEE_LICENSE_SECRET=traefikee-license
export TRAEFIKEE_SWARM_NETWORK=traefikee-control
export TRAEFIKEE_EXPECTED_CONTROL_NODES=3
export TRAEFIKEE_LOG_LEVEL=debug
export TRAEFIKEE_CLUSTER_NAME=traefikee-ingress

# edit bootstrap-node.yml in vim, replace --timeout=120 with --timeout=600
docker stack deploy -c bootstrap-node.yml traefikee-ingress
```

### create control nodes
```bash
# edit control-node.yaml and set compose API version to '3.7'
version: '3.7'

# edit the control-node.yaml file and add update_config, rollback_config, restart_policy
# adjust timers to suit environment & rollout time
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

# and add a username & password combination to authenticate against the dashboard using `htpasswd` for HTTP Simple Auth
# We need to escape each $ in our resulting password string with $ (replacing $ with $$ )
# if you use it directly in docker-compose.yml .
echo $(htpasswd -nbB <USER> "<PASS>") | sed -e s/\\$/\\$\\$/g
> <USER>:$$apr1$$ryHGa8yK$$5lRELezhgkUtJxiJ.XTfZ.

    deploy:
      mode: replicated
      replicas: ${TRAEFIKEE_CONTROL_NODE_REPLICAS_COUNT}
      labels:
        - "traefik.docker.network=traefikee-control"
        - "traefik.enable=true"
        - "traefik.basic.frontend.rule=Host:dashboard.app.traefik.aws.annarchy.net"
        - "traefik.frontend.auth.basic=${TRAEFIKEE_DASHBOARD_PASSWORD}"
        - "traefik.basic.port=8080"
        - "traefik.basic.protocol=http"


# get name of control node join token
docker secret ls
> ID                          NAME                                        DRIVER              CREATED              UPDATED
> iwh6gkt7vdksecv9dg1d1ayy8   traefikee-ingress-control-node-join-token                       About a minute ago   About a minute ago
> ktpk6p88nf31ldo8byrtnobg5   traefikee-ingress-data-node-join-token                          About a minute ago   About a minute ago
> ld6mvz4lh1ox0u7iofqjiwdl1   traefikee-license                                               7 minutes ago        7 minutes ago
> i550mhwctoryyyeujttx0bwr9   ucp-auth-key                                                    4 days ago           4 days ago

# in our case traefikee-ingress-control-node-join-token
export TRAEFIKEE_CONTROL_NODE_JOIN_TOKEN=traefikee-ingress-control-node-join-token
export TRAEFIKEE_PEER_ADDRESSES="traefikee-ingress_bootstrap-node:4242"
export TRAEFIKEE_CONTROL_NODE_REPLICAS_COUNT=3
export TRAEFIKEE_LOG_LEVEL=debug
export TRAEFIKEE_DASHBOARD_PORT=18080
export TRAEFIKEE_CTLAPI_PORT=55055

# deploy the control nodes
docker stack deploy -c control-node.yml traefikee-ingress

# now remove the bootstrap node
docker service rm traefikee-ingress_bootstrap-node

# and hook up traefikeectl to the control nodes
traefikeectl connect --swarm --clustername=traefikee-ingress
> Connecting to Docker API...ok
> Connecting to TraefikEE Control API...ok
> Connecting to Docker Swarm API...ok
> Retrieving TraefikEE Control credentials...ok
> Removing cluster credentials from platform...ok
>   > Credentials saved in "/home/ada/.config/traefikee/traefikee-ingress", please make sure to keep them safe as they can never be retrieved again.
> ✔ Successfuly gained access to the cluster. You can now use other traefikeectl commands.
```

### create data nodes

```bash
# edit data-node-global.yaml and set compose API version to '3.7'
version: '3.7'

# edit data-node-global.yml and add scheduling constraint
# for nodetype==loadbalancer
services:
  data-node:
    image: containous/traefikee:v1.0.1
    deploy:
      mode: global
      placement:
        constraints:
          - node.labels.nodetype == loadbalancer
          - node.role != manager

# then, modify the ports: section and use the long-form syntax to use host mode port publishing
    ports:
      - target: 80
        published: ${TRAEFIKEE_HTTP_PORT}
        protocol: tcp
        mode: host
      - target: 443
        published: ${TRAEFIKEE_HTTPS_PORT}
        protocol: tcp
        mode: host

# next, add the update_config, rollback_config, and restart_policy
# again, adjust timers to suit the rollout time in the environment
services:
  data-node:
    image: containous/traefikee:v1.0.1
    deploy:
      mode: global
      restart_policy:
        condition: on-failure
        delay: 5s
        max_attempts: 5
        window: 30s
      rollback_config:
        parallelism: 1
        delay: 30s
        failure_action: continue
        order: stop-first
      update_config:
        parallelism: 1
        delay: 30s
        failure_action: rollback
        order: stop-first
      placement:
        constraints:
          - node.labels.nodetype == loadbalancer
          - node.role != manager

# now that out bootstrap is completed, set peeraddresses parameter to our control-node service VIP
export TRAEFIKEE_PEER_ADDRESSES=traefikee-ingress_control-node:4242
docker stack deploy -c data-node-global.yml traefikee-ingress
```

### validate deployment and configure initial entrypoint
```bash
traefikeectl list-nodes --clustername=traefikee-ingress
> Name          Availability  Role          Leader
> ----          ------------  ----          ------
> 372e3ae6062f  ACTIVE        CONTROL NODE
> 7494a4e862a1  ACTIVE        DATA NODE
> 59d4c9b90428  ACTIVE        CONTROL NODE  YES
> 1e963e846a8f  ACTIVE        DATA NODE
> 5d85db5d2158  ACTIVE        CONTROL NODE

# traefikeectl deploy --docker.swarmmode --clustername=traefikee-ingress
traefikeectl deploy --clustername=traefikee-ingress \
    --docker.swarmmode \
    --entryPoints='Name:http Address::80' \
    --entryPoints='Name:https Address::443 TLS' \
    --defaultentrypoints=https,http
```

### back up initial configuration
```bash
traefikeectl backup --clustername=traefikee-ingress
> Connecting to TraefikEE Control API...ok
> Running cluster backup...ok
> ✔ Successfully generated backup archive 2019-05-20T134238_traefikee-backup.tar
```

#### quick tear down and redeploy
```bash
traefikeectl uninstall --clustername=traefikee-ingress

export TRAEFIKEE_LICENSE_SECRET=traefikee-license
export TRAEFIKEE_SWARM_NETWORK=traefikee-control
export TRAEFIKEE_EXPECTED_CONTROL_NODES=3
export TRAEFIKEE_LOG_LEVEL=debug
export TRAEFIKEE_CLUSTER_NAME=traefikee-ingress
docker stack deploy -c bootstrap-node.yml traefikee-ingress

export TRAEFIKEE_CONTROL_NODE_JOIN_TOKEN=traefikee-ingress-control-node-join-token
export TRAEFIKEE_PEER_ADDRESSES="traefikee-ingress_bootstrap-node:4242"
export TRAEFIKEE_CONTROL_NODE_REPLICAS_COUNT=3
export TRAEFIKEE_LOG_LEVEL=debug
export TRAEFIKEE_DASHBOARD_PORT=18080
export TRAEFIKEE_CTLAPI_PORT=55055
docker stack deploy -c control-node.yml traefikee-ingress

docker service rm traefikee-ingress_bootstrap-node
traefikeectl connect --swarm --clustername=traefikee-ingress

export TRAEFIKEE_PEER_ADDRESSES=traefikee-ingress_control-node:4242
docker stack deploy -c data-node-global.yml traefikee-ingress
docker stack deploy -c control-node.yml traefikee-ingress

traefikeectl list-nodes --clustername=traefikee-ingress
traefikeectl deploy --clustername=traefikee-ingress \
    --docker.swarmmode \
    --entryPoints='Name:http Address::80' \
    --entryPoints='Name:https Address::443 TLS' \
    --defaultentrypoints=https,http
```
