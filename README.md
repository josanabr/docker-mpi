# docker-mpi

MPI cluster on Docker Swarm.

Another (more complete solution) is available here [https://github.com/NLKNguyen/alpine-mpich](https://github.com/NLKNguyen/alpine-mpich)

Do you have two or more physical machines and want to try your personal MPI cluster for your calculations? This toy project might be useful for you! Setup your Docker Swarm cluster, deploy mpistack image, attach to the master container and simply run your MPI program on multiple hosts.


# Prerequisites
Those things you need to do only once.

Setup Docker Sworm network
(details: TBD)

Set up registry container (this is the approach to distribute image among nodes):
```
docker service create --name registry --publish published=5000,target=5000 registry:2
```

To validate if service is running:

```
docker service ps registry
```

Now we are ready to push your image there. 

# Launching MPI cluster 

Build and push stackmpi image to the registry:
```
docker compose build
docker compose push
```
stackmpi is based on Ubuntu 20.04 with MPICH installed. If you need any additianal software available on each node, modify [`Dockerfile`](Dockerfile) appropriately and build+push image again.

**REMARK**: If you restart your machine, from some reason the image needs to be pushed to the repository again.

There are a few helper scripts available:

- `start-stack` and `stop-stack` starts or stops all nodes (docker stack) specified in docker-compose.yml. By default there is one _master_ node started on _manager_ node (assume this is current one) and 2 _worker_ nodes. **If you want more workers**, go to [`docker-compose.yml`](docker-compose.yml) file and modify `replicas` attribute in `worker` section.

- `attach-stack` attaches to the _master_ container. 

- `ssh-stack` allows to connect via SSH to the _master_. However, SSH port is not exposed therefore in order to expose the SSH port, you need to  uncomment  the `port` section in the `master` section at [`docker-compose.yml`](docker-compose.yml).

Inside master node, there are a few aditional commands available:

- `node-master` displays IP of master node,

- `node-workers` displays list of IPs of worker nodes,

- `machines` get list of IPs of all nodes (master and workers) and put it into the file `/root/machines` (yep, not perfect),

<!-- `sync` replicates directory `/root/project/` to all nodes at the same location (synchronization of nodes using `rsync`). Directory `docker-mpi/project/` (`docker-mpi` repository) is mount as a volume to `/root/project/` and acts as a directory for your project. You should put here binaries of your program and any necessary resources. -->

# Executing MPI program
Attach to the master container with `attach-stack` command and fill the file `/root/machines` with the list of IPs of all nodes with `machines` command. 
Now you are ready to run simplest MPI program:

```
mpiexec -f machinefile -n 3 hostname
```
The output might look like this
```
root@master:~# mpiexec -f machinefile -n 4 hostname
master
worker
master
worker
```

Now lets try something closer to the reality. Go to `/root/project/` and build sample program with `make`. Once the compilation is successful then copy `test-mpi` to the `/shared-directory`. Execute the program with:

```
mpiexec -f machinefile -n 2 /shared_dir/test-mpi
```

The output might look like this
```
root@master:~# mpiexec -f machinefile -n 2 /shared_dir/test-mpi 
Processor master, rank 1 out of 2 processors. CPUs: 4   CPUs available: 4
Processor master, rank 0 out of 2 processors. CPUs: 4   CPUs available: 4
```

