# Vivarium

Build OpenWrt packages using Docker and the OpenWrt SDK

## Introduction

Vivarium is a way to compile packages for [OpenWrt] without having to
install the necessary [build tools][OpenWrt build system install], using
[Docker].

Vivarium includes a "builder" Dockerfile for a local Docker image, based
on the CircleCI Docker image used by the [OpenWrt packages feed]. This
local image will have the [OpenWrt SDK] set up inside.

This also includes a Docker Compose file that holds all of the
configuration options. The Compose file also sets up a number of bind
mounts, allowing access to package source code from the Docker container
and access to build artifacts from the host machine.

[OpenWrt]: https://openwrt.org/
[OpenWrt build system install]: https://openwrt.org/docs/guide-developer/build-system/install-buildsystem
[Docker]: https://www.docker.com/
[OpenWrt packages feed]: https://github.com/openwrt/packages
[OpenWrt SDK]: https://openwrt.org/docs/guide-developer/using_the_sdk

## Requirements

[Docker][Docker install] and [Docker Compose][Docker Compose install]
will need to be installed.

Some familiarity with [Docker][Docker get started] and the OpenWrt
[build system][OpenWrt build system usage] / [SDK][OpenWrt SDK] will be
necessary.

Vivarium has only been tested with Linux (Ubuntu 19.04, to be exact).
Testing with other platforms is welcome.

[Docker install]: https://docs.docker.com/install/#supported-platforms
[Docker Compose install]: https://docs.docker.com/compose/install/
[Docker get started]: https://docs.docker.com/get-started/
[Openwrt build system usage]: https://openwrt.org/docs/guide-developer/build-system/use-buildsystem

## Getting started

1.  Download the source code ([zip][Vivarium zip] or
    [tar.gz][Vivarium tar.gz]) and extract, e.g.:

        $ unzip openwrt-vivarium-master.zip

    If you will be using Git to manage your package source code, then
    you will want to download Vivarium without Git to avoid nesting Git
    repositories.

2.  In the project directory, create a `packages` directory and add
    source code for packages, e.g. by checking out the OpenWrt packages
    feed:

        $ cd openwrt-vivarium-master
        $ git clone https://github.com/openwrt/packages.git

3.  If you are using Linux and your user ID is not 1000, you will need
    to change the ownership of the `packages` and `sdk` directories (and
    all files and subdirectories inside) to UID 1000, e.g.:

        $ sudo chown -R 1000:1000 packages sdk

    These directories need to be readable and writable by the normal
    user inside the builder container (with UID 1000 / GID 1000).

    To check your user ID, you can use the `id` command:

        $ id -u

4.  (Optional) Build the local Docker image:

        $ sudo docker-compose build

    This is optional because Docker Compose will automatically build
    the image when necessary.

5.  Build packages by using `docker-compose run`, e.g.:

        $ sudo docker-compose run --rm builder make package/python/compile V=s

    If the build was successful, the compiled package(s) will be in the
    `sdk/bin` directory.

If you are using an older version of Docker (<1.13.0) or Docker Compose
(<1.10.0), you will need to change `version` in `docker-compose.yml`
from `"3"` to `"2"` after Step 2. Vivarium has not been tested with
these older versions though; upgrading to the latest versions of Docker
and Docker Compose is recommended.

During each builder run, these SDK commands:

    ./scripts/feeds update -a
    ./scripts/feeds install -a
    make defconfig

will be run before the command specified on the `docker-compose run`
command line.

[Vivarium zip]: https://github.com/jefferyto/openwrt-vivarium/archive/master.zip
[Vivarium tar.gz]: https://github.com/jefferyto/openwrt-vivarium/archive/master.tar.gz

## Directory structure

*   `builder`: Files that define the local "builder" Docker image.

*   `packages`: Package source code. (This directory must be created
    locally.)

    This directory will be added either as the "packages" feed or as a
    custom feed, depending on the `CUSTOM_FEED` [image build
    option](#image-build-options).

*   `sdk`: Subdirectories in here are bind mounted into various places
    within the SDK inside the builder container, to cache results and
    allow build artifacts to be inspected from the host machine.

    *   `sdk/bin`: Compiled package files (*.ipk).

    *   `sdk/build_dir`: Where program source code is extracted and
        compiled.

    *   `sdk/dl`: Archives downloaded during package build.

    *   `sdk/feeds`: Where package feeds are checked out / cloned.

    *   `sdk/logs`: Build logs (if enabled) and feed error logs. During
        each builder run, the generated config file (`.config`) is also
        copied into here as `config`.

    *   `sdk/package/feeds`: Symbolic links to packages in `sdk/feeds`.

    *   `sdk/staging_dir/hostpkg`: Where host packages are installed.

    *   `sdk/staging_dir/target`: Headers and other supporting files
        installed by target library packages, for use when compiling
        other target packages.

    *   `sdk/tmp`: Temporary files.

    There are also two "special" subdirectories:

    *   `sdk/overrides`: Files placed here will be copied into the SDK
        directory in the builder container, allowing files to be
        overriden directly.

        Specifically, if there is a file named `diffconfig` in this
        directory, it will be copied to `.config` inside the builder
        container, which will then be expanded by `make defconfig`.

    *   `sdk/sdk`: SDK archives can be saved here to speed up the local
        Docker image build.

        If there is an SDK archive in this directory that matches the
        `SDK_FILE` [image build option](#image-build-options), then it
        will be used instead of downloading the archive from `SDK_HOST`.

        If the `KEEP_SDK_FILE` [run-time option](#run-time-options) is
        enabled, then when the builder is run, the SDK archive saved
        inside the Docker image will be copied into this directory.

## Configuration

All options can be found in the `docker-compose.yml` file.

### Image build options

*   `SDK_HOST`, `SDK_PATH`, `SDK_FILE`: Which SDK to use.

    If there is an SDK archive (matching `SDK_FILE`) in the `sdk/sdk`
    directory, then it will be used instead of downloading the archive
    from `SDK_HOST`.

*   `BRANCH`: Which branch to use when setting up package feeds, e.g.
    `master`, `openwrt-18.06`, etc.

*   `CUSTOM_FEED`: Controls how the `packages` directory is used:
    *   `n`: The directory is added as the "packages" feed, in
        place of the [OpenWrt packages feed].
    *   `y`: The directory is added as a [custom
        feed][OpenWrt custom feeds] (the OpenWrt packages feed will be
        present as well).

[OpenWrt custom feeds]: https://openwrt.org/docs/guide-developer/feeds#custom_feeds

### Run-time options

*   `KEEP_SDK_FILE`: If `y`, the SDK archive within the Docker image will
    be copied to the `sdk/sdk` directory when the builder is run.

*   `CONFIG_AUTOREMOVE`, `CONFIG_BUILD_LOG`: Sets the corresponding SDK
    config options (`y` or `n`).

## Rebuild local Docker image

The local Docker image can be rebuilt to change SDKs or update to the
latest snapshot SDK.

If `KEEP_SDK_FILE` is `y`, it may be necessary clear the `sdk/sdk`
directory first to ensure a new SDK archive is downloaded:

    $ rm -f sdk/sdk/*

Then rebuild the image:

    $ sudo docker-compose build

The `--no-cache` option may be needed to force a rebuild / re-download.

## Directory cleaning

The SDK commands for directory cleaning (`make clean`, `make dirclean`,
etc.) are not aware of the Docker bind mounts and so may not work
correctly.

There are scripts in the `sdk` directory (`clean.sh`, `dirclean.sh`,
`distclean.sh`) that emulate the SDK commands and can be run from the
host.

## License

Copyright (C) 2019 Jeffery To  
https://github.com/jefferyto/openwrt-vivarium

Vivarium is free software: you can redistribute it and/or modify
it under the terms of the GNU General Public License version 2 as
published by the Free Software Foundation.

Vivarium is distributed in the hope that it will be useful,
but WITHOUT ANY WARRANTY; without even the implied warranty of
MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
GNU General Public License for more details.

You should have received a copy of the GNU General Public License
along with Vivarium.  If not, see <https://www.gnu.org/licenses/>.

Contains code from .circleci/config.yml of the OpenWrt packages feed  
Copyright (C) 2018 Etienne Champetier, Ted Hess
