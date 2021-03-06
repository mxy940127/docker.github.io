---
title: Networking with overlay networks
description: Tutorials for networking with swarm services and standalone containers on multiple Docker daemons
keywords: networking, bridge, routing, ports, swarm, overlay
---

This series of tutorials deals with networking for swarm services.
For networking with standalone containers, see
[Networking with swarm services](network-tutorial-standalone.md). If you need to
learn more about Docker networking in general, see the [overview](index.md).

This topic includes four different tutorials. You can run each of them on
Linux, Windows, or a Mac, but for the last two, you need a second Docker
host running elsewhere.

- [Use the default overlay network](#use-the-default-overlay-network) demonstrates
  how to use the default overlay network that Docker sets up for you
  automatically when you initialize or join a swarm. This network is not the
  best choice for production systems.

- [Use user-defined overlay networks](#use-a-user-defined-overlay-network) shows
  how to create and use your own custom overlay networks, to connect services.
  This is recommended for services running in production.

- [Use an overlay network for standalone containers](#use-an-overlay-network-for-standalone-containers)
  shows how to communicate between standalone containers on different Docker
  daemons using an overlay network.

- [Communicate between a container and a swarm service](#communicate-between-a-container-and-a-swarm-service)
  sets up communication between a standalone container and a swarm service,
  using an attachable overlay network. This is supported in Docker 17.06 and
  higher.

## Prerequisites

These requires you to have at least a single-node swarm, which means that
you have started Docker and run `docker swarm init` on the host. You can run
the examples on a multi-node swarm as well.

The last example requires Docker 17.06 or higher.

## Use the default overlay network

In this example, you start an `alpine` service and examine the characteristics
of the network from the point of view of the individual service containers.

This tutorial does not go into operation-system-specific details about how
overlay networks are implemented, but focuses on how the overlay functions from
the point of view of a service.

### Prereqisites

This tutorial requires three physical or virtual Docker hosts which can all
communicate with one another, all running new installations of Docker 17.03 or
higher. This tutorial assumes that the three hosts are running on the same
network with no firewall involved.

These hosts will be referred to as `manager`, `worker-1`, and `worker-2`. The
`manager` host will function as both a manager and a worker, which means it can
both run service tasks and manage the swarm. `worker-1` and `worker-2` will
function as workers only,

If you don't have three hosts handy, an easy solution is to set up three
Ubuntu hosts on a cloud provider such as Amazon EC2, all on the same network
with all communications allowed to all hosts on that network (using a mechanism
such as EC2 security groups), and then to follow the
[installation instructions for Docker CE on Ubuntu](/engine/installation/linux/docker-ce/ubuntu.md).

### Walkthrough

#### Create the swarm

At the end of this procedure, all three Docker hosts will be joined to the swarm
and will be connected together using an overlay network called `ingress`.

1.  On `master`. initialize the swarm. If the host only has one network
    interface, the `--advertise-addr` flag is optional.

    ```bash
    $ docker swarm init --advertise-addr=<IP-ADDRESS-OF-MANAGER>
    ```

    Make a note of the text that is printed, as this contains the token that
    you will use to join `worker-1` and `worker-2` to the swarm. It is a good
    idea to store the token in a password manager.

2.  On `worker-1`, join the swarm. If the host only has one network interface,
    the `--advertise-addr` flag is optional.

    ```bash
    $ docker swarm --join --token <TOKEN> \
      --advertise-addr <IP-ADDRESS-OF-WORKER-1> \
      <IP-ADDRESS-OF-MANAGER>:2377
    ```

3.  On `worker-2`, join the swarm. If the host only has one network interface,
    the `--advertise-addr` flag is optional.

    ```bash
    $ docker swarm --join --token <TOKEN> \
      --advertise-addr <IP-ADDRESS-OF-WORKER-2> \
      <IP-ADDRESS-OF-MANAGER>:2377
    ```

4.  On `manager`, list all the nodes. This command can only be done from a
    manager.

    ```bash
    $ docker node ls

    ID                            HOSTNAME            STATUS              AVAILABILITY        MANAGER STATUS
    d68ace5iraw6whp7llvgjpu48 *   ip-172-31-34-146    Ready               Active              Leader
    nvp5rwavvb8lhdggo8fcf7plg     ip-172-31-35-151    Ready               Active              
    ouvx2l7qfcxisoyms8mtkgahw     ip-172-31-36-89     Ready               Active
    ```

    You can also use the `--filter` flag to filter by role:

    ```bash
    $ docker node ls --filter role=manager

    ID                            HOSTNAME            STATUS              AVAILABILITY        MANAGER STATUS
    d68ace5iraw6whp7llvgjpu48 *   ip-172-31-34-146    Ready               Active              Leader

    $ docker node ls --filter role=worker

    ID                            HOSTNAME            STATUS              AVAILABILITY        MANAGER STATUS
    nvp5rwavvb8lhdggo8fcf7plg     ip-172-31-35-151    Ready               Active              
    ouvx2l7qfcxisoyms8mtkgahw     ip-172-31-36-89     Ready               Active  
    ```

5.  List the Docker networks on `manager`, `worker-1`, and `worker-2` and notice
    that each of them now has an overlay network called `ingress` and a bridge
    network called `docker_gwbridge`. Only the listing for `manager` is shown
    here:

    ```bash
    $ docker network ls

    NETWORK ID          NAME                DRIVER              SCOPE
    495c570066be        bridge              bridge              local
    961c6cae9945        docker_gwbridge     bridge              local
    ff35ceda3643        host                host                local
    trtnl4tqnc3n        ingress             overlay             swarm
    c8357deec9cb        none                null                local
    ```

The `docker_gwbridge` connects the `ingress` network to the Docker host's
network interface so that traffic can flow to and from swarm managers and
workers. If you create swarm services and do not specify a network, they are
connected to the `ingress` network. It is recommended that you use separate
overlay networks for each application or group of applications which will work
together. In the next procedure, you will create two overlay networks and
connect a service to each of them.

#### Create the services

1.  On `manager`, create a new overlay network called `nginx-net`:

    ```bash
    $ docker network create -d overlay nginx-net
    ```

    You don't need to create the overlay network on the other nodes, beacause it
    will be automatically created when one of those nodes starts running a
    service task which requires it.

2.  On `manager`, create a 5-replica Nginx service connected to `nginx-net`. The
    service will publish port 80 to the outside world. All of the service
    task containers can communicate with each other without opening any ports.

    > **Note**: Services can only be created on a manager.

    ```bash
    $ docker service create \
      --name my-nginx \
      --publish target=80,published=80 \
      --replicas=5 \
      --network nginx-net \
      nginx
      ```

      The default publish mode of `ingress`, which is used when you do not
      specify a `mode` for the `--publish` flag, means that if you browse to
      port 80 on `manager`, `worker-1`, or `worker-2`, you will be connected to
      port 80 on one of the 5 service tasks, even if no tasks are currently
      running on the node you browse to. If you want to publish the port using
      `host` mode, you can add `mode=host` to the `--publish` output. However,
      you should also use `--mode global` instead of `--replicas=5` in this case,
      since only one service task can bind a given port on a given node.

3.  Run `docker service ls` to monitor the progress of service bring-up, which
    may take a few seconds.

4.  Inspect the `nginx-net` network on `master`, `worker-1`, and `worker-2`.
    Remember that you did not need to create it manually on `worker-1` and
    `worker-2` because Docker created it for you. The output will be long, but
    notice the `Containers` and `Peers` sections. `Containers` lists all
    service tasks (or standalone containers) connected to the overlay network
    from that host.

5.  From `manager`, inspect the service using `docker service inspect my-nginx`
    and notice the information about the ports and endpoints used by the
    service.

6.  Create a new network `nginx-net-2`, then update the service to use this
    network instead of `nginx-net`:

    ```bash
    $ docker network create -d overlay nginx-net-2
    ```

    ```bash
    $ docker service update \
      --network-add nginx-net-2 \
      --network-rm nginx-net \
      my-nginx
    ```

7.  Run `docker service ls` to verify that the service has been updated and all
    tasks have been redeployed. Run `docker network inspect nginx-net` to verify
    that no containers are connected to it. Run the same command for
    `nginx-net-2` and notice that all the service task containers are connected
    to it.

    > **Note**: Even though overlay networks are automatically created on swarm
    > worker nodes as needed, they are not automatically removed.

8.  Clean up the service and the networks. From `manager`, run the following
    commands. The manager will direct the workers to remove the networks
    automatically.

    ```bash
    $ docker service rm my-nginx
    $ docker network rm nginx-net nginx-net-2
    ```

## Use a user-defined overlay network

### Prerequisites

This tutorial assumes the swarm is already set up and you are on a manager.

### Walkthrough

1.  Create the user-defined overlay network.

    ```bash
    $ docker network create -d overlay my-overlay
    ```

2.  Start a service using the overlay network and publishing port 80 to port
    8080 on the Docker host.

    ```bash
    $ docker service create \
      --name my-nginx \
      --network my-overlay \
      --replicas 1 \
      --publish published=8080,target=80 \
      nginx:latest
    ```

3.  Run `docker network inspect my-overlay` and verify that the `my-nginx`
    service task is connected to it, by looking at the `Containers` section.

4.  Remove the service and the network.

    ```bash
    $ docker service rm my-nginx

    $ docker network rm my-overlay
    ```

## Use an overlay network for standalone containers

This example does the following:

- initializes a swarm on `host1`
- joins `host2` to the swarm
- creates an attachable overlay network
- creates an `alpine` service with 3 replicas, connected to the overlay network
- creates a single `alpine` container on `host2`, which is also attached to the
  overlay network
- proves that the standalone container can communicate with the service tasks,
  and vice versa.

### Prerequisites

For this test, you need two different Docker hosts, which can communicate with
each other. Each host needs to be running Docker 17.06 or higher. The following
ports must be open between the two Docker hosts:

- TCP port 2377
- TCP and UDP port 7946
- UDP port 4789

One easy way to set this is up is to have two VMs (either local or on a cloud
provider like AWS), each with Docker installed and running. If you're using AWS
or a similar cloud computing platform, the easiest configuration is to use a
security group which opens all incoming ports between the two hosts and the SSH
port from your client's IP address.

This example will refer to the hosts as `host1` and `host2`, and the command
prompts will be labelled accordingly.

The example uses Linux hosts, but the same commands work on Windows.

### Walk-through

1.  Set up the swarm.

    1.  On `host1`, run `docker swarm init`, specifying the IP address for the
        interface which will communicate with the other host (for instance, the
        private IP address on AWS).

        ```bash
        (host1) $ docker swarm init --advertise-addr 192.0.2.1

        Swarm initialized: current node (l9ozqg3m6gysdnemmhoychk9p) is now a manager.

        To add a worker to this swarm, run the following command:

            docker swarm join \
            --token SWMTKN-1-3mtj3k6tkuts4cpecpgjdvgj1u5jre5zwgiapox0tcjs1trqim-bfwb0ve6kf42go1rznrn0lycx \
            192.0.2.1:2377

        To add a manager to this swarm, run 'docker swarm join-token manager' and follow the instructions.
        ```

        The swarm is initialized and `host1` runs both manager and worker roles.

    2.  Copy the `docker swarm join` command. Open a new terminal, connect to
        `host2`, and execute the command. Add the `--advertise-addr` flag,
        specifying the IP address for the interface that will communicate with
        the other host (for instance, the private IP address on AWS). The
        last argument is the IP address of `host1`.

        ```bash
        (host2) $ docker swarm join \
                  --token SWMTKN-1-3mtj3k6tkuts4cpecpgjdvgj1u5jre5zwgiapox0tcjs1trqim-bfwb0ve6kf42go1rznrn0lycx \
                  --advertise-addr 192.0.2.2:2377 \
                  192.0.2.1:2377
        ```

        If the command succeeds, the following message is shown:

        ```none
        This node joined a swarm as a worker.
        ```

        Otherwise, the `docker swarm join` command will time out. In this case,
        run `docker swarm leave --force` on `node2`, verify your network and
        firewall settings, and try again.

2.  Create an attachable overlay network called `test-net` on `host1`.

    ```bash
    $ docker network create --driver=overlay --attachable test-net
    ```

    You don't need to manually create the overlay on `host2` because it will
    be created when a container or service tries to connect to it from `host2`.

3.  On `host1`, start a container that connects to `test-net`:

    ```bash
    (host1) $ docker run -dit \
              --name alpine1 \
              --network test-net \
              alpine
    ```

4.  On `host2`, start a container that connects to `test-net`:

    ```bash
    (host2) $ docker run -dit \
              --name alpine2 \
              --network test-net \
              alpine
    ```
    The `-dit` flags mean to start the container detached
    (in the background), interactive (with the ability to type into it), and
    with a TTY (so you can see the input and output).
    
    > **Note**: There is nothing to prevent you from using the same container
    > name on multiple hosts, but automatic service discovery will not work if
    > you do, and you will need to refer to the containers by IP address.

    Verify that `test-net` was created on `host2`:

    ```bash
    (host2) $ docker network ls

    NETWORK ID          NAME                DRIVER              SCOPE
    6e327b25443d        bridge              bridge              local
    10eda0b42471        docker_gwbridge     bridge              local
    1b16b7e2a72c        host                host                local
    lgsov6d3c6hh        ingress             overlay             swarm
    6af747d9ae1e        none                null                local
    uw9etrdymism        test-net            overlay             swarm
    ```

5.  Remember that you created `alpine1` from `host1` and `alpine2` from `host2`.
    Now, attach to `alpine2` from `host1`:

    ```bash
    (host1) $ docker container attach alpine2

    #
    ```

    Automatic service discovery worked between two containers across the overlay
    network!

    Within the attached session, try pinging `alpine1` from `alpine2`:

    ```bash
    # ping -c 2 alpine1

    PING alpine1 (10.0.0.2): 56 data bytes
    64 bytes from 10.0.0.2: seq=0 ttl=64 time=0.523 ms
    64 bytes from 10.0.0.2: seq=1 ttl=64 time=0.547 ms

    --- alpine1 ping statistics ---
    2 packets transmitted, 2 packets received, 0% packet loss
    round-trip min/avg/max = 0.523/0.535/0.547 ms
    ```

    This proves that the two containers can communicate with each other using
    the overlay network which is connecting `host1` and `host2`.

    Detach from `alpine2` using the `CTRL` + `P` `CTRL` + `Q` sequence.

6.  Stop the containers and remove `test-net` from each host. Because the Docker
    daemons are operating independently and these are standalone containers, you
    need to run the commands on the individual hosts.

    ```bash
    (host1) $ docker container stop alpine1
            $ docker container rm alpine1
            $ docker network rm test-net
    ```

    ```bash
    (host2) $ docker container stop alpine2
            $ docker container rm alpine2
            $ docker network rm test-net
    ```

## Communicate between a container and a swarm service

### Prerequisites

You need Docker 17.06 or higher for this example.

### Walkthrough

In this example, you start two different `alpine` containers on the same Docker
host and do some tests to understand how they communicate with each other. You
need to have Docker installed and running.

1.  Open a terminal window. List current networks before you do anything else.
    Here's what you should see if you've never added a network or initialized a
    swarm on this Docker daemon. You may see different networks, but you should
    at least see these (the network IDs will be different):

    ```bash
    $ docker network ls

    NETWORK ID          NAME                DRIVER              SCOPE
    17e324f45964        bridge              bridge              local
    6ed54d316334        host                host                local
    7092879f2cc8        none                null                local
    ```

    The default `bridge` network is listed, along with `host` and `none`. The
    latter two are not fully-fledged networks, but are used to start a container
    connected directly to the Docker daemon host's networking stack, or to start
    a container with no network devices. **This tutorial will connect two
    containers to the `bridge` network.**

2.  Start two `alpine` containers running `ash`, which is Alpine's default shell
    rather than `bash`. The `-dit` flags mean to start the container detached
    (in the background), interactive (with the ability to type into it), and
    with a TTY (so you can see the input and output). Since you are starting it
    detached, you won't be connected to the container right away. Instead, the
    container's ID will be printed. Because you have not specified any
    `--network` flags, the containers connect to the default `bridge` network.

    ```bash
    $ docker run -dit --name alpine1 alpine ash

    $ docker run -dit --name alpine2 alpine ash
    ```

    Check that both containers are actually started:

    ```bash
    $ docker container ls

    CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS               NAMES
    602dbf1edc81        alpine              "ash"               4 seconds ago       Up 3 seconds                            alpine2
    da33b7aa74b0        alpine              "ash"               17 seconds ago      Up 16 seconds                           alpine1
    ```

3.  Inspect the `bridge` network to see what containers are connected to it.

    ```bash
    $ docker network inspect bridge

    [
        {
            "Name": "bridge",
            "Id": "17e324f459648a9baaea32b248d3884da102dde19396c25b30ec800068ce6b10",
            "Created": "2017-06-22T20:27:43.826654485Z",
            "Scope": "local",
            "Driver": "bridge",
            "EnableIPv6": false,
            "IPAM": {
                "Driver": "default",
                "Options": null,
                "Config": [
                    {
                        "Subnet": "172.17.0.0/16",
                        "Gateway": "172.17.0.1"
                    }
                ]
            },
            "Internal": false,
            "Attachable": false,
            "Containers": {
                "602dbf1edc81813304b6cf0a647e65333dc6fe6ee6ed572dc0f686a3307c6a2c": {
                    "Name": "alpine2",
                    "EndpointID": "03b6aafb7ca4d7e531e292901b43719c0e34cc7eef565b38a6bf84acf50f38cd",
                    "MacAddress": "02:42:ac:11:00:03",
                    "IPv4Address": "172.17.0.3/16",
                    "IPv6Address": ""
                },
                "da33b7aa74b0bf3bda3ebd502d404320ca112a268aafe05b4851d1e3312ed168": {
                    "Name": "alpine1",
                    "EndpointID": "46c044a645d6afc42ddd7857d19e9dcfb89ad790afb5c239a35ac0af5e8a5bc5",
                    "MacAddress": "02:42:ac:11:00:02",
                    "IPv4Address": "172.17.0.2/16",
                    "IPv6Address": ""
                }
            },
            "Options": {
                "com.docker.network.bridge.default_bridge": "true",
                "com.docker.network.bridge.enable_icc": "true",
                "com.docker.network.bridge.enable_ip_masquerade": "true",
                "com.docker.network.bridge.host_binding_ipv4": "0.0.0.0",
                "com.docker.network.bridge.name": "docker0",
                "com.docker.network.driver.mtu": "1500"
            },
            "Labels": {}
        }
    ]
    ```

    Near the top, information about the `bridge` network is listed, including
    the IP address of the gateway between the Docker host and the `bridge`
    network (`172.17.0.1`). Under the `Containers` key, each connected container
    is listed, along with information about its IP address (`172.17.0.2` for
    `alpine1` and `172.17.0.3` for `alpine2`).

4.  The containers are running in the background. Use the `docker attach`
    command to connect to `alpine1`.

    ```bash
    $ docker attach alpine1

    / #
    ```

    The prompt changes to `#` to indicate that you are the `root` user within
    the container. Use the `ip addr show` command to show the network interfaces
    for `alpine1` as they look from within the container:

    ```bash
    # ip addr show

    1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN qlen 1
        link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
        inet 127.0.0.1/8 scope host lo
           valid_lft forever preferred_lft forever
        inet6 ::1/128 scope host
           valid_lft forever preferred_lft forever
    27: eth0@if28: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 1500 qdisc noqueue state UP
        link/ether 02:42:ac:11:00:02 brd ff:ff:ff:ff:ff:ff
        inet 172.17.0.2/16 scope global eth0
           valid_lft forever preferred_lft forever
        inet6 fe80::42:acff:fe11:2/64 scope link
           valid_lft forever preferred_lft forever
    ```

    The first interface is the loopback device. Ignore it for now. Notice that
    the second interface has the IP address `172.17.0.2`, which is the same
    address shown for `alpine1` in the previous step.

5.  From within `alpine1`, make sure you can connect to the internet by
    pinging `google.com`. The `-c 2` flag limits the command two two `ping`
    attempts.

    ```bash
    # ping -c 2 google.com

    PING google.com (172.217.3.174): 56 data bytes
    64 bytes from 172.217.3.174: seq=0 ttl=41 time=9.841 ms
    64 bytes from 172.217.3.174: seq=1 ttl=41 time=9.897 ms

    --- google.com ping statistics ---
    2 packets transmitted, 2 packets received, 0% packet loss
    round-trip min/avg/max = 9.841/9.869/9.897 ms
    ```

6.  Now try to ping the second container. First, ping it by its IP address,
    `172.17.0.3`:

    ```bash
    # ping -c 2 172.17.0.3

    PING 172.17.0.3 (172.17.0.3): 56 data bytes
    64 bytes from 172.17.0.3: seq=0 ttl=64 time=0.086 ms
    64 bytes from 172.17.0.3: seq=1 ttl=64 time=0.094 ms

    --- 172.17.0.3 ping statistics ---
    2 packets transmitted, 2 packets received, 0% packet loss
    round-trip min/avg/max = 0.086/0.090/0.094 ms
    ```

    This succeeds. Next, try pinging the `alpine2` container by container
    name. This will fail.

    ```bash
    # ping -c 2 alpine2

    ping: bad address 'alpine2'
    ```

7.  Detach from `alpine2` without stopping it by using the detach sequence,
    `CTRL` + `p` `CTRL` + `q` (hold down `CTRL` and type `p` followed by `q`).
    If you wish, attach to `alpine2` and repeat steps 4, 5, and 6 there,
    substituting `alpine1` for `alpine2`.

8.  Stop and remove both containers.

    ```bash
    $ docker container stop alpine1 alpine2
    $ docker container rm alpine1 alpine2
    ```

Remember, the default `bridge` network is not recommended for production. To
learn about user-defined bridge networks, continue to the
[next tutorial](#use-user-defined-bridge-networks).

## Other networking tutorials

Now that you have completed the networking tutorials for overlay networks,
you might want to run through these other networking tutorials:

- [Host networking tutorial](network-tutorial-host.md)
- [Standalone networking tutorial](network-tutorial-standalone.md)
- [Macvlan networking tutorial](network-tutorial-macvlan.md)

