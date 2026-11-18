<p align="center">
  <a href="https://github.com/lupaxa-docker-toolbox">
    <img src="https://raw.githubusercontent.com/the-lupaxa-project/brand-assets/master/logos/organisations/docker-toolbox/readme-logo.png" alt="The Lupaxa Project: Docker Toolbox" />
  </a>
</p>

<h1 align="center">Container Framework</h1>

Container Framework generates a family of images from one package list, one install fragment, and one Dockerfile template. Each supported operating system version is a directory under `Dockerfiles`.

Generating that directory asks [Version Helper](https://github.com/lupaxa-docker-toolbox/version-helper) for pinned install lines, then writes the Dockerfile.

The supported set is the same window Version Helper uses: releases still in the vendor's own security support.

## Requirements

Docker and Python 3. Generate with an empty package list does not start a container. A package list does: Version Helper runs inside the base image, so Docker has to be able to pull it.

## How it Works

From a version directory, `manage.sh` reads the operating system and tag from the path and calls `Scripts/core.sh`.

`core.sh` loads `Config/config.cfg`, the labels, `Templates/install.tpl`, and the cleanup fragment linked from that version. It substitutes those into `Templates/Dockerfile.tpl`.

When `Config/packages.cfg` lists packages, generate downloads `get-versions` and `version-grabber` from Version Helper into `Helpers` and inserts the fragment they print. `Helpers` is a cache.

An empty package list skips the download and leaves the install fragment as the first command in `RUN`.

Build runs `docker build`. The image creation time is the build arg `IMAGE_CREATED`. Publish pushes the version tag. The newest version of a family also pushes `latest`. Debian and Ubuntu also push the codename.

## Supported Operating Systems

| Operating System | Tags                                               | Image                                                  |
| ---------------- | -------------------------------------------------- | ------------------------------------------------------ |
| Alma Linux       | 8, 8-minimal, 9, 9-minimal, 10, 10-minimal         | [Official Image](https://hub.docker.com/_/almalinux)   |
| Alpine           | 3.22, 3.23, 3.24                                   | [Official Image](https://hub.docker.com/_/alpine)      |
| Amazon Linux     | 2023                                               | [Official Image](https://hub.docker.com/_/amazonlinux) |
| Arch Linux       | latest                                             | [Official Image](https://hub.docker.com/_/archlinux)   |
| CentOS Stream    | stream9, stream10                                  | [Quay Image](https://quay.io/repository/centos/centos) |
| Debian           | 12, 12-slim, 13, 13-slim                           | [Official Image](https://hub.docker.com/_/debian)      |
| Oracle Linux     | 8, 8-slim, 9, 9-slim, 10, 10-slim                  | [Official Image](https://hub.docker.com/_/oraclelinux) |
| Photon           | 4.0, 5.0                                           | [Official Image](https://hub.docker.com/_/photon)      |
| Rocky Linux      | 8, 8-minimal, 9, 9-minimal, 10, 10-minimal         | [Official Image](https://hub.docker.com/_/rockylinux)  |
| Ubuntu           | 22.04, 24.04, 26.04                                | [Official Image](https://hub.docker.com/_/ubuntu)      |

> **Note:** Alpine uses `ash`. Slim and minimal images use `sh`. Every other image uses `bash`.
>
> Arch Linux publishes `linux/amd64`. Generate and build pass that platform.
>
> CentOS Stream is `quay.io/centos/centos`. The directory name stays `centos`.

Codename tags, pushed in addition to the version:

| Version        | Extra tag     |
| -------------- | ------------- |
| Debian 12      | bookworm      |
| Debian 12 slim | bookworm-slim |
| Debian 13      | trixie        |
| Debian 13 slim | trixie-slim   |
| Ubuntu 22.04   | jammy         |
| Ubuntu 24.04   | noble         |
| Ubuntu 26.04   | resolute      |

`latest` is Alma 10, Alpine 3.24, Amazon Linux 2023, Arch Linux, CentOS Stream 10, Debian 13, Oracle Linux 10, Photon 5.0, Rocky Linux 10, and Ubuntu 26.04.

`Config/versions.cfg` is that catalogue. `Config/os.cfg` lists the families this repository builds, one name per line.

`./manage-all.sh --sync` keeps those rows and removes the others. A repository with `SINGLE_OS=true` lists one family, and that name is `SINGLE_OS_NAME`. Image repositories download the catalogue from this repository.

## Configuration

`Config/config.cfg` sets the image name prefix and the registries. GHCR publishes under `lupaxa-docker-toolbox`. Docker Hub publishes under `lupaxa`.

`SINGLE_OS=true` is the one-family layout. The directories under `Dockerfiles` are version tags only, and `SINGLE_OS_NAME` is the image name. `NO_OS_NAME_IN_CONTAINER=true` drops the operating system from the image name.

`IMAGE_SHELL` sets the entrypoint when the finished image starts a different shell from the base image. Leave it empty to follow the base image.

Version Helper always uses the base shell, so Alpine is queried with `ash` even when `IMAGE_SHELL` is `bash`.

`Config/packages.cfg` uses the Version Helper lists. `DNF_PACKAGES` falls back to `YUM_PACKAGES` when it is unset. `APK_VIRTUAL_PACKAGES` is the unversioned Alpine list.

```text
APK_PACKAGES=
APK_VIRTUAL_PACKAGES=
APT_PACKAGES=
PACMAN_PACKAGES=
TDNF_PACKAGES=
DNF_PACKAGES=
DNF_GROUPS=
YUM_PACKAGES=
YUM_GROUPS=
```

`Config/labels.cfg` is copied into the Dockerfile, including `org.opencontainers.image.created`. Build sets that value from `IMAGE_CREATED`. Source and documentation are added from the template.

`Templates/install.tpl` is the command prefix shared by every image. `Templates/Dockerfile.tpl` is the file generate fills in. The tokens are:

```text
@@CONTAINER_IMAGE_NAME@@
@@CONTAINER_OS_VERSION_ALT@@
@@LABELS@@
@@PACKAGES@@
@@INSTALL@@
@@CLEANUP@@
@@ENTRYPOINT@@
@@GIT_URL@@
```

Each version links `Templates/cleanup.tpl` at the cleanup fragment for its package manager: apk, apt, dnf, microdnf, pacman, or tdnf.

`yum-cleanup.tpl` and `yum-cleanup-with-leaves.tpl` are the fragments for a yum-only image you add yourself.

The supported set uses dnf or microdnf.

## Usage

From the repository root, a family directory, or a version directory:

```shell
./manage-all.sh --sync
./manage-all.sh --generate
./manage-all.sh --build
./manage-all.sh --generate --build --clean
./manage-all.sh --publish --ghcr
```

`--sync` rewrites the version directories from `Config/versions.cfg` for the families in `Config/os.cfg`. Cleanup links point at the fragment in this repository.

A version directory has `manage.sh` instead of `manage-all.sh`. The flags are the same. That script already adds `latest` and the codename tag where this tree uses them.

```shell
./manage.sh --generate --build
```

```text
-h, --help       Show the help
-d, --debug      Trace the script
-g, --generate   Write the Dockerfile
-b, --build      Build the image
-c, --clean      Build without the cache
-p, --publish    Push the image
-G, --ghcr       Push to GHCR instead of Docker Hub
-l, --latest     Also tag latest
-t, --tags       Extra publish tag, repeatable
```

`--publish` pushes to Docker Hub. `--ghcr` pushes to GHCR. The machine has to be logged in to that registry. Generate, build, and publish can be combined.

A stable tag `vX.Y.Z` runs Generate Release. That workflow builds every image and pushes it to Docker Hub and GHCR. Docker Hub uses `DOCKERHUB_USERNAME` and `DOCKERHUB_TOKEN`. GHCR uses the workflow token.

A release candidate `vX.Y.Z-rcN` runs Generate Test Release. That workflow builds every image and does not push it.

## Image Names

The local name is `<prefix>-<os>-<version>` with dots written as dashes, for example `container-framework-debian-13`.

The GHCR name is `ghcr.io/lupaxa-docker-toolbox/<prefix>-<os>:<version>`, for example `ghcr.io/lupaxa-docker-toolbox/container-framework-debian:13`. Docker Hub uses `lupaxa/<prefix>-<os>:<version>`.

## Caveats

Images that share a package manager still disagree about which packages are already installed. A list that is the union of those images is the practical fix. An already installed package does not change the image.

Slim and minimal images have no bash. The entrypoint is `/bin/sh` on those images and `/bin/ash` on Alpine. Change `Templates/Dockerfile.tpl` when the image should start bash, and install bash from the package list.

<a href="https://github.com/the-lupaxa-project">
    <img src="https://raw.githubusercontent.com/the-lupaxa-project/brand-assets/master/logos/components/footer-for-child-orgs.svg" alt="The Lupaxa Project Footer" width="100%" />
</a>
