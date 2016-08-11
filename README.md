Swarm Mode Dojo
===============

This is the material for a short Docker Swarm Mode Dojo.

See accompanying slides in [swarmmode-dojo.key](file://./swarmmode-dojo.key)

**Credits**: Much of this dojo is paraphrasing of the official Docker [Swarm mode tutorial](https://docs.docker.com/engine/swarm/swarm-tutorial/)

Setup
-----

You'll need:
  - Docker 1.12.0+
  - Docker Machine 0.8.0+

The easiest way to obtain these is to download the **Docker Toolbox** v1.12.0 which bundles the correct versions of these tools together.

### Download Docker Toolbox (direct links):

* Mac: [DockerToolbox-1.12.0.pkg](https://github.com/docker/toolbox/releases/download/v1.12.0/DockerToolbox-1.12.0.pkg), [Installation instructions](https://docs.docker.com/toolbox/toolbox_install_mac/)
* Windows: [DockerToolbox-1.12.0.exe](https://github.com/docker/toolbox/releases/download/v1.12.0/DockerToolbox-1.12.0.exe), [Installation instructions](https://docs.docker.com/toolbox/toolbox_install_windows/)

### Create the machines for the swarm mode cluster

```sh
docker-machine create --driver virtualbox manager1
docker-machine create --driver virtualbox worker1
docker-machine create --driver virtualbox worker2
```

**Note**: This may download the `boot2docker.iso` (~38MB) if you do not have it cached locally already.

Verify machines are running:

```sh
docker-machine ls
```

Creating a swarm (with Docker Engine swarm mode)
------------------------------------------------

Record the IP of the `manager1` machine:

```sh
MANAGER_IP=$(docker-machine ip manager1)
```

Every swarm needs at least one manager:

```sh
docker-machine ssh manager1 docker swarm init --advertise-addr ${MANAGER_IP}
```

```
# Result ==>

Swarm initialized: current node (ejky1mjblvj05r22ksrjfbywz) is now a manager.

To add a worker to this swarm, run the following command:
    docker swarm join \
    --token SWMTKN-1-25qrry7uy8uhdsx7i0q945iw8c5rbuhz6xeqnaafohzg8w0vgl-e964ew7q6ab19b31c3u7s3f0o \
    192.168.99.104:2377

To add a manager to this swarm, run the following command:
    docker swarm join \
    --token SWMTKN-1-25qrry7uy8uhdsx7i0q945iw8c5rbuhz6xeqnaafohzg8w0vgl-elx0yjzl4e9krxl1biqz4jckr \
    192.168.99.104:2377
```

*Note:* The output includes the commands to add nodes to the swarm. Nodes will join as managers or workers depending on the value for the `--token` flag.

---

**Production tip:**

Docker recommends three or five manager nodes per cluster to implement high availability.  There must be an odd number of managers so that the swarm can continue to function after as long as a quorum of more than half of the manager nodes are available (i.e. 3 nodes => supports 1 node failure, 5 nodes => supports 2 node failures)

---

**Security tip:**

It is recommended to regularly rotate your manager and worker tokens (at least every few months).  This won't affect existing nodes in the swarm.

Example of rotating the join tokens:

```sh
docker swarm join-token --rotate worker
docker swarm join-token --rotate manager
```

If you forget the join tokens or have recently rotated them, you can get the current tokens like so:

```sh
docker swarm join-token manager
docker swarm join-token worker
```
---

We will be working with the manager to interact with the swarm.

You can either issue commands by prefixing them with `docker-machine ssh`:

```sh
docker-machine ssh manager1 docker info
```

...but it's probably easier to just set the Docker environment variables for Docker client:

```sh
eval $(docker-machine env manager1)
docker info
```

We'll use the later approach for rest of this dojo.

View the current state of the swarm:

```sh
docker node ls
```

```
# Result ==>

ID                           HOSTNAME  STATUS  AVAILABILITY  MANAGER STATUS
ejky1mjblvj05r22ksrjfbywz *  manager1  Ready   Active        Leader
```

The `*` after the node id indicates that you’re currently pointed to this node (via env variables).

### Add nodes to the swarm

```sh
WORKER_TOKEN=$(docker swarm join-token --quiet worker)

# add worker1
docker-machine ssh worker1 \
    docker swarm join --token ${WORKER_TOKEN} ${MANAGER_IP}:2377

# add worker2
docker-machine ssh worker2 \
    docker swarm join --token ${WORKER_TOKEN} ${MANAGER_IP}:2377
```

Verify nodes are in the swarm cluster:

```sh
docker node ls
```

```
# Result ==>

ID                           HOSTNAME  STATUS  AVAILABILITY  MANAGER STATUS
7x1weqj7zx7kjf6eidxhfdqji    worker2   Ready   Active
abgl5l3ery4spnduwat3k67ag    worker1   Ready   Active
bdtx1i0umjk1otktlm16ycdix *  manager1  Ready   Active        Leader
```

You can add more managers to the swarm and they will cluster and elect a leader automatically.  No external KV store (e.g. Consul or etcd) is required for swarm mode.

When a node joins the swarm the following happen [1]:

* Docker Engine for node switches to swarm mode.
* TLS certificate is requested from the manager.
* Swarm node is named with the machine hostname.
* Current node joins the swarm via the manager listen address (2377) using correct the swarm join-token.
* Current node is set to Active availability, meaning it can receive tasks from the scheduler.
* The `ingress` overlay network is extended to include the current node.

**Homework**:

* Explore node options: `docker node --help`, in particular `docker node update --help`

Deploy a service to the swarm
-----------------------------

Create a service with a single task (instance) running:

```sh
docker service create --replicas 1 --name helloworld alpine ping docker.com
```

Verify it is running:

```sh
docker service ls
```

```
# Result ==>

ID            NAME        REPLICAS  IMAGE   COMMAND
7wtte1nf5tiy  helloworld  1/1       alpine  ping docker.com
```

The number of replicas will initially be `0/1` then become `1/1`  after the Docker image is pulled down and the task container is started.

We can also see the individual tasks that are running for the service:

```sh
docker service ps helloworld
```

```
# Result ==>

ID                         NAME          IMAGE   NODE      DESIRED STATE  CURRENT STATE          ERROR
7syo7zb9famyeqhzqk9pfwtcx  helloworld.1  alpine  manager1  Running        Running 5 minutes ago
```

We can see which node the individual task(s) are running on.  In this case, it is `manager1`.

*Note:* By default, manager nodes in a swarm can execute tasks just like worker nodes.  Set its availability to `DRAIN` (via `docker node update --availability drain <node_name>`) to avoid scheduling onto manager nodes.

How does a task map to a container?  Each task is a schedule slot and can hold a single container.  You can see the container's details (assuming it is running on `manager1`, otherwise run `docker ps` on the corresponding node):

```sh
# Env points at manager1
docker ps
```

```
# Result ==>

CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS               NAMES
610194d2300e        alpine:latest       "ping docker.com"   5 minutes ago       Up 5 minutes                            helloworld.1.7syo7zb9famyeqhzqk9pfwtcx
```

We can see that the TASK NAME (`helloworld.1`) + TASK ID (`7syo7zb9famyeqhzqk9pfwtcx`) = CONTAINER NAME (`helloworld.1.7syo7zb9famyeqhzqk9pfwtcx`).  Where TASK NAME is really the service name (`helloworld`) followed by a period then the instance number (`1`).

We can see the container logs:

```sh
docker logs $(docker ps -q -f 'name=helloworld.1')

# Result ==>

PING docker.com (54.165.204.109): 56 data bytes
```

Inspect the service for further details:

```sh
docker service inspect --pretty helloworld
```

```
# Result ==>

ID:		7wtte1nf5tiyvm479ink33kt2
Name:		helloworld
Mode:		Replicated
 Replicas:	1
Placement:
UpdateConfig:
 Parallelism:	1
 On failure:	pause
ContainerSpec:
 Image:		alpine
 Args:		ping docker.com
Resources:
```

### Scale the service in the swarm

Scale the number of tasks (instances) for the `helloworld` service:

```sh
docker service scale helloworld=5
```

Verify the number of replicas is now 5:

```sh
docker service ls
```

```
# Result ==>

ID            NAME        REPLICAS  IMAGE   COMMAND
7wtte1nf5tiy  helloworld  5/5       alpine  ping docker.com
```

See the state and placement of individual tasks for the service:

```sh
docker service ps helloworld
```

```
# Result ==>

ID                         NAME          IMAGE   NODE      DESIRED STATE  CURRENT STATE          ERROR
7syo7zb9famyeqhzqk9pfwtcx  helloworld.1  alpine  manager1  Running        Running 5 hours ago
5z61sq51a5czkggivscqc6cmb  helloworld.2  alpine  worker2   Running        Running 3 minutes ago
0lcas1u18y7lkk8dgrnrkbo4z  helloworld.3  alpine  worker1   Running        Running 3 minutes ago
d8mdm681dh2za33phww846ksg  helloworld.4  alpine  worker1   Running        Running 3 minutes ago
110y342etrmdc8r1q9o44e8z9  helloworld.5  alpine  manager1  Running        Running 3 minutes ago
```

### Drain a node on the swarm

The swarm manager will assign new tasks to any `ACTIVE` node.  Setting a node's availability to `DRAIN` prevents a node from receiving new tasks from the swarm manager.  Any running tasks on the node in `DRAIN` availability are stopped and relaunched on a node with `ACTIVE` availability.   This is useful during planned maintenance of a node but also kicks in when a worker node fails.

Check out the availability of your nodes:

```sh
docker node ls
```

```
# Result ==>

ID                           HOSTNAME  STATUS  AVAILABILITY  MANAGER STATUS
341se8va0x5y2r1eekc2jjmpj    worker1   Ready   Active
3vez6odbggl60qnryovwrvszv *  manager1  Ready   Active        Leader
46yv4be644u8bv53n5cud1yzt    worker2   Ready   Active
```

They are all active.

Set the `manager1` node to `DRAIN` state:

```sh
docker node update --availability drain manager1
```

Verify its state is set to `DRAIN`:

```sh
docker node ls
```

```
# Result ==>

ID                           HOSTNAME  STATUS  AVAILABILITY  MANAGER STATUS
7x1weqj7zx7kjf6eidxhfdqji    worker2   Ready   Active
abgl5l3ery4spnduwat3k67ag    worker1   Ready   Active
bdtx1i0umjk1otktlm16ycdix *  manager1  Ready   Drain         Leader
 ```

 Check that the swarm manager has moved the tasks from the drained node to another node:

 ```sh
 docker service ps redis
 ```

```
# Result ==>

ID                         NAME              IMAGE   NODE      DESIRED STATE  CURRENT STATE            ERROR
cu1rj1rsultoq1w3q9z77dad7  helloworld.1      alpine  worker2   Running        Running 56 seconds ago
0ipadf2exqp0z1axhjhtt7fe1   \_ helloworld.1  alpine  manager1  Shutdown       Shutdown 56 seconds ago
6xu2kx91dalm7bphroqj3c8ad  helloworld.2      alpine  worker2   Running        Running 14 minutes ago
d47vveuz6z9yesxuzqlbz3eja  helloworld.3      alpine  worker1   Running        Running 14 minutes ago
3gn53cag59m3y6rioo6uc17r9  helloworld.4      alpine  worker1   Running        Running 14 minutes ago
ew9h3r6u3jnuevo120bxvwuq5  helloworld.5      alpine  worker2   Running        Running 56 seconds ago
02behooqon0f9a7sq2rw7xuwr   \_ helloworld.5  alpine  manager1  Shutdown       Shutdown 56 seconds ago
```

No tasks are running on `manager1`, they have been moved to one of the workers.

To put the `worker1` node back in ACTIVE state run:

```sh
docker node update --availability active manager1
```

When you set the node back to `ACTIVE` availability, it can receive new tasks under the following circumstances [1]:

* during a service update to scale up
* during a rolling update
* when you set another node to `DRAIN` availability
* when a task fails on another active node

### Delete the service running on the swarm

Remove the `helloworld` service and all its tasks form the swarm:

```sh
docker service rm helloworld
```

Verify the service is gone:

```sh
docker service ls
```

```
# Result ==>

ID  NAME  REPLICAS  IMAGE  COMMAND
```

```sh
docker service inspect helloworld
```

```
# Result ==>

Error: no such service or task: helloworld
```

*Note:* All associated containers are cleaned up from the nodes in the swarm.

### Homework: Apply rolling updates to a service

This section shows how to roll out updates to a service in a swarm.

Deploy Redis 3.0.6 to the swarm and configure the swarm with a 10 second update delay:

```sh
docker service create \
  --replicas 3 \
  --name redis \
  --update-delay 10s \
  redis:3.0.6
```

Verify service is create and see the `UpdateConfig`:

```sh
docker service inspect --pretty redis
```

```
# Result ==>

ID:		39847ewf0adwnxdna15dejxib
Name:		redis
Mode:		Replicated
 Replicas:	3
Placement:
UpdateConfig:
 Parallelism:	1
 Delay:		10s
 On failure:	pause
ContainerSpec:
 Image:		redis:3.0.6
Resources:
```

**Notes on update policy (`UpdateConfig`):**

* `--update-delay` flag configures the time delay between updates to a service task or sets of tasks (e.g. `10s`, `10m30s`, etc.)
* `--update-parallelism` flag configures the maximum number of service tasks that the scheduler updates simultaneously (default is `1`)
* `--update-failure-action` flag can be set to `pause` (stop update of tasks when one returns `FAILED`) or `continue` (ignore failed tasks)

Update redis to version 3.0.7.  The swarm manager applies the update to nodes according to the `UpdateConfig` policy:

```sh
docker service update --image redis:3.0.7 redis
```

Check the update progress:

```sh
docker service ps redis
```

```
# Result ==>

ID                         NAME         IMAGE        NODE      DESIRED STATE  CURRENT STATE            ERROR
9svkq9pkwjium1k4yedyfgnfo  redis.1      redis:3.0.7  manager1  Running        Preparing 5 seconds ago
1qp06o2rzo9ex80n3h1uer536   \_ redis.1  redis:3.0.6  manager1  Shutdown       Shutdown 5 seconds ago
e4qxqn6asvy0tgytw19zi3chu  redis.2      redis:3.0.6  worker2   Running        Preparing 3 minutes ago
asenzak1w439nzp9m97w3ta3c  redis.3      redis:3.0.6  worker1   Running        Preparing 3 minutes ago
```

Here we see the first task for the service `redis` is being replaced.

The scheduler applies rolling updates as follows by default:

1. Stop the first task.
2. Schedule update for the stopped task.
3. Start the container for the updated task.
4. If the update to a task returns RUNNING, wait for the specified delay period then stop the next task.
5. If, at any time during the update, a task returns FAILED, pause the update.

See the new image in the desired state:

```sh
docker service inspect --pretty redis
```

```
# Result ==>

ID:		39847ewf0adwnxdna15dejxib
Name:		redis
Mode:		Replicated
 Replicas:	3
Update status:
 State:		updating
 Started:	4 minutes ago
 Message:	update in progress
Placement:
UpdateConfig:
 Parallelism:	1
 Delay:		10s
 On failure:	pause
ContainerSpec:
 Image:		redis:3.0.7
Resources:
```

Here we can see the new image (`redis:3.0.7`) for the desired state and that an update is in progress.  Any failures will also be shown here in the `Update status` `message` section.

A failed update can be restarted by running another update:

```sh
docker service update redis
```

### About Docker Compose and swarm mode

Docker Compose is a useful tool for development purposes but not a good choice for production deployments.

**Why?**

* Docker Compose [does not support working with the new Docker swarm mode](http://stackoverflow.com/questions/38353959/docker-compose-with-docker-1-12-swarm-mode).
* You can use Compose with the the old Docker Swarm implementation to deploy into a multi-host environment; or
* You can use the new `docker-compose build` command to create a DAB then `docker stack deploy` to deploy the application stack into a Docker swarm mode cluster (note: experimental features)
* However, for microservices you probably would not want to deploy a whole application stack together in one go; you would deploy/upgrade each service/part independently*****

(*****) Technically, this is possible with Compose but services are a client-side concept not a server-side concept pre-swarm-mode:

```sh
docker-compose build <pseudo_service>
docker-compose up --no-deps -d <pseudo_service>
```

Next Steps
----------

Try out and read through the Swarm Tutorial [2](https://docs.docker.com/engine/swarm/swarm-tutorial/) (which this dojo is based upon).

Investigate DABs ([Distributed Application Bundles](https://blog.docker.com/2016/06/docker-app-bundle/)) which are an experimental feature in Docker 1.12.

References/Credits:
-------------------

[1] [Swarm mode overview](https://docs.docker.com/engine/swarm) -- more detailed info on Swarm mode can be found here.
[2] [Swarm mode tutorial](https://docs.docker.com/engine/swarm/swarm-tutorial/) -- much of the material in this dojo comes from the work described here.
