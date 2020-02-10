# Metal-Stack Mini-Lab

Small lab to start two leaf switches and the metal-api to try `metalctl` and the creation of machines.

This requires:

- vagrant >= 2.2.7 with vagrant-libvirt plugin >= 0.0.45 for running the switch and machine VMs
- docker for using containerized `metalctl`
- ansible
- kvm as hypervisor for the VMs
- [ovmf](https://wiki.ubuntu.com/UEFI/OVMF) to have a uefi firmware for virtual machines
- [kind](https://github.com/kubernetes-sigs/kind/releases) >= 0.7.0 to start the metal control-plane on a kubernetes cluster
- [helm](https://github.com/helm/helm/releases) >= 3.0.3 to deploy the metal control-plane to this cluster

Known limitations:

- to keep the demo small there is no EVPN
- machine destroy does not work (virtual-bmc is buggy)
- login to the machines is only possible with virsh console

 ```bash
# Install vagrant
wget https://releases.hashicorp.com/vagrant/2.2.7/vagrant_2.2.7_x86_64.deb
apt-get install ./vagrant_2.2.7_x86_64.deb docker.io qemu-kvm virt-manager ovmf net-tools libvirt-dev

# Ensure that your user is member of the group "libvirt"
usermod -G libvirt -a ${USER}

# Install libvirt plugin for vagrant
vagrant plugin install vagrant-libvirt

# Install helm from https://github.com/helm/helm/releases
# Install kind from https://github.com/kubernetes-sigs/kind/releases
```

Try it out:

```bash
# Start kind with a metal-api instance as well as some vagrant VMs with two leaf switches and two machine skeletons.
> ./start.sh

# Two machines in status `PXE booting` are visible with `metalctl machine ls`
> docker run -it \
    -e METALCTL_HMAC=metal-admin \
    -e METALCTL_URL=http://api.192.168.121.1.xip.io:8080/metal \
    registry.fi-ts.io/metal/metalctl \
        machine ls

ID                                          LAST EVENT   WHEN     AGE  HOSTNAME  PROJECT  SIZE          IMAGE  PARTITION
e0ab02d2-27cd-5a5e-8efc-080ba80cf258        PXE Booting  3s
2294c949-88f6-5390-8154-fa53d93a3313        PXE Booting  5s

# Wait until the machines reach the waiting state
> docker run -it \
    -e METALCTL_HMAC=metal-admin \
    -e METALCTL_URL=http://api.192.168.121.1.xip.io:8080/metal \
    registry.fi-ts.io/metal/metalctl \
        machine ls

ID                                          LAST EVENT   WHEN     AGE  HOSTNAME  PROJECT  SIZE          IMAGE  PARTITION
e0ab02d2-27cd-5a5e-8efc-080ba80cf258        Waiting      8s                               v1-small-x86         vagrant
2294c949-88f6-5390-8154-fa53d93a3313        Waiting      8s                               v1-small-x86         vagrant

# Create a machine with `metalctl machine create`
> docker run -it \
    -e METALCTL_HMAC=metal-admin \
    -e METALCTL_URL=http://api.192.168.121.1.xip.io:8080/metal \
    registry.fi-ts.io/metal/metalctl \
        machine create \
        --description test \
        --name machine \
        --hostname machine \
        --project 00000000-0000-0000-0000-000000000000 \
        --partition vagrant \
        --image ubuntu-19.10 \
        --size v1-small-x86

# See the installation process in action
> virsh console metal_machine01/02
...
Ubuntu 19.10 machine ttyS0

machine login:

# One machine is now installed and has status "Phoned Home"
> docker run -it \
    -e METALCTL_HMAC=metal-admin \
    -e METALCTL_URL=http://api.192.168.121.1.xip.io:8080/metal \
    registry.fi-ts.io/metal/metalctl \
        machine ls
ID                                          LAST EVENT   WHEN   AGE     HOSTNAME  PROJECT                               SIZE          IMAGE         PARTITION
e0ab02d2-27cd-5a5e-8efc-080ba80cf258        Phoned Home  2s     21s     machine   00000000-0000-0000-0000-000000000000  v1-small-x86  Ubuntu 19.10  vagrant
2294c949-88f6-5390-8154-fa53d93a3313        Waiting      8s                                                             v1-small-x86                vagrant

# Login with user name metal and the console password from
> docker run -it \
    -e METALCTL_HMAC=metal-admin \
    -e METALCTL_URL=http://api.192.168.121.1.xip.io:8080/metal \
    registry.fi-ts.io/metal/metalctl \
        machine describe e0ab02d2-27cd-5a5e-8efc-080ba80cf258 | grep password \

consolepassword: ...
```
