
# CoreOS on Libvirt Virtual Hardware

CoreOS can be booted and configured on virtual hardware within a libvirt environment (under Linux) with different network services running as Docker containers on the `docker0` virtual bridge. Client VMs or even baremetal hardware attached to the bridge can be booted and configured from the network.

Docker containers run on the `docker0` virtual bridge, typically on a subnet 172.17.0.0/16. Docker assigns IPs to containers started through the docker cli, but the bridge does not run a DHCP service. List network bridges on your host and inspect the bridge Docker 1.9+ created (Docker cli refers to `docker0` as `bridge`).

    brctl show
    docker network inspect bridge

## Config Service

Setup `coreos/bootcfg` according to the [docs](bootcfg.md). Pull the `coreos/bootcfg` image, prepare a data volume with `Machine` definitions, `Spec` definitions and cloud configs. Optionally, include a volume of downloaded image assets.

Run the `bootcfg` container to serve configs for any of the network environments we'll discuss next.

    docker run -p 8080:8080 --name=bootcfg --rm -v $PWD/data:/data:Z -v $PWD/images:/images:Z coreos/bootcfg:latest -address=0.0.0.0:8080 -log-level=debug

Note, the `Spec` examples in [data](../data) point `cloud-config-url` kernel options to 172.17.0.2, the first container IP Docker is likely to assign. Ensure your kernel option urls point to where `bootcfg` runs.

    docker inspect bootcfg   # look for Networks, bridge, IPAddress

## Network Setups

We'll show how to setup PXE, iPXE, or Pixiecore network boot environments on the `docker0` bridge and configure them to use `bootcfg`.

The [dnsmasq](http://www.thekelleys.org.uk/dnsmasq/doc.html) program and included [coreos/dnsmasq](../dockerfiles/dnsmasq) docker image will be used to run DHCP and TFTP. Build the Docker image before continuing.

### PXE

To boot PXE clients, configure a PXE network environment to [chainload iPXE](http://ipxe.org/howto/chainloading). The iPXE setup below configures DHCP to send `undionly.kpxe` over TFTP to older PXE clients for this purpose.

With `dnsmasq`, the relevant `dnsmasq.conf` settings would be:

    enable-tftp
    tftp-root=/var/lib/tftpboot
    # if PXE request came from regular firmware, serve iPXE firmware (via TFTP)
    dhcp-boot=tag:!ipxe,undionly.kpxe

### iPXE

Create a PXE/iPXE network environment by running a PXE-enabled DHCP server and TFTP server on the `docker0` bridge, alongside `bootcfg`.

```
sudo docker run --rm --cap-add=NET_ADMIN coreos/dnsmasq -d -q --dhcp-range=172.17.0.43,172.17.0.99 --enable-tftp --tftp-root=/var/lib/tftpboot --dhcp-userclass=set:ipxe,iPXE --dhcp-boot=tag:#ipxe,undionly.kpxe --dhcp-boot=tag:ipxe,http://172.17.0.2:8080/boot.ipxe
```

The `coreos/dnsmasq` image runs DHCP and TFTP as a container. It allocates IPs in the `docker0` subnet to VMs and sends options to chainload older PXE clients to iPXE. iPXE clients are pointed to the `bootcfg` service iPXE endpoint (assumed to be running on 172.17.0.2:8080).

To run dnsmasq as a service, rather than from the commandline, the `dnsmasq.conf` might be:

```
# dnsmasq.conf
dhcp-range=172.17.0.43,172.17.0.99,30m
enable-tftp
tftp-root=/var/lib/tftpboot
# set tag "ipxe" if request comes from iPXE ("iPXE" user class)
dhcp-userclass=set:ipxe,iPXE
# if PXE request came from regular firmware, serve iPXE firmware (via TFTP)
dhcp-boot=tag:!ipxe,undionly.kpxe
# if PXE request came from iPXE, serve an iPXE boot script (via HTTP)
dhcp-boot=tag:ipxe,http://172.17.0.2:8080/boot.ipxe
```

<img src='img/libvirt-ipxe.png' class="img-center" alt="Libvirt iPXE network environment"/>

Continue to [clients](#clients) to create a client VM or attach a baremetal machine to boot.

### Pixiecore

Create a Pixiecore network environment by running a DHCP server and `danderson/pixiecore` on the `docker0` bridge, alongside `bootcfg`. Pixiecore is a combined proxyDHCP/TFTP/HTTP server.

Run a DHCP server (not PXE-enabled)

```
sudo docker run --rm --cap-add=NET_ADMIN coreos/dnsmasq -d -q --dhcp-range=172.17.0.43,172.17.0.99
```

Run Pixiecore, using a script which detects the `bootcfg` container IP:port on docker0.

    ./scripts/pixiecore

Continue to [clients](#clients) to create a client VM or attach a baremetal machine to boot.

## Clients

Once a network environment is prepared to boot client machines, create a libvirt VM configured to PXE boot or attach a baremetal machine to your Docker host.

### libvirt VM

Use `virt-manager` to create a new client VM. Select Network Boot with PXE and for the network selection, choose "Specify Shared Device" with the bridge name `docker0`.

<img src='img/virt-manager.png' class="img-center" alt="Virt-Manager showing PXE network boot method"/>

The VM should PXE boot using the boot config and cloud config based on its UUID, MAC address, or your configured defaults. The `virt-manager` shows the UUID and the MAC address of the NIC on the shared bridge, which you can use when naming configs.

### Bare Metal

Connect a baremetal client machine to your libvirt Docker host machine and ensure that the client's boot firmware (probably BIOS) has been configured to prefer PXE booting.

Find the network interface and attach it to the virtual bridge.

    ip link show                      # find new link e.g. enp0s20u2
    brctl addif docker0 enp0s20u2

Restart the client machine and it should PXE boot using the boot config and cloud config based on its UUID, MAC address, or your configured defaults.

## Next

If you'd like to boot and configure a baremetal machine network, follow the [baremetal guide](physical-hardware.md).