# Sailfish OS build environment based on containers

This is an alternative build environment for Sailfish OS. In contrast
to the official SDKs, it doesn't use
[Scratchbox2](https://github.com/sailfishos/scratchbox2)
(SB2). Instead, it relies on running either native or through QEMU
hardware emulation. When using QEMU emulation, expect slow
compilation speeds when compared to SB2.

This environment can be used either on PC/server or in a cloud using container
instances. If needed, in addition to work with the local files, it allows you
to pull the sources from git repository before the build and push artifacts to S3
bucket.

This build environment was created to address official SDK limitations
encountered while packaging Qt 5.15 for Sailfish OS. You may want to
use it if you have a project that is difficult or impossible to
compile using the official SDK.

On PC/servers, it is recommended to use the environment with `podman`. While docker may work
as well, it is not tested with it.

## How to use it

Builder images can be generated locally
([see below for instructions](#how-to-create-builder-images)) or pulled from
[ghcr.io](https://github.com/orgs/sailfishos-open/packages?repo_name=docker-sailfishos-builder).

To use, go to the folder of your cloned repository root and run (for `sailfishos-i486-4.6.0.13`)
build command similar to:

```
podman run --rm -it -v `pwd`:/source \
   ghcr.io/sailfishos-open/docker-sailfishos-builder-i486:4.6.0.13 \
   buildrpm \
     -r https://repo.sailfishos.org/obs/sailfishos:/chum:/testing/4.6.0.13_i486/
```

If you have selinux enabled, you may have to use `:z` while mounting source
folder:
```
podman run --rm -it -v `pwd`:/source:z \
   ghcr.io/sailfishos-open/docker-sailfishos-builder-i486:4.6.0.13 \
   buildrpm \
     -r https://repo.sailfishos.org/obs/sailfishos:/chum:/testing/4.6.0.13_i486/
```

In this example, Podman container gets access to the sources by its
volume mapping your current folder (`pwd`) to `/source` inside the
container. Container executes `buildrpm` script inside it (see
[sources](scripts/buildrpm)) that handles building RPMs from SPEC in
`/source/rpm`.

There are few options that can be given to `buildrpm` script:

```
 -b NAME   build name that is used to generate build information
           file. If absent, SPEC filename is used to derive it.

 -s SPEC   builds using given SPEC. Here, SPEC basename is only
	   expected and should be located under rpm/ subfolder
	   of the sources. By default, the first SPEC in rpm/
	   subfolder is used

 -r REPO   additional RPM repository that is needed to fetch packages
	   required for building SPEC. Can be specified multiple times

 -d GIT:TAG download Git repository version described by Git commit ID or tag

 -v VENDOR set vendor for RPM

 -p        skip generation of source package and use the one in /source/rpm
```

If all goes well, RPMs will be created under subfolder `RPMS` of the
sources.

Note that it is recommended to use `--rm` to remove container as soon
as it is finished. On every build, a clean environment is used and all
the dependencies are pulled in again.

### Using with sources in archive

In addition to the mode, where the build is performed using checked
out sources, it is possible to build packages when the sources are
already available in packaged form. For example, Node.js as packaged
at
[OBS](https://build.merproject.org/package/show/sailfishos:chum:testing/nodejs18). In
this case, you have to use option `-p` and mount volumes separately
for `/source/rpm` and `/source/RPMS`. Here, `/source/rpm` should be
linked with the host folder that has RPM SPEC, source archive as
referenced in SPEC, and all patches. Folder `/source/RPMS` has to be
linked with a host folder that will receive compiled RPMS. If you
forget to specify the latter, your compiled RPMS would stay in the
container. It is expected that `/source/rpm` and `/source/RPMS` point
to different folders on host.

Example command :

```
podman run --rm -it \
   -v `pwd`/../nodejs18:/source/rpm \
   -v `pwd`:/source/RPMS \
   ghcr.io/sailfishos-open/docker-sailfishos-builder-i486:4.6.0.13 \
   buildrpm -p -v chum \
       -r https://repo.sailfishos.org/obs/sailfishos:/chum:/testing/4.6.0.13_i486/
```

In this example, Node.js RPMs are built and saved into the current
folder (`pwd`). Corresponding sources are in a folder
`../nodejs18`. It is also setting vendor to `chum` and is using one of
Chum repositories.

### Using with sources that are pulled from Git repository

Instead of working with local sources, it is possible to request the
build environment to pull sources using Git and build them. For that,
specify Git repository and its tag using `url:tag` format and provide it
by giving `-d` option to the buildrpm script in the container. Example:

```
podman run --rm -v `pwd`:/source \
  ghcr.io/sailfishos-open/docker-sailfishos-builder-aarch64:4.6.0.13 \
  buildrpm \
    -b tinybuild \
    -d https://github.com/sailfishos-chum/tinyxml2.git:68b139533c605e4ef9761b1f06c99d80215a1afd
```

### Uploading build artifacts to S3

You can instruct the builder to upload your RPMs into S3 bucket. Internally,
[s3cmd](https://s3tools.org/usage) is used for it. To activate this mode,
specify and forward two environment variables:

- `S3_BUCKET` name of the bucket in S3
- `S3_OPTIONS` command line options passed to s3cmd that should define your
  access to S3. Read s3cmd [manual page](https://s3tools.org/usage) for options

It is advisable to use `-q` option of s3cmd to avoid logging your access
codes. When using on PC, you can set environment variables in a file as shown below

```
S3_BUCKET=build-store
S3_OPTIONS=-q --access_key=MYKEY --secret_key=SECRET --host=super.server.org --host-bucket=%(bucket)s.super.server.org
```

and then import these variables as in

```
podman run --rm \
   --env-file .s3env \
   ghcr.io/sailfishos-open/docker-sailfishos-builder-aarch64:4.6.0.13 \
   buildrpm -b tinybuild \
   -d https://github.com/sailfishos-chum/tinyxml2.git:68b139533c605e4ef9761b1f06c99d80215a1afd
```

## How it works

Inside Podman container, the build is performed in several steps.

First, SPEC file is parsed and all the build requirements are
installed.

Second, if sources are in Git repository and `-p` option was not used,
RPM version will be determined based on the latest git tag and its
offset from HEAD. Regardless of whether sources are in Git or not,
release will be offset by UTC timestamp. Such handling of RPM version
and release will ensure the newest builds would have larger
version-release pair.

Next, RPM build proceeds in a classical way using `rpmbuild`, under a
dedicated user `builder`. For that, folders for building are setup
under `/builder/rpmbuild`. It is possible to debug intermediate steps
through creation of the volume linking to `/builder/rpmbuild` in
the container.

Before starting the build, your sources are packed into tar.gz (or
bz2, xz, as given in `Source0`) under
`/builder/rpmbuild/SOURCES`. This step is skipped if `-p` option was
given. In addition, all files from your sources `rpm` subfolder are
copied to `/builder/rpmbuild/SOURCES`, including all patches.

Then the build proceeds under user `builder` with
`rpmbuild`. `rpmbuild` will unpack the sources, apply patches and
proceed building your packages. After successful build, RPMs are
copied into your source folder under `RPMS` subfolder. For a feedback,
`rpmlint` is applied as well.

Note that, as a new clean environment is created for every build, it
maybe advantageous to debug the build process and reuse the same
container in the case of failure. Easiest is just start the container
with `bash` and execute `buildrpm` already while inside the
container. It is possible to apply the steps done by `buildrpm`
manually as well when inside the container.

## Limitations

The scripts are not handling many cases in a flexible manner provided
by `mb2` from official SDK. For example, during packaging, sources are
packed and then unpacked which wastes time and allocates storage,
until the container is destroyed. There are probably many other
cases where `mb2` would be able to handle building better.

Due to the way the builder works, Source0 from RPM SPEC is expected to
be in the form `%{name}-%{version}.tar.bz2` (.gz and .zx are supported
as well).

While we get rid of SB2 bugs, there are possible QEMU issues that can
interfere. For example, armv7hl could be hit with [issue
#6](https://github.com/sailfishos-open/docker-sailfishos-builder/issues/6).
See issue for problem description. It is recommended then to check
whether QEMU distributed by Jolla works better.

## How to create builder images

Builder container images are very easy to create using `makeimage`
script. Make sure you have QEMU setup working for all architectures
that you want to use (see below for QEMU setup, if needed).

Script `makeimage` takes two arguments: SFOS architecture (i486,
aarch64, armv7hl) and SFOS version. It is also possible to specify
whether to use podman (default) or docker for containers. Example:

```
./makeimage i486 4.6.0.13
```

This will create locally Podman container image `docker-sailfishos-builder-i486:4.6.0.13`. This
image can be used for building your packages.

## QEMU setup

You need to setup binfmt for qemu for the architectures you want to use. Follow the instructions for your system

### Arch-based

Install package `qemu-user-binfmt`.

### Debian-based

Install package `qemu-user-static`.

### Other

(This part is not meant for Debian-based systems and must be skipped to avoid breaking `systemd-binfmt`)

To enable QEMU in most systemd based Linux, add `qemu-custom.conf` in
`/etc/binfmt.d`. Example:

```
:qemu-aarch64:M::\x7fELF\x02\x01\x01\x00\x00\x00\x00\x00\x00\x00\x00\x00\x02\x00\xb7\x00:\xff\xff\xff\xff\xff\xff\xff\x00\xff\xff\xff\xff\xff\xff\xff\xff\xfe\xff\xff\xff:/usr/bin/qemu-aarch64:F
:qemu-arm:M::\x7fELF\x01\x01\x01\x00\x00\x00\x00\x00\x00\x00\x00\x00\x02\x00\x28\x00:\xff\xff\xff\xff\xff\xff\xff\x00\xff\xff\xff\xff\xff\xff\xff\xff\xfe\xff\xff\xff:/usr/bin/qemu-arm:F
```

After that, restart systemd service:

```
systemctl restart systemd-binfmt
```

You should see formats registered in `/proc`:

```
# cat /proc/sys/fs/binfmt_misc/qemu-aarch64
enabled
interpreter /usr/bin/qemu-aarch64
flags: F
offset 0
magic 7f454c460201010000000000000000000200b700
mask ffffffffffffff00fffffffffffffffffeffffff
```

See References below for how to test QEMU support using Docker images.

## References

- SFOS Docker scripts: https://github.com/CODeRUS/docker-sailfishos-baseimage
- Docker architectures: https://github.com/docker-library/official-images#architectures-other-than-amd64
- QEMU setup: https://wiki.gentoo.org/wiki/Embedded_Handbook/General/Compiling_with_qemu_user_chroot
- QEMU tests with Docker: https://github.com/multiarch/qemu-user-static
