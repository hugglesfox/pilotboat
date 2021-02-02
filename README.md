# Pilotboat

If your servers are cattle rather then pets, reinstalling and reconfiguring
them can get a bit tedious, especially if you have more then one.

That's where Pilotboat comes in. Pilotboat serves iPXE scripts and [cloudinit
auto-install configs](https://ubuntu.com/server/docs/install/autoinstall) to
help automate provisioning of bare metal Ubuntu servers.

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
tftp-root=/srv/tftp

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
`/srv/pilotboat/configs/` (e.g. `/srv/pilotboat/configs/cloudinit.yaml`).

### Configure Pilotboat

Create `/etc/pilotboat/config.yaml` to configure pilotboat.

``` yaml
# Pilotboat configuration
pilotboat:
  # Set the log level (optional, defaults to warning)
  # Possible values are debug, info, warning or error
  log-level: info
  # URL of Ubuntu ISO to deploy
  image: http://releases.ubuntu.com/focal/ubuntu-20.04.1-live-server-amd64.iso

# A list of servers to deploy to
servers:
    # MAC address of the server
  - mac-address: 15:A1:F3:2E:15:39
    # The cloudinit autoinstall configuration to use for this server
    # Defaults to looking in /etc/pilotboat/configs/
    cloud-init: cloudinit.yaml

  - mac-address: 74:F1:C6:EC:56:10
    cloud-init: /etc/pilotboat/configs/cloudinit-arm.yaml
```

## Architecture

The process which pilotboat uses to provision machines can be described by the
following steps

1. Any iso images which are defined in the config file which have not been
   downloaded to `/srv/pilotboat/images` will be.
2. Mount then copy the initrd and kernel from the iso if not already done.
3. iPXE runs the `http://pilotboat.example.com/boot.ipxe` script from the
   pilotboat server as instructed by the DHCP server.
4. The `boot.ipxe` script then boots the iPXE script found at
   `http://pilotboat.example.com/ipxe?mac=${mac}`.
5. Pilotboat will try to match the MAC address given by the `mac` parameter with
   a server defined in the pilotboat config. If there is no match then a HTTP
   403 status code is returned. Otherwise a boot config such as the following is
   returned
  
   ```
   #!ipxe
   kernel /images/vmlinuz root=/dev/ram0 ramdisk_size=1500000 ip=dhcp url=http://pilotboat.example.com/images/ubuntu-server.iso autoinstall ds=nocloud-net;s=http://pilotboat.example.com/cloud-init?mac=${mac}
   initrd /images/initrd
   boot
   ```
6. The Ubuntu installer will then start.

### Folders

**`/etc/pilotboat/`**  
Pilotboat configuration (See [Getting Started](#configure-pilotboat)).

**`/srv/pilotboat/images/`**  
Image storage. For an example, if pilotboat downloads an iso, it'd be stored and
served from here.

**`/srv/pilotboat/configs/`**  
Cloud-init autoinstall configuration storage (see [Getting
Started](#create-a-cloud-init-autoinstall-config))

## Contributing

Find a problem or want to add a missing feature?

Feel free to create an issue or a pull request.

