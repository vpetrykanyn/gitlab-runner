# Using libvirt with the Custom executor

Using [libvirt](https://libvirt.org/), the Custom executor will create a
new disk and VM for every job it executes, after which the disk and VM
will be deleted.

This example is inspired by a Community Contribution
[!464](https://gitlab.com/gitlab-org/gitlab-runner/merge_requests/464)
to add libvirt as a GitLab Runner executor.

This document does not try to explain how to set up libvirt, since it's
out of scope. However, this executor was tested using [GCP Nested
Virtualization](https://cloud.google.com/compute/docs/instances/enable-nested-virtualization-vm-instances),
which also has [details on how to setup
libvirt](https://cloud.google.com/compute/docs/instances/enable-nested-virtualization-vm-instances#starting_a_private_bridge_between_the_host_and_nested_vms)
with bridge networking. This example will use the `default` network that
comes with when installing libvirt so make sure it's running.

This executor requires bridge networking since each VM needs to have
it's own dedicated IP address so GitLab Runner can SSH inside of it to
run commands. An SSH key can be generated [using the following
commands](https://docs.gitlab.com/ee/ssh/#generating-a-new-ssh-key-pair).

A base disk VM image is created so that dependencies are not downloaded
every build. In the following example,
[virt-builder](http://libguestfs.org/virt-builder.1.html) is used to
create a disk VM image.

```sh
virt-builder debian-9 \
    --size 8G \
    --output /var/lib/libvirt/images/gitlab-runner-base.qcow2 \
    --format qcow2 \
    --hostname gitlab-runner-stretch \
    --network \
    --install curl \
    --run-command "curl -L https://packages.gitlab.com/install/repositories/runner/gitlab-runner/script.deb.sh | bash" \
    --run-command "curl -s https://packagecloud.io/install/repositories/github/git-lfs/script.deb.sh | bash" \
    --run-command 'useradd -m -p "" gitlab-runner -s /bin/bash' \
    --install gitlab-runner,git,git-lfs,openssh-server \
    --run-command "git lfs install --skip-repo" \
    --ssh-inject gitlab-runner:file:/root/.ssh/id_rsa.pub \
    --run-command "echo 'gitlab-runner ALL=(ALL) NOPASSWD: ALL' >> /etc/sudoers" \
    --run-command "sed -E 's/GRUB_CMDLINE_LINUX=\"\"/GRUB_CMDLINE_LINUX=\"net.ifnames=0 biosdevname=0\"/' -i /etc/default/grub" \
    --run-command "grub-mkconfig -o /boot/grub/grub.cfg" \
    --run-command "echo 'auto eth0' >> /etc/network/interfaces" \
    --run-command "echo 'allow-hotplug eth0' >> /etc/network/interfaces" \
    --run-command "echo 'iface eth0 inet dhcp' >> /etc/network/interfaces"
```

The command above will install all the
[prerequisites](../custom.md#prerequisite-software-for-running-a-job) specified
earlier.

`virt-builder` will set a root password automatically which is printed
at the end. If you want to specify a password yourself, pass
[`--root-password
password:$SOME_PASSWORD`](http://libguestfs.org/virt-builder.1.html#setting-the-root-password).

## Configuration

The following is an example of Runner configuration for libvirt:

```toml
concurrent = 1
check_interval = 0

[session_server]
  session_timeout = 1800

[[runners]]
  name = "libvirt-executor"
  url = "https://gitlab.com/"
  token = "xxxxx"
  executor = "custom"
  builds_dir = "/home/gitlab-runner/builds"
  cache_dir = "/home/gitlab-runner/cache"
  [runners.custom_build_dir]
  [runners.cache]
    [runners.cache.s3]
    [runners.cache.gcs]
  [runners.custom]
    prepare_exec = "/opt/libvirt-executor/prepare.sh" # Path to a bash script to create VM.
    run_exec = "/opt/libvirt-executor/run.sh" # Path to a bash script to run script inside of VM over ssh.
    cleanup_exec = "/opt/libvirt-executor/cleanup.sh" # Path to a bash script to delete VM and disks.
```

## Base

Each stage ([prepare](#prepare), [run](#run), and [cleanup](#cleanup))
will use the base script below to generate variables that are used
throughout other scripts.

It's important that this script is located in the same directory as the
other scripts, in this case `/opt/libivirt-executor/`.

```sh
#!/usr/bin/env bash

# /opt/libivrt-executor/base.sh

VM_IMAGES_PATH="/var/lib/libvirt/images"
BASE_VM_IMAGE="$VM_IMAGES_PATH/gitlab-runner-base.qcow2"
VM_ID="runner-$CUSTOM_ENV_CI_RUNNER_ID-project-$CUSTOM_ENV_CI_PROJECT_ID-concurrent-$CUSTOM_ENV_CI_CONCURRENT_PROJECT_ID-job-$CUSTOM_ENV_CI_JOB_ID"
VM_IMAGE="$VM_IMAGES_PATH/$VM_ID.qcow2"

_get_vm_ip() {
    virsh -q domifaddr "$VM_ID" | awk '{print $4}' | sed -E 's|/([0-9]+)?$||'
}
```

## Prepare

The prepare script:

- Copies the disk to a new path.
- Installs a new VM from the copied disk.
- Waits for the VM to get an IP.
- Waits for ssh to respond on the VM.

```sh
#!/usr/bin/env bash

# /opt/libivrt-executor/prepare.sh

currentDir="$( cd "$( dirname "${BASH_SOURCE[0]}" )" >/dev/null 2>&1 && pwd )"
source ${currentDir}/base.sh # Get variables from base script.

set -eo pipefail

# trap any error, and mark it as a system failure.
trap "exit $SYSTEM_FAILURE_EXIT_CODE" ERR

# Copy base disk to use for Job.
cp "$BASE_VM_IMAGE" "$VM_IMAGE"

# Install the VM
virt-install \
    --name "$VM_ID" \
    --os-variant debian9 \
    --disk "$VM_IMAGE" \
    --import \
    --vcpus=2 \
    --ram=2048 \
    --network default \
    --graphics none \
    --noautoconsole

# Wait for VM to get IP
echo 'Waiting for VM to get IP'
for i in $(seq 1 30); do
    VM_IP=$(_get_vm_ip)

    if [ -n "$VM_IP" ]; then
        echo "VM got IP: $VM_IP"
        break
    fi

    if [ "$i" == "30" ]; then
        echo 'Waited 30 seconds for VM to start, exiting...'
        # Inform GitLab Runner that this is a system failure, so it
        # should be retried.
        exit "$SYSTEM_FAILURE_EXIT_CODE"
    fi

    sleep 1s
done

# Wait for ssh to become available
echo "Waiting for sshd to be available"
for i in $(seq 1 30); do
    if ssh -i /root/.ssh/id_rsa -o StrictHostKeyChecking=no gitlab-runner@"$VM_IP" >/dev/null 2>/dev/null; then
        break
    fi

    if [ "$i" == "30" ]; then
        echo 'Waited 30 seconds for sshd to start, exiting...'
        # Inform GitLab Runner that this is a system failure, so it
        # should be retried.
        exit "$SYSTEM_FAILURE_EXIT_CODE"
    fi

    sleep 1s
done
```

## Run

This will run the script generated by GitLab Runner by sending
the content of the script to the VM via `STDIN` through SSH.

```sh
#!/usr/bin/env bash

# /opt/libivrt-executor/run.sh

currentDir="$( cd "$( dirname "${BASH_SOURCE[0]}" )" >/dev/null 2>&1 && pwd )"
source ${currentDir}/base.sh # Get variables from base script.

VM_IP=$(_get_vm_ip)

ssh -i /root/.ssh/id_rsa -o StrictHostKeyChecking=no gitlab-runner@"$VM_IP" /bin/bash < "${1}"
if [ $? -ne 0 ]; then
    # Exit using the variable, to make the build as failure in GitLab
    # CI.
    exit "$BUILD_FAILURE_EXIT_CODE"
fi
```

## Cleanup

This script removes the VM and deletes the disk.

```sh
#!/usr/bin/env bash

# /opt/libivrt-executor/cleanup.sh

currentDir="$( cd "$( dirname "${BASH_SOURCE[0]}" )" >/dev/null 2>&1 && pwd )"
source ${currentDir}/base.sh # Get variables from base script.

set -eo pipefail

# Destroy VM.
virsh shutdown "$VM_ID"

# Undefine VM.
virsh undefine "$VM_ID"

# Delete VM disk.
if [ -f "$VM_IMAGE" ]; then
    rm "$VM_IMAGE"
fi
```
