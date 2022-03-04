- Feature Name: `docker-desktop-replacement`
- Start Date: 2022-01-06
- RFC PR: [mbta/technology-docs#0010](https://github.com/mbta/technology-docs/pull/0010)
- Asana task: [Docker Desktop situation](https://app.asana.com/0/1200506724882024/1201470397690247)
- Status: Accepted

# Summary
[summary]: #summary

Standardize on [Colima][colima] for working with Docker on macOS.

[colima]: https://github.com/abiosoft/colima

# Motivation
[motivation]: #motivation

As of [August 2021][docker-subscriptions], Docker Desktop requires a license ($5
to $21/month/developer) for professional use. As the MBTA is a large
organization, we need to either pay for the licenses or find a new option for
development. Many of our developers use Docker Desktop to build and run
containers, and we would like to continue that style of development.

[docker-subscriptions]: https://www.docker.com/blog/updating-product-subscriptions/

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

[Colima][colima] provides a Docker-compatible runtime for containers on macOS.
It supports port forwarding and read-only volume mounting out of the box.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

Colima (like similar tools on OS X) works by running a virtualized Linux VM, and
then running the containers inside that VM.

To install Colima:

```bash
brew install colima docker
colima start
```

To build/run a container:

```bash
docker build -t <tag> .
docker run <tag>
```

Port forwarding (with `-p`), read-only volume mounting (with `-v`), and interactive use (with `-it`) work as expected.

# Drawbacks
[drawbacks]: #drawbacks

Writable volume mounts do not work out of the box (issues
[#83](https://github.com/abiosoft/colima/issues/83) and
[#102](https://github.com/abiosoft/colima/issues/102)). However, there are
workarounds listed in those tickets that have not been fully investigated.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

## Podman

[Podman](https://podman.io) is a tool for working with containers: building and
running them, in particular. It provides a command line interface (CLI) which is
compatible with the Docker CLI.

```bash
brew install podman
podman machine init --disk-size 20
podman machine start

podman build -t <tag> .
podman run <tag>
```

However, it doesn't currently support mounting volumes, which several developers use.

## Minikube

[Minikube][minikube] is a local Kubernetes instance, which also provides a Docker-compatible API.

```bash
brew install minikube hyperkit docker
minikube start --driver=hyperkit
source $(minikube docker-env)

docker build -t <tag> .
docker run <tag>
```

However, in order to connect to the container, you need to use `minikube ip` to
get the separate IP address to use: `localhost` or `127.0.0.1` do not work.

[minikube]: https://minikube.sigs.k8s.io/docs/start/

## Paying for licenses

This would allow us to continue using Docker Desktop, but would need to go through the procurement process.

## Giving developers Linux machines

This would also need to go through procurement. It would also involve everyone learning a lot more about Linux.

# Prior art
[prior-art]: #prior-art

- Docker Desktop alternatives: https://matt-rickard.com/docker-desktop-alternatives/
- Top Docker alternatives: https://blog.logrocket.com/top-docker-alternatives-2022/
- Using Docker with Multipass: https://ubuntu.com/blog/replacing-docker-desktop-on-windows-and-mac-with-multipass 

# Unresolved questions
[unresolved-questions]: #unresolved-questions

- ~Are there other Docker features we need to validate before making this decision?~
- Should we procure Docker Desktop in the future?

# Future possibilities
[future-possibilities]: #future-possibilities

In the future, I could see CTD providing a setup script (similar to
[thoughtbot/laptop](https://github.com/thoughtbot/laptop)) which automates all
of this for new developers during onboarding.
