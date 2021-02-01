# Pilotboat

Pilotboat serves iPXE scripts and [cloudinit auto-install
configs](https://ubuntu.com/server/docs/install/autoinstall) to clients to
automate bare metal provisioning of Ubuntu servers.

Inspiration for pilotboat was taken from [CoreOS'
Matchbox](https://github.com/poseidon/matchbox).

## Installation

TBD

## Getting Started

This getting start guide assumes that pilotboat will be provisioning UEFI
machines using PXE boot.

The tl;dr of this guide is as follows

- Configure a DHCP server to chainload iPXE
- Configure a DHCP server to boot the `boot.ipxe` script
- Create a cloud-init autoinstallation config
- Configure pilotboat to provision servers

### Configure the DHCP Server

Pilotboat requires a DHCP server which can chainboot iPXE to already be
configured so that pilotboat can serve iPXE scripts to the client.

An example Dnsmasq configuration could look like the following

`/etc/dnsmaq.d/pilotboat.conf`

```
# Don't run a dns server
port=0

# Proxy dhcp requests to another server
dhcp-range=192.168.0.1,proxy,255.255.255.0

enable-tftp
tftp-root=/var/lib/tftpboot

# If request comes from older PXE ROM, chainload to iPXE (via TFTP)
pxe-service=tag:#ipxe,X86-64_EFI,
# If request comes from iPXE user class, set tag "ipxe"
dhcp-userclass=set:ipxe,iPXE
# Point ipxe tagged requests to the pilotboat iPXE boot script (via HTTP)
pxe-service=tag:ipxe,x86PC,"iPXE",http://pilotboat.example.com:8080/boot.ipxe

# verbose
log-queries
log-dhcp
```

### Create a Cloud-init Autoinstall Config

Create a cloud-init autoinstallation configuration file as described
[here](https://ubuntu.com/server/docs/install/autoinstall) and save it into
`/etc/pilotboat/` (e.g. `/etc/pilotboat/cloudinit.yaml`).

### Configure Pilotboat

Create `/etc/pilotboat/config.yaml` to configure pilotboat.

``` yaml
# Pilotboat configuration (optional)
pilotboat:
  # Set the log level (optional)
  # Possible values are debug, info, warning or error
  log-level: info

# A list of servers to deploy to
servers:
    # MAC address of the server
  - mac-address: 15:A1:F3:2E:15:39
    # The cloudinit autoinstall configuration to use for this server
    cloud-init: /etc/pilotboat/cloudinit.yaml
    # Location of the ubuntu installer iso (can either be a url or path)
    image: https://releases.ubuntu.com/20.04.1/ubuntu-20.04.1-live-server-amd64.iso

  - mac-address: 74:F1:C6:EC:56:10
    cloud-init: /etc/pilotboat/cloudinit-arm.yaml
    image: /var/lib/pilotboat/ubuntu-server-arm64.iso
  
```

## Folders

**`/etc/pilotboat/`**  
&nbsp;&nbsp;&nbsp;&nbsp;Pilotboat configuration and cloud-init configs

**`/var/lib/pilotboat/`**  
&nbsp;&nbsp;&nbsp;&nbsp;Image storage location. All images which are downloaded
are stored here.
