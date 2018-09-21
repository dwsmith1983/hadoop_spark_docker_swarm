# Resources
I had to adapt my set up from many different
configurations. Here is the short list of what I can
remember.
* [big-data-europe](https://github.com/big-data-europe)
* [Getty Images](https://github.com/gettyimages)
* [SequenceIQ](https://github.com/sequenceiq)
* [Jamie Pillora](https://github.com/jpillora) for
DNSmasq
* [Portainer](https://portainer.io/)

Before you start using these images or `Dockerfiles`,
make sure to go through the scripts, configuration
files, and `Dockerfiles` looking for network specific
or user specific settings to change. Most will be
marked `<name>`.

# Required Files Hadoop/Spark Python Docker Swarm
+ DNSmasq directory
  - dnsmasq.conf
  - dnsmasq-run.sh
+ Hadoop directory
  - Dockerfile
  - entrypoint.sh
  - datanode directory
    * Dockerfile
    * run.sh
  - namenode directory
    * Dockerfile
    * run.sh
+ Spark directory
  - Dockerfile
+ Anaconda Python directory
  - Dockerfile
+ Data Science Dockerfile
  - Dockerfile
  - requirements.txt
  - conda\_requirement\_install.sh
+ build.sh
+ docker-compose-hadoop.yml
+ docker-compose-spark.yml
+ hadoop.env
+ pyspark_example.py

# Build
Run the build script in the current directory containing cluster directories.
The build scripts has six arguments. In our case, we would name them:
1. hadoop
2. namenode
3. datanode
4. spark
5. anaconda
6. datascience

The usages would be
```bash
sh ./build.sh hadoop namenode datanode spark anaconda datascience
```
where the argument names are optional. The default values are shown above.
This will build, tag, and push our images to our docker registry at
`<dns_name>:5000`.

# Setup
## DNSmasq
1. Move the `.conf` and `.sh` files to desired machine to act as the server.
2. Setup the desired configuration in `.conf`.
3. Run `sh ./dnsmasq-run.sh`.

## Hadoop Docker Swarm
1. Update the `docker-compose.yml` to the desired settings, specify the
image names, and set the replica number for the datanodes. Replica must be
equal to or less than the number of available datanodes.
2. Update the `hadoop.env` file to configure Hadoop.
3. On the machine that will be used as the manager node, launch the docker
compose file with
   ```bash
   docker stack deploy -c <file.yml> <name of network>
   docker stack deploy -c docker-compose-hadoop.yml hadoop
   ```
4. For the first run on the system, the `join-token` will be needed for the
workers. On the manager node, run `docker swarm join-token worker`  to see
the token.
5. ssh into all the worker nodes and paste the token into the command line.
6. Enter the container with `docker exec -it hadoop_swarm_namenode.... bash`
note that you can auto-complete the container name with `tab`.
7. Test Hadoop is running correctly with
`$HADOOP_HOME/bin/hdfs dfs -cat /user/test.md` which will print out `Success`
to the command line.
8. To drop a worker, ssh into the desired machine and enter
`docker swarm leave`.

## Spark Docker Swarm
1. Update the `docker-compose.yml` to the desired settings, specify the
image names, and set the replica number for the workers. Replica must be
equal to or less than the number of available worker nodes.
2. On the manager node, launch the docker compose file with the same deploy
command but change the `<file.yml>` and `<name of network>`.

# Shutting Down the Cluster
1. Get a list of the running services with `docker service ls`.
2. Then run `docker service rm <name, name, ...>` to rm the service.

# Enabling Portainer
Once the cluster is running, we access the manager node and enter the following
commands:
1. ```bash
   docker service create --name portainer_agent --network
   hadoop-spark-swarm-network -e AGENT_CLUSTER_ADDR=task.portainer_agent --mode
   global --constraint 'node.platform.os==linux' --mount type=bind,
   src=//var/run/docker.sock, dst=/var/run/docker.sock --mount type=bind,
   src=//var/lib/docker/volumes, dst=/var/lib/docker/volumes portainer/agent
   ```
2. ```bash
   docker service create --name portainer --publish 9000:9000 --network
   hadoop-spark-swarm-network --replicas=1 --constraint 'node.role==manager'
   portainer/portainer -H "tcp://tasks.portainer_agent:9001" --tlsskipverify
   ```
3. Check portainer.io `<dns_name>:9000` for the join status of the workers.

# Enable NFS
In the `docker-compose-spark.yml`, we needed to add
~~~~
volumes:
  nfsshare:
    driver: local
    driver_opts:
      type: "nfs"
      o: "addr=<ip_add>,nfsvers=4"
      device: ":/"
~~~~
Next, log into the manager node, `<dns_name>` and run
```bash
docker run -e NFS_EXPORT_0='/nfsshare
*(rw,fsid=0,async,no_subtree_check,no_auth_nlm,insecure,no_root_squash)'
--privileged -p 2049:2049 -d --name glcalcs erichough/nfs-server
```

# Scaling
Once the cluster is running, if a new worker needs to be added, we can run
```
docker service scale <container_name>=<num>
```
where `<container_name>` is correct container you would like to spin up
another worker on (Hadoop, Spark) and `<num>` is an integer greater than what
was set in the `docker-compose.yml` as replica but less than or equal to the
total number of workers available. For example, if we have a cluster of `5`
workers and would like to scale Spark to `7` workers, we would run
```
docker service scale spark_swarm_worker=7
```

# Executing Python
## Replace the Notebook token with a Password
By replacing the token with a password, we can
store `http://dns_name:8888` into our browser and
simple navigate to this address as opposed to
copying the address and token when Jupyter lab or
notebook launches.

This is optional. Simply comment out the lines at the
bottom of the `Dockerfile` if you don't want to set
this up.
1. Run `jupyter notebook --generate-config` to
generate the config file.
2. Run `jupyter notebook password` and enter your
password twice.
3. Copy `jupyter_notebook_config.py` to the anaconda
folder containing the `Dockerfile`.
4. Build the docker.

## Jupyter Lab
We can run a jupyter notebook or lab from the bash of the anaconda or
datascience images. From the bash, run the command
```bash
jupyter-lab --allow-root --ip=0.0.0.0
```
Then copy the URL into your local web browser and change the ip address to
host machines ip or DNS name.

## Python Script