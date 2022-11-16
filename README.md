# RonDB with Docker

This repository creates the possibility of:
- building cross-platform RonDB images
- running local (non-production) RonDB clusters with docker-compose
- demo the usage of managed RonDB (upcoming)

To learn more about RonDB, have a look [here](rondb.com).

## Quickstart

Dependencies:
- Docker, docker-compose, Docker Buildx

Important:
- Every container requires an amount of memory; to adjust the amount of resources that Docker allocates to each of the different containers, see the [docker.env](docker.env) file. To check the amount actually allocated for the respective containers, run `docker stats` after having started a docker-compose instance. To adjust the allowed memory limits for Docker containers, do as described [here](https://stackoverflow.com/a/44533437/9068781). It should add up to the reserved aggregate amount of memory required by all Docker containers. As a reference, allocating around 27GB of memory in the Docker settings can support 1 mgmd, 2 mysqlds and 9 data nodes (3 node groups x 3 replicas).
- The same can apply to disk space - Docker also defines a maximum storage that all containers can use in the settings. It could however also be that a previous RonDB cluster run (or entirely different Docker containers) are still occupying disk space. In this case, run `docker container prune` and `docker volume prune`.
- This repository requires a tarball of the RonDB installation to run. Pre-built binaries can be found on [repo.hops.works](https://repo.hops.works/master). Make sure the target platform of the Docker image and the used tarball are identical.

Commands to run:
```bash
# Build and run image **for local platform** in docker-compose using local RonDB tarball (download it first!)
# Beware that the local platform is linux/arm64 in this case
./build_run_docker.sh \
  --rondb-tarball-is-local \
  -ruri ./rondb-21.04.9-linux-glibc2.35-arm64_v8.tar.gz \
  -v 21.04.9 -m 1 -d 2 -r 2 -my 1 -a 1

# Build cross-platform image (linux/arm64 here)
docker buildx build . --platform=linux/arm64 -t rondb-standalone:21.04.6 \
  --build-arg RONDB_VERSION=21.04.6 \
  --build-arg RONDB_TARBALL_LOCAL_REMOTE=remote \  # alternatively "local"
  --build-arg RONDB_TARBALL_URI=https://repo.hops.works/master/rondb-21.04.6-linux-glibc2.31-arm64_v8.tar.gz # alternatively a local file path

# Explore image
docker run --rm -it --entrypoint=/bin/bash rondb-standalone:21.04.6
```

Exemplatory commands to run with running docker-compose cluster:
```bash
# Check current ongoing memory consumption of running cluster
docker stats

# Open shell inside a running container
docker exec -it <container-id> /bin/bash

# If inside mgmd container; check the live cluster configuration:
ndb_mgm -e show

# If inside mysqld container; open mysql client:
mysql -uroot
```

## Making configuration changes

For each run of `./build_run_docker.sh`, we generate a fresh
- docker-compose file
- MySQL-server configuration file (my.cnf)
- RonDB configuration file (config.ini)
- (Multiple) benchmarking configuration files for Sysbench & DBT2

When attempting to change any of the configurations inside my.cnf or config.ini, ***do not*** change these in the autogenerated files. They will simply be overwritten with every run. Change them in [resources/config_templates](resources/config_templates).

## Running Benchmarks

The Docker images come with a set of benchmarks pre-installed. To run any of these benchmarks with the default configurations, run:

```bash
# The benchmarks are run on the API containers and make queries towards the mysqld containers; this means that both types are needed.
./build_run_docker.sh \
  --rondb-tarball-is-local \
  -ruri ./rondb-21.04.9-linux-glibc2.35-arm64_v8.tar.gz \
  -v 21.04.9 -m 1 -d 2 -r 2 -my 1 -a 1 \
  --run-benchmark <sysbench_single, sysbench_multi, dbt2_single, dbt2_multi>
```

To run benchmarks with custom settings, omit the `--run-benchmark` flag and open a shell in a running API container of a running cluster. See the RonDB documentation on running benchmarks to change the benchmark configuration files. The directory structure is equivalent to the directory structure found on Hopsworks clusters.

It may be the case that the benchmarks require more DataMemory than is available. You can change the config.ini or the benchmarking configuration files as discussed above to account for this.

*Note*: Benchmarking RonDB with a docker-compose setup on a single machine may not bring optimal performance results. This is because both the mysqlds and the ndbmtds (multi-threaded data nodes) scale in performance with more CPUs. In a production setting, each of these programs would be deployed on their own VM, whereby mysqlds and ndbmtds will scale linearly with up to 32 cores. The possibility of benchmarking was added here to give the user an introduction of benchmarking RonDB without needing to spin up a cluster with VMs.

## Goals of this repository

1. Create an image with RonDB installed "hopsworks/rondb-standalone:21.04.9"
   - Purpose: basic local testing & building stone for other images
   - No building of RonDB itself
   - Supporting multiple CPU architectures
   - No ndb-agent; no reconfiguration / online software upgrades / backups, etc.
   - Push image to hopsworks/mronstro registry
   - Has all directories setup for RonDB; setup like in Hopsworks
   - Is the base-image from which other binaries can be copied into
   - Useable for quick-start of RonDB
   - Need:
     - all RonDB scripts
     - dynamic setup of config.ini/my.cnf
     - dynamic setup of docker-compose file
     - standalone entrypoints

2. Create an image with ndb-agent installed "hopsworks/rondb-managed:21.04.9-1.0"
   - use "rondb-standalone" as base image
   - use this for demos of how upgrades/scaling/backups of RonDB can be used in the cloud
   - use this for testing managed RonDB to avoid the necessity of a Hopsworks cluster
   - install other required software there such as systemctl

3. Reference in ePipe as base image
    - create builder image to build ePipe itself
    - copy over ePipe binary into hopsworks/rondb

## TODO

- Change to #node-groups x #replFactor
- Are env files even needed in this image?
  - Add ndb-cluster-connection-pool-nodeids as env to Dockerfile
