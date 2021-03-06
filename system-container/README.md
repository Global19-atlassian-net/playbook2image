# System Container support

A system container is run with runC instead of Docker and it is managed using
the [atomic](https://github.com/projectatomic/atomic/) tool.  As it doesn't
require the Docker daemon to run, a container that makes changes to the Docker
daemon's configuration/status can be run on a sytem that is itself a target/part
of the inventory.

Therefore, if the playbook(s) that are packaged into the image manipulate the
container runtime configuration/status (potentially stopping running containers)
and the system where the container will run is part of the inventory (i.e. you
want the playbooks to alter configuration/status of the container engine in the
same system where the playbook is being run inside a container), then you might
want to run the container/playbook as a system container to prevent the changes
performed by the playbook from killing the container that is running the
playbook itself.

The files under [`exports/`](exports/) are needed to run the the built images as
an [Atomic system container](http://www.projectatomic.io/blog/2016/09/intro-to-system-containers/):

* `config.json.template` is the configuration file to run the runC container.

* `manifest.json` defines various settings for the system container, such as the
  default values to use for the installation.

* `service.template` is the template for the systemd service.

* `tmpfiles.template` is the template for systemd-tmpfiles.

The base playbook2image container image already contains all the required files
to run as a system container, so no specific action needs to be taken to build a
system container-ready image based on playbook2image.

Once a container image is built to package your playbooks you can import it into
the OSTree storage with:

    atomic pull --storage ostree docker:my/image:latest

where `my/image:latest` is the name and tag of the image you built. See the main
[README](../README.md) for more details about building a container image using
playbook2image.

## Running an image as a System Container

To run the playbook `myplaybook.yml` from `myimage` as a system container:

```sh
atomic install --system \
	   --set INVENTORY_FILE=$(pwd)/myinventory \
	   --set PLAYBOOK_FILE=myplaybook.yml \
	   myimage
```

This command actually will perform 3 steps: install, interactively run, and
finally uninstall the image as a system container. This is thanks to the
`atomic.run="once"` label in the image.

### Older versions of atomic

*NOTE:* the `atomic.run="once"` functionality was added to the `atomic` tool in
 [PR 952](https://github.com/projectatomic/atomic/pull/952). If you are using an
 older version of the `atomic` command that does not include that functionality,
 the `atomic install` command above will only install the image but not run it
 or clean up automatically.

On older versions of `atomic`, after running the `atomic install ...` command
above you will have to manually request systemd to run a container with it:

    systemctl start myimage

and finally clean up when it's done:

    atomic uninstall myimage

Note that with that approach the container is run by systemd and you will not
get feedback from Ansible during the playbook's execution: the `systemctl start`
command above will trigger the container/playbook execution, but the `systemctl`
command itself will not show the output of Ansible while the playbook runs. You
will have to search for that output on the logs generated by the systemd unit
e.g. using `journalctl`.

## Environment variables

* See the [main README](../README.md) for details about
  `INVENTORY_FILE`,`PLAYBOOK_FILE` and `OPTS`

* `VAR_LIB_PLAYBOOK2IMAGE` (optional) If the playbooks need additional files
  then they can use the path `/var/lib/playbook2image` in the container. This is
  bind mounted from the host. The source directory in the host can be specified
  via the `VAR_LIB_PLAYBOOK2IMAGE` environment variable, which defaults to
  `/var/lib/playbook2image` (i.e. the same path in the host and in the
  container).

* `VAR_LOG_ANSIBLE_LOG` (default: `/var/log/ansible.log`) points to the log file
  where playbook execution will be logged.

* `SSH_ROOT` (default: `/root/.ssh`) points to the ssh client configuration
  (including ssh keys).

## Known limitations

Initially, system containers can only use a single static `INVENTORY_FILE`.
