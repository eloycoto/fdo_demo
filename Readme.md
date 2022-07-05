# FDO demo with Flotta

# Prerequisites:


- RHEL 9.0 linux already subscribed

```

dnf copr enable @osbuild/osbuild-composer -y
dnf copr enable @osbuild/osbuild -y
dnf install -y qemu-kvm libvirt  osbuild-* weldr-client
systemctl enable libvirtd
```

# Create RPMOstre REPO

Initialize the blueprint:

```
sudo tee -a repo.toml > /dev/null <<EOF
name = "repo"
description = ""
version = "0.0.1"

[[modules]]
name = "ansible-core"
version = "*"

[[customizations.sshkey]]
user = "root"
key = "ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQC9HcO3yWh7xRA9jyNL41teLmWATWCMqU0emzDsdm6IJJXtdBcCiBPIRui88AdTYVY2UWX+qJKJOKFHaKyiouIUTGMtqMPWH58pcZmHrj3DlWcO+k30x47kVsRM5yQKb3H5s2roNAy6rPT3qGIeaYmos3IjJ8aw4j4sYuxqjAj1jlNtvd+JWmhapedHtbg+MaOWaUGygoaPudSQqMZbY0xP5U1SYGHsfgtfbpHZj29WXQluZYxEd4nuVIUe8zwOkYk3Dm/r2gHldovHmQvQFOxdgXLaqL6tWTI61U72RkDyRGSz1GZ/MrQ3uaIUdUzG6ujIIiMSiYYaSuNYWRkT20PO+tMoNjxjWGVCajJczQwt5wsI2lqUp/WkUBsS8kGPjUSPlDmb6qA9aj/prjftdljq5r717ng5Q0cpJo5GhICAQKW8AI0Ht4bYbsckaFzyeHjsY2jkfc98VL0dAnLltNYOJF3UnMUfnt5z2pI3sTOdzl6TrkSem+5q3fT5dQIPo7U= eloy@localhost.localdomain"

[[customizations.user]]
name = "admin"
description = "Administrator account"
password = "admin"
home = "/home/admin/"
groups = ["wheel"]

EOF
```

This will create a image with the ssh key added and the user admin already
created.

Let's build it:

Push the blueprint:
```
composer-cli blueprints push repo.toml
```

Start the compose and get the uuid:
```
composer-cli compose start-ostree repo edge-container
```

Download the image:
```
composer-cli compose image <uuid>
```

Let's start serving the image:

```
skopeo copy oci-archive:<UUID>-container.tar containers-storage:localhost/rfe-mirror:latest
podman run --rm -p 8000:8080 rfe-mirror
```

# FDO server:

Install and enable it:

```
yum -y install fdo-admin-cli
systemctl enable --now fdo-aio
```

Validate that is running as expected:
```
curl http://localhost:8080/ping -X POST
```

Append the following commands to the serviceinfo api config:

```
  files:
  - path: /opt/install-dnf.sh
    permissions: 777
    source_path: /opt/fdo_demo/install-agent-rpm-ostree.sh
  commands:
    - command: /opt/install-dnf.sh
      args:
        - "-i"
        - "$flotta-ip"
        - "-p"
        - "8043"
```

And restart the process to get the changes:
```
systemctl restart fdo-aio
```

The install script can be getted from flotta-project documentation, to be part
of the iso in the next few weeks.

# Let's build the ISO image with FDO enabled:

Start with the blueprint:

```
sudo tee -a simplified.toml > /dev/null <<EOF
name = "simplified_installer"
description = "A rhel-edge simplified-installer image"
version = "0.0.5"
packages = []
modules = []
groups = []
distro = ""

[customizations]
installation_device = "/dev/sda"

[customizations.fdo]
manufacturing_server_url = "http://10.46.41.71:8080"
diun_pub_key_insecure = "true"
EOF
```


Compose:

```
composer-cli blueprints push simplified.toml
composer-cli compose start-ostree simplified_installer edge-simplified-installer --url http://10.46.41.71:8000/repo/ --ref "rhel/9/x86_64/edge"
```
When finished, donwload the iso:

```
composer-cli compose image <uuid>
```

# Start a VM:
```
qemu-img create -f qcow2 vm.qcow2 20G
virt-install \
  --name="fdo-image"\
  --disk path="vm.qcow2",format=qcow2 \
  --ram 3072 \
  --vcpus 2 \
  --os-type linux \
  --cdrom "/opt/fdo_demo/4753d983-a5d7-481d-a97a-35157e0a2dc4-simplified-installer.iso" \
  --boot uefi,loader_ro=yes,loader_type=pflash,nvram_template=/usr/share/edk2/ovmf/OVMF_VARS.fd,loader_secure=no \
  --wait=15
```

And let's wait until the device shows at the kubernetes cluster:
```
kubectl get edgedevice
```

