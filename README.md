# interlock-demo

###Pre-requisites
- Docker Swarm should be configured, up & running. There should exist at least two other nodes in the cluster.
- Run `export DOCKER_HOST=tcp://<swarm-manager-ip>:<swarm-manager-exposed-port>`
- `docker info` should run on all the cluster nodes and should show information about the cluster.
- For this demo, we are not going to be using TLS / SSL, hence verify DOCKER_TLS_VERIFY is unset.

###Setup
Three docker hosts with docker-cs 1.6 or docker open source 1.8.1 or later installed.
- node0 - This will be setup as the swarm manager (or also known as swarm master)
- node1 and node2 - These will be other nodes that will be joined to the cluster. The swarm manager (node0) itself can also be a member of the cluster.

###Steps
1. On any node that is part of the cluster, run a container from the ehazlett/interlock image using the command below. We want to use the haproxy plugin for this lab.
  ```
  docker run -p 80:80 --name lb0 -d ehazlett/interlock --swarm-url $DOCKER_HOST --plugin haproxy start
  ```
2. Due to the cluster scheduling, the haproxy container may actually be running on a different host than the one where the above command was run. Use `docker ps | grep interlock` to identify the host it is running on.
  - Alternatively, specify a filter (ie., affinity:nodename or constraint:container) to restrict the container to a specific docker host. ~~It seems to make sense to run the load balancer(s) on the same set of hosts that host the swarm manager(s).~~

  *The following steps will assume that the interlock/haproxy container was started on `node1` .*

3. Ensure DNS is setup (or update /etc/hosts on your client machine) so as to ensure that node1 is resolvable.

4.  On the client machine, open a browser and point it to http://stats:interlock@[node1-ip]/haproxy?stats. This should show the stats page for the haproxy container. Note that there is only a frontend listening on port 80 and no backends registered to perform any work.

5. Let's now spin up a few webserver containers. Run the following command, on any node in the cluster:
  ```
  for i in {0..9}; do docker run -d -p 80$i:80 --hostname web.docker.demo \
  --name www$i nginx; done
  ```
  - This will spawn 10 nginx containers, each having a unique name, but bound to the same hostname and exposing port 80 which is mapped to a unique port on the host. Depending on the clustering strategy used (the default is spread), the 10 containers will be distributed across the cluster.
  - Check the URL again as in the previous step. You should see the backends automatically registering themselves with the haproxy load balancer.
  - If you are following the haproxy logs `docker logs -f lb0`, you should see a lot of event based activity scroll up. The interlock container listens on the swarm manager's port for all events to query, add or detach containers.

6. Let's now run an actual demo website to see this in action. This can practically be any container that runs a website or networked aplication. Run the command below:

  ```
  for i in {1..2}; do docker run -d -P --hostname docker-training.com \
  --name website$i ehazlett/docker-demo; done
  ```
7. On the client machine (the host where you run the internet browser), add docker-training.com as an alias to the node1 host in /etc/hosts.

8. Browse to http://docker-training.com and you should see the website up and running. It should display the docker logo.

9. On any node in the cluster, run `docker ps | grep website`. Note that the containers are arbitrarily distributed and run on random ports. Go ahead and bring one of the containers down by running `docker stop website1`

10. Go back to the browser and hit refresh, the site should still be up. You can also tail the container logs to see requests being served from the container that is up and running `docker logs -f website2`.

#### Known issues & lessons learned in Swarm:
1. When joining nodes to the cluster (using docker run -d swarm join --addr...), the = between addr is optional.
2. When joining nodes to the cluster (using docker run -d swarm join --addr...), do NOT use tcp:// after --addr.
3. When joining nodes to the cluster (using docker run -d swarm join --addr...), on AWS at least use only the DNS name after --addr.
4. tcp:// is required in the DOCKER_HOST env variable. [e.g.: export DOCKER_HOST=tcp://localhost:9999 ]

