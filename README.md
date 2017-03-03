# atomic-system-containers-quickstart
A quick start "guide" thrown together to test out system containers with atomic CLI

Updated as of Mar. 3, 2017

## Overview

System Containers were conceived to containerize services Docker needs to run (etcd, flannel) and even Docker itself. Essentially they are read-only, OS-specific runc containers managed by the atomic CLI and systemd, with OSTree as backend storage and uses skopeo to pull from registries. It is available for atomic CLI version 1.12 and later, but this guide will focus on the upstream version.

There are many features you can make use that will be highlighted in this guide, but basically, in one line, system containers can be run with: `atomic install --system $image`, which will, as Giuseppe (gscrivan@redhat.com) nicely summarizes:

> shortly, these are the most important operations done by atomic install --system $IMAGE:
> - a checkout of the image from the OSTree repository to /var/lib/containers/atomic/$NAME
> - use /exports/{config.json, config.json.template} from the checked-out image rootfs to generate the OCI configuration.
> - use /exports/service.template from the checked-out image rootfs to generate the OCI configuration.
> - use systemctl to start the service

> --system expects one of /exports/{config.json,config.json.template} for the OCI configuration, /exports/service.template and (used at the moment only to set default values in the template files) /exports/manifest.json

> If any of these files is missing, then a default one is generated (for the runc configuration, runc spec is used, while the systemd default configuration file is hard coded in atomic)

For more info, the official blog post is located here: http://www.projectatomic.io/blog/2016/09/intro-to-system-containers/

The very first version of a system container can be found here: http://scrivano.org/static/system-containers-demo

## Requirements and Versioning

Currently, system containers is supported by atomic CLI 1.12 or later, and is tested on feodra/centos/rhel (and atomic variants)

- Upstream atomic repo is [HERE](https://github.com/projectatomic/atomic)
- The container images can be found here: https://github.com/projectatomic/atomic-system-containers
- runc must be installed. runc 0.0.9 and specification 0.4.0 is minimum requirement
- ostree must be installed and an ostree repo must be set up to store images (if no repo is specified, a default one will be created during installation)


### Building atomic from upstream repo (may be needed for various functionalities in newer versions)

- The requirements for atomic can be found [HERE](http://pkgs.fedoraproject.org/cgit/rpms/atomic.git/tree/atomic.spec)

Manually, I tend to do the following:

`$ sudo dnf install -y make git python libffi-devel python-devel libselinux-python ostree-devel python-gobject-base pylint golang-github-cpuguy83-go-md2man redhat-rpm-config gcc PyYAML python-dbus python-docker-py rpm-python docker skopeo python-slip-dbus gcc-go python3-pylint python3-dbus python3-slip-dbus python3-docker-py python3-gobject-base python3-dateutil python2-dateutil python2-coverage`

`$ pip install python-dateutil xattr`

`$ sudo systemctl start docker`

`$ git clone https://github.com/projectatomic/atomic.git`

`$ cd atomic`

`$ sudo make`

`$ sudo make install`

## System Container Examples:

### Hello-world Container

This is a test container that uses ncat to greet you when you `curl localhost:$PORT` ($PORT defaults to 8081, You can change that with --set), it will respond with a "Hi world". You can install it directly with: `atomic install --system --set=PORT=$PORT --set=RECEIVER=$NAME gscrivano/hello-world`, where the --set flags and values are optional.

You can now start the systemd service with `systemctl start hello-world`. You can see the container of the status with `atomic containers list` and `systemctl status hello-world`. If the container is running fine, `curl localhost:$PORT` should respond with a "Hi world".

One can also play around with parameters, such as `--set=RECEIVER=Jerry`, and it will output "Hi Jerry" instead.

`atomic uninstall hello-world` stops and removes the container.

### Etcd container

You can directly install a pre-built image from the repo here: `atomic install --system gscrivano/etcd`

This will pull the pre-built etcd image from [docker hub](https://hub.docker.com/r/gscrivano/etcd/) and install the system container.

Alternatively, this container image also lives within the fedora registry. You can pull from: registry.fedoraproject.org/f25/etcd

The container is now installed but not running. If you use `atomic containers list -a` you will see that it is created but inactive. You can start the service with `systemctl start etcd`. or `atomic run etcd` (which for a system container just re-routes the command to systemctl start)

You can check the status of the container with `atomic containers list`, or `systemctl status etcd`

To stop and remove the container, you can directly use `atomic uninstall etcd`. Don't do yet this if you want to test out flannel as well.


### Flannel container

The etcd container (or an etcd service) must be running. If you are running the above Etcd container, you can set the network config as such: `runc exec etcd etcdctl set /atomic.io/network/config '{"Network":"172.17.0.0/16"}'`

Again from the repo, one can: `atomic install --system gscrivano/flannel`.

Alternatively, this image lives within the fedora registry here: registry.fedoraproject.org/f25/flannel

As of atomic 1.12 you have to start the service manually with `systemctl start flannel`.

You can check the status of the container with `atomic containers list`, or `systemctl status flannel`, or `ifconfig | grep flannel`. In case of failure refer to logs or troubleshooting below.

If you had Docker running at this point, the install will invoke `systemctl daemon-reload` to use the new flannel configurations from the container. Docker will be stopped and can be restarted with `systemctl start docker` to use the new configuration file.

Similarily, `atomic uninstall flannel` cleans it up.

Note that presently, if you restart your machine, the config file for flannel (inside Docker) will be re-created. Although the config has not changed, post-restart it will prompt you to `systemctl daemon-reload` again (since it thinks the config file was changed). That can be ignored.

Note for flannel with specified $NAME for the container, /run/$NAME will be created and bound to /run/$NAME in the container. The folder will include things such as the subnet.env file.



Note 1: for the above 3 containers, one can add `--name` as a flag to specify what the container name will be. If you do not, there are default names that will be assigned (the image name). Currently we don't actually have IDs associated with containers, so the ID is just the name, and this will be used in further actions.

Note 2: atomic run/stop can be used to start and stop containers as well. For system containers they are basically wrappers for systemctl start/stop.


### Other commands with atomic (update, rollback, --rootfs, list)

#### Update

By default, the conatiners are checked out at /var/lib/containers/atomic/. The first time you create a container, a $CONTAINER.0 is created, and a $CONTAINER symlink will point to that location. One can update a container with `atomic containers update $CONTAINER`, which will detect newer images for the container, check them out into $CONTAINER.1, point the $CONTAINER symlink to the new location, and restart the service. Note that only 2 versions are saved, so the next time you update, $CONTAINER.0 is overwritten and used as the new location.

An exmaple usage of update: `atomic containers update --set=RECEIVER=foo --container hello-world` will cause `curl localhost:8081` to respond with `Hi foo`. Updating also applies any changes to the image the container is using stored in ostree. If packages/scripts/etc. have changed in the image, it will be there after updating.


#### rollback

You can roll back a system container to the previous deployment. The systemd service file and any tmpfiles from the older deployment are re-installed. Example usage would be `atomic containers rollback hello-world`, which will cause the RECEVIER environment variable to return to what it was before the update. Note that since only 2 deployments of a system container are saved, if you invoke the above command again, it will change to the new deployment (much like atomic host rollbacks) and once again RECEIVER will be "foo".

#### A container with a remote rootfs

Let's say you had the above helloworld running, and you want to duplicate that container 5 times. Instead of checking out 5 images into exploded containers, you could use --rootfs to specify an existing rootfs folder as a read-only rootfs to run the service. For example, you can `atomic install --system --name=hello-world-remote --rootfs=/var/lib/containers/atomic/hello-world --set=RECEIVER=remote-user --set=PORT=8083 gscrivano/hello-world`, and the container will run as normal, i.e. curl'ing port 8083 will return you "Hi remote-user". But if you take a look at /var/lib/containers/atomic/hello-world-remote, you'll notice there isn't a rootfs folder, just config and info.

The new container is using the rootfs located in /var/lib/containers/atomic/hello-world, and if that gets updated, the remotes will automatically get updated as well. Conversely, no updates to images can be directly applied to the remote containers. This reduces necessary space to run multiple conatiners, if they can use the same image.

Note that by default, all atomic system containers MUSt have a read-only rootfs.

#### Checking other system container stats with list

By default, atomic containers list shows a truncated version of all running containers (including docker) on the system. If you just want to see system container images, you can `atomic containers list -a --no-trunc -f runtime=ostree` to filter for the system containers.

## System Container Images

### pulling

System containers use images stored in ostree. If an image doesn't exist during install locally, the atomic CLI will attempt to pull said image from a registry (default docker hub). Most of the time it is better to have the image pulled to ostree before installation, and that can be done as follows:

`atomic pull --storage ostree $IMAGE` will pull the image directly to ostree. $IMAGE would be something like registry/repo/image or just repo/image if it is from the docker hub.

`atomic pull --storage ostree docker:$IMAGE` will pull the image from local docker instead.

`atomic pull --storage ostree dockertar:$IMAGE` will pull the image from a dockertar.

### list and info

The images from above can be viewed with `atomic images list`. You'll notice that for images in use by running containers, the corresponding atomic image has a ">" next to it. To filter for just ostree images, you can filter with `atomic images list -f type=ostree`

`atomic version --storage ostree $IMAGE` will give you some versioning info.

`atomic info --storage ostree $IMAGE` will show you the labels of the image, but more importantly will display environment variables for that container image. Environment variables can be: runc process environment, systemd service working/runtime directories, runscript variables, etc. Most of these will have a default value as specified by a manifest.json included with the file, or will be set by the OS during installation. Variables with no default value and not set by OS will be displayed as a separate category, and if not specified, will cause installation to fail. These variables can be set with `--set` flag during installation.

### building your own image

The above containers can be found at https://github.com/projectatomic/atomic-system-containers.

Basically, an image will consist of some combination of the following:
 - `config.json.template`: mandatory runc config file, used to run the container
 - `service.template`: mandatory systemd service file, to define how the service will be run
 - `manifest.json`: optional(?) define default values for environment variables
 - `tmpfiles.template`: optional, used to remove tmpfiles created on the host by the container
 - `init.sh`: optional(?) define the starting point of the container (setup, run)
 - `Dockerfile`: mandatory, define packages, and include the above files to /exports in the rootfs

Once you are satisfied with the config files, you can locally build (for example, with etcd)

`docker build -t etcd .`

`atomic pull --storage=ostree docker:etcd`

`atomic install --system etcd`

Note that you could also set up a docker hub repo and push to that (name the image DOCKER_REPO_NAME/etcd for example). If you don't have a local pull, skopeo will try to pull docker hub by default, and if there is no docker repo associated with the image you want, the install will fail.

To remove a local atomic image, you can invoke `atomic images delete` and then `atomic images prune`

## Troubleshooting (some errors I've ran into)

**My container failed to install at a weird point, how do I start clean?**

The following locations container information about the containers:

`/var/lib/containers/atomic` - this is where the containeres are checked out to

`/run/runc` - this is where the state of the containers are being stored

`/etc/systemd/system/multi-user.target.wants/`, 
`/etc/systemd/system` - this is where the service is created

you can remove these by hand and try the installation again. Alternatively `atomic --debug install ...` helps find where the error happens.

**My container does not restart, and journal shows that the container exists even though I have already `atomic uninstall`ed**

Sometimes this happens, and you need to manually delete the container with runc as well (e.g. `runc delete etcd`). Then uninstall/reinstall.

**I get an error like: [Errno -2] Name or service not known**

Most likely this is a skopeo error, where it is trying to pull from docker hub and failing. View above on building images.

**When I pull an image, I get "unable to ping registry endpoint" "certificate signed by unknown authority"**

Add that registry in /etc/sysconfig/docker to INSECURE_REGISTRY (INSECURE_REGISTRY='--insecure-registry REGISTRY_LINK')
