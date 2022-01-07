- Feature Name: `docker-desktop-replacement`
- Start Date: 2022-01-06
- RFC PR: [mbta/technology-docs#0000](https://github.com/mbta/technology-docs/pull/0000)
- Asana task: [Docker Desktop situation](https://app.asana.com/0/1200506724882024/1201470397690247)
- Status: Proposed

# Summary
[summary]: #summary

Standardize on Podman for working with containers: https://podman.io/

# Motivation
[motivation]: #motivation

As of August 2021, Docker Desktop requires a license ($5 to $21/month/developer)
for professional use.
[https://www.docker.com/blog/updating-product-subscriptions/]. As the MBTA is a
large organization, we need to either pay for the licenses or find a new option
for development. Many of our developers use Docker Desktop to build and run
containers, and we would like to continue that style of development.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

[Podman](https://podman.io) is a tool for working with containers: building them
and running them, in particular. It provides a command line interface (CLI)
which is compatible with the Docker CLI, at the level we currently use it.

Instead of installing Docker Desktop and the Docker CLI tools, developers will
install Podman and use it to work with containers on their local machines. In
CI/CD and in AWS, we will continue to use Docker.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

Podman (like similar tools on OS X) works by running a virtualized Linux VM, and
then running the containers inside that VM.

To install Podman:

```bash
brew install podman
podman machine init --disk-size 20
```

To build/run a container:

```bash
podman build -t <tag> .
podman run <tag>
```

Port forwarding (with `-p`) and interactive use (with `-it`) work as expected.

If you want to continue to use `docker` on the command line, you can alias it:
```bash
alias docker=podman
```

# Drawbacks
[drawbacks]: #drawbacks

Podman provides a CLI-compatible interface, but it's possible that there are
Docker CLI options which are different or that Podman does not support.

@paulswartz ran into an issue where the clock in the virtual machine got out of sync. To re-sync them:

```bash
podman machine ssh "sudo systemctl restart chronyd && timedatectl --adjust-system-clock"
```

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

## Minikube

[Minikube](https://minikube.sigs.k8s.io/docs/start/) is a local Kubernetes instance, which also provides a Docker-compatible API.

```bash
brew install minikube hyperkit docker
minikube start --driver=hyperkit
source (minikube docker-env)

docker build -t <tag> .
docker run <tag>
```

However, in order to connect to the container, you need to use `minikube ip` to
get the separate IP address to use: `localhost` or `127.0.0.1` do not work.

## Paying for licenses

This would allow us to continue using Docker Desktop, but would need to go through the procurement process.

## Giving developers Linux machines

This would also need to go through procurement. It would also involve everyone learning a lot more about Linux.

# Prior art
[prior-art]: #prior-art

- Docker Desktop alternatives: https://matt-rickard.com/docker-desktop-alternatives/
- Top Docker alternatives: https://blog.logrocket.com/top-docker-alternatives-2022/

# Unresolved questions
[unresolved-questions]: #unresolved-questions

- Are there other Docker features we need to validate before making this decision?

# Future possibilities
[future-possibilities]: #future-possibilities

In the future, I could see CTD providing a setup script (similar to
[thoughtbot/laptop](https://github.com/thoughtbot/laptop)) which automates all
of this for new developers during onboarding.
