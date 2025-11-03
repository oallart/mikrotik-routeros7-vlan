# Mikrotik RouterOS 7.x VLAN and automation howto

*This document exists because I never found a reference on the topic in a concise written form, so I took all my research and experience and wrote the missing manual I wish I was given when I started. I hope it helps you on your journey.*

## Preamble

Defining VLANS in RouterOS v7 is notoriously complicated, under-documented, and incredibly easy to do wrong. I have found countless incomplete or wrong examples and tutorials online.

The only proper reference is in the form of a couple youtube videos, especially [this one](https://www.youtube.com/watch?v=US2EU6cgHQU&t=19s)

The configuration of VLANs can be achieved and even automated (see further down) provided a few precautions are taken, and the right order of configuration is respected:
- bridge creation with vlan filtering
- bond (LAG) creation if needed
- untagged ingress configuration
- tagged ingress configuration
- tagged egress configuration

*Note 1:* untagged egress does not need configuration, it's dynamically done by ROS

*Note 2:* the above order also covers hybrid port definition (more on this below)

**It may be useful to think of VLAN configuration on RouterOS 7 as a firewall configuration with filtering defined as egress and ingress rules.**

## Some definitions

- trunk: Unlike cisco wants to pretend, a trunk is not a bond (LAG) but a tagged VLAN interface
- access port: an untagged VLAN interface
- hybrid port: an interface both tagged and untagged

## A special note on Hardware Offloading

It matters *enormously* because if not configured properly, VLAN traffic will be CPU processed, not switch-chip processed, meaning it will be slow and use resources unnecessarily.
Most of the work here ensure we get L2 hardware offload. The first requirement is to have `vlan-filtering=yes` on the bridge.

## Starting clean

A RouterOS device very likely has defaults you don't want if you are to use and manage VLANs, especially in an automated way. A new device is also very likely to have outdated software.

Starting clean is the best way in my opinion. There are two main ways to do that in my opinion:
1. reset on boot
2. netinstall (bonus flashing a new ROS version at the same time)

Read below for more on that. I can't guarantee the steps are 100% accurate for your use case.

### Manual reset and update

- Stop the device and power it back on while having the reset button pressed for 12-21 seconds (see manual)
- Connect port 1 of the device directly to your machine
- Use winbox to connect to the device and set your password, skip the license, don't use defaults etc.
- Download the appropriate update for your arch from the mikrotik website and drop it in files via winbox, restart the device
- Dont't forget to run `/system routerboard upgrade`

### Netinstall update and reset

- Download the appropriate update for your arch from the mikrotik website
- Download [Netinstall (CLI Linux)](https://mikrotik.com/download) under RouterOS v7, X86
- Poweroff the device and connect it to your machine directly on port 1 or the boot port of your device
- Run `netinstall` by supplying the name of your ethernet device (configure it to be in 192.168.88.0/24) and the update package, IE
```
sudo ./netinstall-cli -e -b -i enp3s0 -a 192.168.88.101 -v routeros-7.19.3-arm64.npk
```
- Power on the device by pressing and holding the reset button for at least 23 seconds to place it in boot mode. `netinstall` should pick it and proceed.
- Dont't forget to run `/system routerboard upgrade`


## Defining VLANs manualy

### Bridge

To enable hardware offloading, the bridge **must** have vlan filtering enabled

```
/interface bridge add name=bridge vlan-filtering=yes
```

### LAG (bond)

If you use link aggregation (LAG), define it now
```
/interface bonding add name=lag1 slaves=sfp-sfplus1,sfp-sfpplus2 mode=802.3ad
```

### Untagged ingress

The untagged interface definition is done when adding a port to the bridge, specifying the PVID that will internally mark the VLAN traffic. Filtering exludes tagged traffic (unless you need a hybrid port, see further).
```
/interface bridge port add bridge=bridge frame-types=admit-only-untagged-and-priority-tagged interface=ether1 pvid=20
```

As mentioned earlier, **there is no need for configuring untagged egress**.

### Tagged ingress configuration

The tagged interface definition is done similarly to the untagged but with a different filter accepting only tagged frames (again unless you need a hybrid port).
```
/interface bridge port add bridge=bridge frame-types=admit-only-vlan-tagged interface=ether5
```

you can also define your trunk LAG by specifying `interface=lag1` here.

### Tagged egress configuration

This is probably where most of the confusion takes place. No, you **don't need to define VLANs in `/interface/vlan`.** (although it helps with clarity) but you DO have to be careful when creating the entries so that they are grouped by *VLAN*, and not by *interface*.
The configuration is done in the subset `vlan` of `bridge`
```
/interface bridge vlan add bridge=bridge vlan-ids=20 tagged=ether5,lag1
/interface bridge vlan add bridge=bridge vlan-ids=50 tagged=ether5,lag1
/interface bridge vlan add bridge=bridge vlan-ids=100 tagged=ether5,lag1
/interface bridge vlan add bridge=bridge vlan-ids=201 tagged=lag1
/interface bridge vlan add bridge=bridge vlan-ids=202 tagged=lag1
/interface bridge vlan add bridge=bridge vlan-ids=204 tagged=lag1
```

*Note:* grouping by VLAN will be necessary for automating (see further)

### Hybrid interface notes

A *hybrid* interface is one that is both tagged and untagged at the same time. Therefore it has the `PVID` set to the *untagged* VLAN and has one or more *tagged* VLANs defined.

The first thing to note about a hybrid port is that the *ingress filter* has to accomodate both types.
```
/interface bridge port add bridge=bridge frame-types=admit-all interface=ether10 pvid=204
```

The *egress* configuration is the same as a normal tagged port

### Updating a trunk

Updating requires the annoying id number which is **dynamically printed and unpredictable**, meaning we need to run a `print` command for every operation

Ingress does not need to be updated unless the port is removed
Egress is updated by doign the following steps
1. `/interface/bridge/vlan/export` to show the running config
2. `/interface/bridge/vlan/print` to show the dynamic entries and IDs
3. `/interface/bridge/vlan/set tagged=lag1,ether1,ether2,ether10 numbers=3,4,5`

`numbers` are a single or collection of IDs from the `print` command executed before
`tagged` are a single or collection of interface names associated with a given VLAN as per your requirement (add or delete) from the configuration shown in `export`

**Be especially carful to match the ID line to the VLAN config and interfaces you need**

### Updating an access port

Much like the trunk, we need a few steps
1. `/interface/bridge/port/export` to show the running config
2. `/interface/bridge/port/print where interface=ether5` to show the dynamic entries and IDs
3. `/interface/bridge/port/set pvid=120 numbers=12`

I highly recomend using a single entry in `numbers` to the ID of the matching entry in `print`. Also double check that it matches your target interface against the running config from `export` when setting the `pvid`.

### Checking for operation

Some useful commands to check for proper operation of VLANs

*Note:* some commands only show data when the interface is **active**. `export` is generally a safer way to see what is actually configured.

#### checking for hardware offloading

Check for **H** *hardware-offload* as flag on the left in the output of
```
/interface/bridge/port/print
```

#### checking for untagged ports

**Important:** This table is only populated by *active* interfaces, aka those who have the **R** *Running* in `/interface/print`
```
/interface/bridge/vlan/print
```

#### checking for connectivity on a given VLAN

The easiest way is to add an IP address on the VLAN interface. This requires seting up a VLAN first
```
/interface vlan add interface=bridge name=test vlan-id=800
/interface bridge port add bridge=bridge frame-types=admit-only-untagged-and-priority-tagged interface=ether8 pvid=800
/ip address add address=192.168.1.25/24 interface=test network=192.168.1.0
```
You may need to add routing information if you are trying to reach from a different subnet.

## Automation

*Note 1:* At this stage, automation of hybrid ports is still a work in progress.

*Note 2:* Do **NOT** mix manual and automated configuration. If something is set manually, keep it that way. If something is configured via terraform, don't mess with it manually. You have been warned.

The best tool for automating ROS in my view is terraform. Its provider is much more advanced than the ansible one.
It also can be converted to use with pulumi if you want a different language than hcl.
Still, the provider only does so much and a lot of the logic needs to be computed in hcl before being passed to the API.
As such, there is a chicken and egg issue:
- configuring API for automation
- automating API configuration

I have taken the approach of a script that sets up the device all the way to the automation part.
That meant preconfiguring the bridge and a management port on a given VLAN.
It also configures the API requirements like certificates.

For my use, I leverage flat configurations in yaml file and tofu worskspaces to distinguish the yaml files based on device name.
The state is stored in a minio s3 type object storage, although it could just be a flat local directory.

### pre-requisite: API and access

The chicken and egg problem: I want to automate configuration, but to automate configuration I need some basic configuration.
My solution: A script (initial_script) that sets up the bare minimum so we can access the device through the https REST API.
It sets up:
- hostname
- self-signed certificate (required for https)
- api access using the self-sgned cert
- api user (prompts for password)
- common bridge with vlan-filtering enabled
- management VLAN
- management port for that management VLAN
- management IP address, route and DNS (nameserver is only useful for updates)
- SNMP default community removal

These are all single operations that don't change in the lifecycle of the device. More can be added as needed. The rest can be managed via APi, but now we have our entry point

#### Using the script

** Reminder: start from a VERY LEAN AND CLEAN BASE **

- adapt it to your needs with the proper IP and settings
- log in, navigate to folder, copy paste, run ...

### Automating with terraform

Terraform has the best *provider* but it's still incomplete and the features I really need are missing, therefore I do some heavy lifting in terraform logic. This document makes use of the `terraform-routeros/routeros` provider

```
terraform {
  required_providers {
    routeros = {
      source = "terraform-routeros/routeros"
    }
  }
```

#### Workspaces

The workspaces feature of terraform allows the same code to be used for different routeros devices. Each device has its own workspace. Leveraging yaml (see further), we can give each device their own yaml configuration file based on the workspace.

After defining a s3 bucket (I use a minio container) in the code, init tofu

```
  backend "s3" {
    endpoints = {
      s3 = "http://your.server:9000"
    }
    shared_credentials_files    = [".aws/credentials"]
    profile                     = "minio"
    bucket                      = "mikrotik-tfstate"
    key                         = "terraform.tfstate"
    region                      = "main"
    skip_credentials_validation = true
    skip_metadata_api_check     = true
    skip_region_validation      = true
    use_path_style              = true
  }
```

```
tofu init -reconfigure
tofu workspace new mk-sw1.yours.lan
```

#### yaml

There are many ways to go about it but my approach uses the following syntax:

```
---
mk-sw1.yours.lan:
  hardware: RB5009UG+S+
  release: 7.20
  interfaces:
    sfp-sfpplus1: { role: ~ , bond: 1, pvid: 1,  tagged: [100, 110, 999] }
    ether1:       { role: ~ , bond: ~, pvid: 30, tagged: [] }
    ether2:       { role: ~ , bond: ~, pvid: 30, tagged: [] }
    ether3:       { role: ~ , bond: ~, pvid: 30, tagged: [] }
```

Which is loaded to be processed as such:

locals {
  # Load yaml by workspace name
  yaml_file = "${terraform.workspace}/config.yaml"

  # Decode the inventory YAML file
  config = yamldecode(file(local.yaml_file))
}


#### computing for LAG interfaces

This is the first "layer": Bonded interfaces (Link Aggregation) sit on top of the physical interface where needed. The new virtual interface can then be used as a normal interface to create VLANs etc.

#### computing for access port ingress

#### computing for trunk port ingress

#### computing for trunk port egress

#### computing for hybrid port

*WIP* not functional yet
