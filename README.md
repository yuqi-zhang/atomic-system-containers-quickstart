# atomic-system-containers-quickstart
A quick start "guide" thrown together to test out system containers with atomic, mostly for personal reference

## Overview

As Giuseppe (gscrivan@redhat.com) nicely summarizes:

> shortly, these are the most important operations done by atomic install --system --name=$NAME $IMAGE:
> - a checkout of the image from the OSTree repository to /var/lib/containers/atomic/$NAME
> - use /exports/{config.json, config.json.template} from the checked-out image rootfs to generate the OCI configuration.
> - use /exports/service.template from the checked-out image rootfs to generate the OCI configuration.
> - use systemctl to start the service

> --system expects one of /exports/{config.json,config.json.template} for the OCI configuration, /exports/service.template and (used at the moment only to set default values in the template files) /exports/manifest.json

> If any of these files is missing, then a default one is generated (for the runc configuration, runc spec is used, while the systemd default configuration file is hard coded in atomic)

For more information, check out: http://scrivano.org/static/system-containers-demo

## Requirements and Versioning

On fedora 24/Centos CI:
- Runc 1.0 release is **optional**, and can be found [HERE](https://github.com/opencontainers/runc). There are changes compared to previous versions which require different usage. (runc 0.0.9 and specification 0.4.0 is minimum requirement)
- Upstream atomic repo is [HERE](https://github.com/projectatomic/atomic)
- An ostree repo must be set up to store images (if no repo is specified, a default one will be created during installation)

Runc 1.0 and earlier versions are both supported in the current upstream atomic (Aug. 19). Versioning differences will be highlighted in this document.

### Building atomic from upstream repo (may be needed for various functionalities in newer versions)

- The requirements for atomic can be found [HERE](http://pkgs.fedoraproject.org/cgit/rpms/atomic.git/tree/atomic.spec)

For a fresh fedora 24 image:

`$ sudo dnf install -y make git python libffi-devel python-devel libselinux-python ostree-devel python-gobject-base pylint golang-github-cpuguy83-go-md2man redhat-rpm-config gcc PyYAML python-dbus python-docker-py rpm-python docker skopeo python-slip-dbus gcc-go python3-pylint python3-dbus python3-slip-dbus python3-docker-py python3-gobject-base python3-dateutil python2-dateutil python2-coverage`

`$ pip install python-dateutil xattr`

`$ sudo systemctl start docker`

`$ git clone https://github.com/projectatomic/atomic.git`

`$ cd atomic`

`$ sudo make`

`$ sudo make install`

## System Container Examples:

### Etcd container

You can directly pull a pre-built image from the repo here: `atomic install --system --name=etcd gscrivano/etcd`

This will pull the pre-built etcd image from [docker hub](https://hub.docker.com/r/gscrivano/etcd/) and install the system container.

You can check the status of the container with `atomic ps`(as of v1.12 this has been refractored to `atomic containers list`), or `systemctl status etcd`

To stop and remove the container, you can directly use `atomic uninstall etcd`. Don't do yet this if you want to test out flannel as well.


### Flannel container

The etcd container (or an etcd service) must be running. If you are running the above Etcd container, you can set the network config as such: `runc exec etcd etcdctl set /atomic.io/network/config '{"Network":"172.17.0.0/16"}'`

Again from the repo, one can: `atomic install --system --name=flannel gscrivano/flannel`.

You can check the status of the container with `atomic ps` (as of v1.12 this has been refractored to `atomic containers list`), or `systemctl status flannel`, or `ifconfig | grep flannel`.

Similarily, `atomic uninstall flannel` cleans it up.

Note for flannel with specified $NAME for the container, /run/$NAME will be created and bound to /run/$NAME in the container. The folder will include things such as the subnet.env file.


### Helloworld Container

This is a test container that, when you `curl localhost:$PORT` ($PORT defaults to 8081, You can change that with --set), it will respond with a "Hi world". You can build it directly with: `atomic install --system --name=hello-world --set=PORT=$PORT --set=RECEIVER=$NAME gscrivano/hello-world`

One can also play around with parameters, such as `--set=RECEIVER=Jerry`, and it will output "Hi Jerry" instead.

Again, `atomic uninstall helloworld` stops and removes the container.


Note that for the above 3 containers, the "name" field is optional. But currently we don't actually have IDs associated with containers, so the ID is just the name, and the default names would look something like gscrivano-flannel. Giving a --name parameter tells what service should be started with the container.


### Other commands with atomic (update, install --rootfs, ps)

#### Update

By default, the conatiners are checked out at /var/lib/containers/atomic/. The first time you create a container, a $CONTAINER.0 is created, and a $CONTAINER symlink will point to that location. One can update a container with `atomic update`, which will detect newer images for the container, check them out into $CONTAINER.1, point the $CONTAINER symlink to the new location, and restart the service. Note that only 2 versions are saved, so the next time you update, $CONTAINER.0 is overwritten and used as the new location.

An exmaple usage of update: `atomic update --set=RECEIVER=foo --container hello-world` will cause `curl localhost:8081` to respond with `Hi foo`.

#### A container with a remote rootfs

Let's say you had the above helloworld running, and you want to duplicate that container 5 times. Instead of checking out 5 images into exploded containers, you could use --rootfs to specify an existing roofts file as a read-only rootfs to run the service. For example, you can `atomic install --system --name=hello-world-remote --rootfs=/var/lib/containers/atomic/hello-world --set=RECEIVER=remote-user --set=PORT=8083 gscrivano/hello-world`, and the container will run as normal, i.e. curl'ing port 8083 will return you "Hi remote-user". But if you take a look at /var/lib/containers/atomic/hello-world-remote, you'll notice there isn't a rootfs folder, just config and info.

The new container is using the rootfs located in /var/lib/containers/atomic/hello-world, and if that gets updated, the remotes will automatically get updated as well. Conversely, no updates to images can be directly applied to the remote containers. This reduces necessary space to run multiple conatiners, if they can use the same image.

Note that by default, all atomic system containers MUSt have a read-only rootfs.

#### Checking other system container stats with "atomic ps" (as of v1.12 this has been refractored to `atomic containers list`)

By default, atomic ps shows a truncated version of all running containers (including docker) on the system. If you just want to see system container images, you can `atomic ps -a --no-trunc -f runtime=runc` to filter for the system containers.

## Building an Image

The images from above can be viewed with `atomic images list`. You'll notice that for images in use by running containers, the corresponding atomic image has a ">" next to it.

The above containers can be found at https://github.com/giuseppe/atomic-oci-containers. Once you are satisfied with the config files, you can locally build (for example, with etcd)

`docker build -t etcd .`

`atomic pull docker:etcd  ## the docker: prefix means to pull from the local Docker`

`atomic install --system --name=etcd etcd`

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

