# Mustafar Lab

A self contained penetration testing lab built on a single Kali Linux machine.
One attacker, one intentionally vulnerable victim, wired together on a private
network that never touches the internet or any real LAN. Built as hands on
practice for offensive security coursework.

## Overview

The lab models the most basic security relationship: an attacker machine and a
victim machine on the same network segment. The attacker is a full Kali Linux
install. The victim is Metasploitable2, a deliberately broken Linux image packed
with vulnerable services for training.

Everything runs on one laptop. The victim lives inside VirtualBox on a host only
network, which means only the host (Kali) can reach it and it has no route out.
This keeps a machine that is full of real, exploitable holes completely sealed
off from anything that matters.

## Topology

* Attacker: Kali Linux (bare metal host), 192.168.56.1
* Victim: Metasploitable2 in VirtualBox (named "hoth"), 192.168.56.101
* Network: VirtualBox host only interface vboxnet0, 192.168.56.0/24, no gateway,
  no internet access

## Components

* Host OS: Kali Linux
* Virtualization: VirtualBox 7.2.16
* Victim image: Metasploitable2 (vsftpd, Samba, Tomcat, and roughly thirty other
  vulnerable services)
* Core tooling: nmap, Metasploit Framework, Wireshark

## Naming

Folder and machine names follow a Star Wars theme. The lab root on the attacker
is ~/mustafar, Vader's volcanic forge. Victim disks live in ~/mustafar/carbonite.
The first victim is named hoth, the rebel base Vader personally assaults, which
fits the attacker and victim roles.

## Setup Summary

1. Prepared Kali: full system upgrade, installed kali-linux-default, initialized
   the Metasploit database with msfdb init, verified core tools.
2. Built the workspace: ~/mustafar/{scans,payloads,captures,notes}.
3. Downloaded Metasploitable2 and attached its existing virtual disk to a new
   VirtualBox VM on the IDE controller. SATA and EFI both cause this old image to
   fail to boot, so the disk goes on IDE and EFI stays off.
4. Created an isolated host only network (vboxnet0) and attached the victim's
   only network card to it, with no NAT adapter, so the victim has no path to the
   internet.
5. Booted the victim and confirmed reachability from Kali with ping.

### Key commands

```bash
# Create the private host only network and give Kali its address
VBoxManage hostonlyif create
sudo VBoxManage hostonlyif ipconfig vboxnet0 --ip 192.168.56.1 --netmask 255.255.255.0

# Point the victim at that network only, with no internet facing adapter
VBoxManage modifyvm hoth --nic1 hostonly --hostonlyadapter1 vboxnet0

# Disable the I/O APIC so the old kernel boots
VBoxManage modifyvm hoth --ioapic off

# Boot it
VBoxManage startvm hoth --type gui
```

## Troubleshooting Notes

Real problems hit during the build, kept here because solving them was half the
lesson.

* VirtualBox could not create the host only network: the vboxdrv kernel driver
  was missing. Fixed by installing virtualbox-dkms and the matching
  linux-headers, which build the driver for the running kernel.
* Starting the VM then failed with VERR_SUPDRV_COMPONENT_NOT_FOUND: the network
  filter driver had not registered with the main driver. Fixed by reloading
  vboxnetflt and loading vboxnetadp alongside vboxdrv.
* The VM window opened and closed instantly: the old kernel aborts on
  VirtualBox's I/O APIC. Fixed by disabling ioapic on the VM.

## Current Status

Reconnaissance complete:

* Full TCP port sweep with nmap (all 65535 ports), saved to scans/.
* Service and version fingerprinting with nmap -sV. Roughly thirty open services
  identified, including vsftpd 2.3.4, Samba 3.x, Apache Tomcat, and a stray root
  bind shell on port 1524.

Next: exploitation, starting with the vsftpd 2.3.4 backdoor, then moving to
Samba, the Java services, and the vulnerable web applications.

## Planned Expansion

* A second physical victim (a Windows laptop) reached over a direct cable, for
  networked attacks against a real host.
* A separate malware analysis segment using FLARE-VM, kept on its own isolated
  network and never bridged.
* SEED Labs network security exercises.

## Safety

The victim is intentionally insecure and is never exposed to any untrusted
network. It sits on an isolated host only segment with no route to the internet
or any real LAN. Do not reproduce this setup on a bridged or internet facing
network.
