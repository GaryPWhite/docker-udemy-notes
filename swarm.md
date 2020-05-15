### Docker Swarm Init

PKI and security automation for root signing, and certificate/tokens are issued

### Raft DB

made to make root CA, configs, and secrets. Encrypts on disk, is running as a database on disk. Swarm includes the database / management of secrets and key value store. Usually you'd need a redundant db system alongside the orchestrator, but it's handled here.

Works a lot like CAPI storing credentials, but ahead of curve in automation of it's internal code certification.

there's only one leader at a time, `docker node ls` shows the leader, since it's the only one.
You can promote/demote nodes in swarm

`docker swarm` is like `cf target` or `cf init`, allows us to get back and forth from swarms or start them.

### Docker Service

`docker service` is a lot like `docker run`. Docker run is for local systems, docker service is more for swarm and redundant systems. Makes it easier to make docker containers like cattle, instead of like services.

docker service allows you to start a _service_, which runs commands in containers according to necessary requirements like cpu/how many replicas. when a container goes down (through `docker container rm` or otherwise) the orchestrator from RAFT/swarm will notice and bring up another replica. You can see this happen by spinning up something like alpine, setting replicas or not, then `docker container rm`-ing the id or name found from `docker service ls`. Shows something like:

```shell
ID                  NAME                     IMAGE               NODE                DESIRED STATE       CURRENT STATE            ERROR                         PORTS
1016d1mgruoq        admiring_greider.1       alpine:latest       docker-desktop      Running             Running 5 minutes ago
hyfrxv4rb00r        admiring_greider.2       alpine:latest       docker-desktop      Running             Running 20 seconds ago
6atizj93h8ri         \_ admiring_greider.2   alpine:latest       docker-desktop      Shutdown            Failed 25 seconds ago    "task: non-zero exit (137)"
```

cool stuff. By bringing up and down containers in the service, we exemplify what swarm is supposed to do, orchestrate. There's `docker container update` and some other command to add CPU/RAM and modify more DNS/secret level entries that we didn't get into.

ooh, tidbit! These service create / update commands are synchronous. They'll wait and stop unless you use `--detatch`, which might or might not be what you'd want in CI!

### Setting up a swarm cluster

play-with-docker.com <-- Will be using this to set up swarm. resets after about 4 hours.
might use Digital Ocean, maybe. They're great to use, I should definitely set it up :/

docker-machine is not something that was designed for production. Worth noting. get.docker.com is the best way to install docker.

using swarm to create the host with `docker swarm init --advertise-addr <IP>` was simple enough. By default, the nodes are not as priveleged as the leader. You can promote them with a manager token. It also allows us to get more nodes to join easily with a provided command like `docker swarm join --token <token> <IP-ADDR>

you can also get keys with `docker swarm join-token manager` frmo a manager node. THen you can rotate keys as needed.

### Overlay Mutli-Host Networking
container-to-container traffic can be done with a driver called "Overlay". Optionally, AES encryption is available through IPSec connections.
Each service can be connected through multiple overlays or local networks, just like docker-compose.

can get logs from docker service logs :O

By using drupal (which is basically just wordpress) we can see in an overlay network that postgres runs, and drupal can connect to it to init a site.

Big question is how do we get a DNS entry for a docker swarm?
It looks like the site runs on all three nodes because we can visit the site from every node. With `docker service inspect` we'll see that there's only one IP on the internal network.
There's a routing mesh automatically created with swarm.

### Routing mesh
Routes ingress traffic for a Service to the right Task
Works throughout Swarm with IPVS.
It's automatic load balancing for swarm services across tasks (a lot like go-router tbh) it doesn't use round-robin; it uses virtual IP's. VIP allows us to rely on our hardware, instead of risking caching problems on our clients and not distributing load correctly.

Idea here is that we don't have to care which node is running a workload. Routing Mesh will route to the right Task (container) that's available.
the load balancer gets hit on any of the nodes and distrubutes appropriately

in 17.03 -- the Routing Mesh and Load balancer is stateless (could still be now)
If you need session cookies/other stuff that's stateful, load balancing gives you issues.
VIP routing and LB is OSI layer, not Layer 4 (DNS)
sticky routing is doable with just nginx or some other equivalent. Honestly fuck that just do stateless wtf

### Notes from multi-service multi-node app
obvious network overlays
`docker network create -d overlay frontend|backend`, and services are created linked to those networks.
create commands are self-explanatory for multiple networks (two network specifications)

### Docker Stack = production grade compose
Stack allows us to manage services, networks, etc. at the same abstraction as docker compose. `build:` is not available with stack deployments. Swarm uses `deploy` instead of using `build`. Compose allows build, swarm allows deploy.

docker-compose cli is not needed on a swarm server.

Swarm allows us to link many `services` together with `volumes` and `overlay networks` across different nodes.

`deploy` as a swarm keyword allows us to specify swarm-specific config like:
  replicas
  update_config (parallelism by number and delay in time)
  restart_policy (max_attempts, delay)
  placement (constraints in metadata on nodes like node.role == manager)
`docker stack deploy -c docker-compose <name>`
deploys services, makes volumes, replicas, networks, etc. Can use `docker stack ps <name>` to see services running


### Secrets storage
Easiest """secure""" solution for storing secrets in Swarm

Supports generic strings or binary content up to 500mb in size
RaftDB is encrypt on disk, and secrets are only on manager nodes.

Managers and Workers only use TLS. secrets must be assigned directly to services.

### Docker compose secrets (for local dev)
Docker-compose can bind-mount the files at runtime. So if we have a file in the local filesystem, you can mount it into the container at runtime.

Compose file would have something like
```
secrets:
  secret_username:
    file: ./my-username
....
```

### Docker-compose locally vs. Swarm
something named specifically `docker-compose.override.yml` will allow us to override shared configuration between local environment and a swarm deployment.

Remember that we use docker-compose for both local and swarm, so if, we needed `volumes`, `secrets`, etc. configured differently for local dev; we can do so with docker-compose.

You can also use `-f` to override specific configurations like `.prod` or `.test`. 

`docker-compose config` allows us to input multiple `-f`iles and output a full one. SO putting `docker-compose -f first-config -f config-two config > docker-compose.combined.yml`

### Service Updates
Should Limit downtime with a rolling replacement of tasks/containers in a service

Update has many many cli options to control the update. Create options are able to -add or -rm things. ALSO has rollback/healthcheck options built into swarm

For any pre-existing deployments, swarm will auto-update on options instead of replacing.
EX: updating an image to a newer version with `docker service update --image`
update env vars with `--env-add`, remove stuff with `--publish-rm PORT`
popular one is `scale service #`

### HEALTHCHECK
HEALTHCHECK was added mostly for swarm, but is kept in dockerfile, compose, run, etc.

Docker engine will `exec` the command in the container. It expects an exit 0 or specifically exit 1

three container states: starting, healthy and unhealthy

up to HEALTHCHECK it just used "is this application running?" -- also doesn't provide monitoring / graphs / performance
with swarm it shows up with `docker container ls / inspect`

note that `docker run` does nothing with healthchecks. While they exist, they don't run anything after something goes wrong.
HEALTHCHECK will coinside with updates to make sure that everything is healthy before proceeding to more services or sucessfully declaring an update finished.
easy way to get docker to work on non-one error codes is using `|| false` at the end of a maybe-failing command.

interval, timeout, start-period, retries all exist for healthcheck command.

you can also use `healthcheck` in compose/stack files directly, which is cool af


### Brett Fischer talk with Swarm and Docker :O

Worked with Sysadmin + Dev since 1994 legit when I was born ugh
solutions you don't need day one in containers:
  Fully automatic CI/CD
  Dynamic performance scaling
  Containerizing everying or containerizing nothing
  Starting with persistent data (persistent data is the harder thing to deal with) -- db's don't upgrade as frequently and they are tricky in containers
  12 factor is a horizon we're always chasing

What to Focus on First: Dockerfiles
  Make it start
  Make it log to stdout/err
  document in file
  make it work for others
  trim it down
  make it scalable

Don't use latest omfg
set ENV / config together, preferably at the top.
apt-get packages should also have versions, not for evertything, but for specfics like libraries or utils are important.

Don't leave default config or randomly copy VM configuration into a file
dont copy specific configuration .json or .ini files into your images, use specific ENV's or entrypoints per environment

docker is very kernel and storage driver dependent. Containers are a complicated part of the infrastructure. Use a 4.x kernel PLEASE if you can

baby-swarm (so cute), it's okay to do! Just better features than docker run
HA swarm is a 3 node swarm. managers should have at least odd numbers.
biz swarm should be at least 5 nodes :O
past that you're making it up

Don't turn cattle into pets. Nodes can't be favorites, containers will die, linuxkit and infrakit EXPECT THIS

Bad reasons for multiple swarms
  Different hardware configs
  different subnets or security groups
  different HA
  Security boundaries for compliance

Good Reasons
  Learning: run stuff on test swarm
  geolgraphical bounaries
  management boundaries using docker API (or EE RBAC or other auth plugins)

It's hard to have a windows only swarm, mixing with linux nodes is a good idea.
manage on linux, reserve windows for windows only workloads

Outsource well-defined plumbing
beware "not built here" syndrome
  Image registries
  logs
  monitoring and alerting

Full stack example as full-stack.png
