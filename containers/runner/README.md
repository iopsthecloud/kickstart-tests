Running kickstart tests in containers
-------------------------------------

Tooling for running tests in VMs in cloud (OpenStack) is in [../../linchpin](../../linchpin).

The motivation for moving from VMs to containers to run kickstart tests is:
* To be able to use containers tooling for deployment, scheduling and scaling of resources for batches of kickstart tests.
* To allow the user to run a kickstart test manually in an easy way.


Host requirements
-----------------

Dependencies needed to be installed are defined in `host_packages` variable of [vars.yml](vars.yml).
```
sudo dnf install git policycoreutils-python-utils podman
```

The `squashfs` kernel module needs to be running on the host. To check run:
```
lsmod | grep squashfs
```

The host needs to have enough available loop device nodes created to be able to mount various images.
To check existing nodes run:
```
ls /dev/loop[0-9]*
```
To check used nodes run:
```
losetup
```
There should be at least three available nodes.
To create a new node on the host run mknod, for example to create /dev/loop<NUM> run:
```
sudo mknod /dev/loop<NUM> b 7 <NUM>
```

There is a [playbook](runner-host.yml) for deployment of the host on Fedora Cloud Base Image. See [Troubleshooting] for preferred version to be used.

Run a test in a container
-------------------------

Build the container:
```
sudo podman build -t kstest-runner .
```
Note: we can't use rootless containers because of mounting of images in the contaier.

Define the directory for passing of data between the container and the system (via volume):
```
export VOLUME_DIR=${PWD}/data
```

Create subdirs for test inputs (installation image) and outputs (logs):
```
mkdir -p ${VOLUME_DIR}/images
mkdir -p ${VOLUME_DIR}/logs
```
Set the proper selinux context for the volume directory:
```
./set-volume-dir-permissions.sh ${VOLUME_DIR}
```

Download the test subject (installer boot iso):
```
curl http://mirror.karneval.cz/pub/linux/fedora/linux/releases/32/Everything/x86_64/os/images/boot.iso --output ${VOLUME_DIR}/images/boot.iso
```

Run the test:
```
sudo podman run --env KSTESTS_TEST=keyboard -v ${VOLUME_DIR}:/opt/kstest/data --name last-kstest --privileged=true --device=/dev/kvm kstest-runner
```
Instead of keeping named container you can remove it after the test by replacing `--name last-kstest` option with `--rm`.

See the results:
```
tree -L 3 ${VOLUME_DIR}/logs
cat ${VOLUME_DIR}/logs/kstest-*/anaconda/anaconda.log
```

Configuration of the test
-------------------------

Environment variables for the container (`--env` option):
* KSTESTS_TEST - name of the test to be run
* UPDATES_IMAGE - HTTP URL of updates image to be used
* KSTESTS_REPOSITORY - kickstart-tests git repository to be used
* KSTESTS_BRANCH - kickstart-tests git branch to be used
* BOOT_ISO - name of the installer boot iso from ${VOLUME_DIR}/images to be tested (default is "boot.iso")


Troubleshooting
---------------

### Host and container base image version
Fedora 30 seems to be the container base image version with the highest chances of being able to run the tests (with regard to the libvirt version). It was run successfuly on Fedora 30 Cloud Base image deployed with the [playbook](runner-host.yml). On Fedora 30 Workstation on bare metal also Fedora 31 container worked.

Other combinations failed with errors:
* F32 host, F30 container image:
```
2020-05-12 12:36:28.714+0000: 16: error : virCgroupSetValueStr:477 : Unable to write to '/sys/fs/cgroup//cgroup.subtree_control': Operation not supported
```
* F32 container image:
```
2020-05-12 12:51:32,856 INFO: ERROR    Requested operation is not valid: network 'default' is not active
```
* F31 host, F31 container image:
```
2020-05-12 12:54:53.713+0000: 14: error : virCgroupV2ParseControllersFile:281 : Unable to read from '/sys/fs/cgroup/machine/cgroup.controllers': No such file or directory
```
Some likely related BZs:
* [https://bugzilla.redhat.com/show_bug.cgi?id=1751120](https://bugzilla.redhat.com/show_bug.cgi?id=1751120)
* [https://bugzilla.redhat.com/show_bug.cgi?id=1760233](https://bugzilla.redhat.com/show_bug.cgi?id=1760233)

### Tests runnable in container
Some tests require services, resources, or configration in VM hypervisor that might not be working in container. Checking the tests and trying to resolve the issues is TBD.
