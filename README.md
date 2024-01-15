# HA Postgres with pgvector on fly.io

This repo is a forked and extended base image for postgres meant to include pgvector as an additional dependency. This in effect enables to use "fly pg" as normal with the exception of upgrading (i.e. via "fly pg update"). You'll need to maintain the image and manually upgrade.

## Deploying with the extended pg image

1. Ensure authentication for both fly, dockerhub, and fly+dockerhub

```bash
fly auth login
fly auth docker
```

2. Build and push the extended base image

```bash
docker build . -t <dockerhub-username>/pgvector --platform "linux/amd64"
docker push <dockerhub-username>/pgvector

# 3. Make a new cheapskate deployment
```

3. Deploy the extended image

```bash
fly pg create --name exjobb-db \
    --image-ref stigsvensson/pgvector \
    --region arn --volume-size 1 \
    --initial-cluster-size 1 \
    --vm-size shared-cpu-1x \
```

4. Toggle extension and validate connection

```bash
# database is in private network, you connect via proxy
fly pg connect -a exjobb-db

# toggle extension
postgres> CREATE EXTENSION vector;

# validate that extension does indeed show up
postgres> \dx
```

# High Availability Postgres on Fly.io

This repo contains all the code and configuration necessary to run a [highly available Postgres cluster](https://fly.io/docs/postgres/) in a Fly.io organization's private network. This source is packaged into [Docker images](https://hub.docker.com/r/flyio/postgres-flex/tags) which allow you to track and upgrade versions cleanly as new features are added.

## Getting started

```bash
# Be sure you're running the latest version of flyctl.
fly version update

# Provision a 3 member cluster
fly pg create --name <app-name> --initial-cluster-size 3 --region ord --flex
```

## High Availability

For HA, it's recommended that you run at least 3 members within your primary region. Automatic failovers will only consider members residing within your primary region. The primary region is represented as an environment variable defined within the `fly.toml` file.

## Horizontal scaling

Use the clone command to scale up your cluster.

```
# List your active Machines
fly machines list --app <app-name>

# Clone a machine into a target region
fly machines clone <machine-id> --region <target-region>
```

## Staying up-to-date!

This project is in active development so it's important to stay current with the latest changes and bug fixes.

```
# Use the following command to verify you're on the latest version.
fly image show --app <app-name>

# Update your Machines to the latest version.
fly image update --app <app-name>

```

## TimescaleDB support

We currently maintain a separate TimescaleDB-enabled image that you can specify at provision time.

```
fly pg create  --image-ref flyio/postgres-flex-timescaledb:15
```

## Having trouble?

Create an issue or ask a question here: https://community.fly.io/

## Contributing

If you're looking to get involved, fork the project and send pull requests.
