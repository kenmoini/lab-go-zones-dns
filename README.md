# Lab GoZones DNS Configuration and Automation

This repository contains the configuration and automation to deploy GoZones DNS via containers as a SystemD Service running on a number of hosts.  You can even deploy multiple instances of the GoZones DNS server on a single host.

[GoZones DNS](https://github.com/kenmoini/go-zones) is simply a Golang app that takes DNS defined in YAML files and generate BIND/named compliant configuration and zone files, and optionally start the server with them.

In this configured form takes in a YAML definition of the DNS networks, ACLs, policies, zones, and records and then generates the configuration files for BIND - then starts BIND with those generated configuration files.  There are sane defaults that make this work out of the box, however, you can override the defaults with your own configuration if you choose to do so.

The configuration as represented in this repository deploys two container services (dns-core-1, dns-core-2) to a single host (raza.kemo.labs) for redundant container services that have bridged IPs on the LAN.

## Host Requirements

- Podman

The requests/limits of the containers can be set otherwise.  The default is 256/512MB of RAM and 0.5/1 CPU requests/limits.

## Using this repository

First off, the configuration in this repo is mine which means it's not likely to be usable by you - and you don't have edit access to this repo more than likely.  So, you'll need to fork this repo and then clone your forked repo to your local machine.

1. [Fork this repo](https://github.com/kenmoini/lab-go-zones-dns/fork)
2. Clone your forked repo to your local machine - `git clone https://github.com/YOUR_USERNAME/lab-go-zones-dns.git`.
3. If you'll be running the Playbook manually with the CLI then set up your inventory file - see the [example.inventory](example.inventory) file for an example of how to do this.  If you plan on using this with Ansible Tower you can skip creating the inventory file.
4. 

## Setting Up Ansible Tower for GitOps-y goodness

Ideally as soon as you commit a change to the `main` branch of this repo, the changes will be synced to the actual services running on the hosts.  To do this, we'll use Ansible Tower to run the `deploy.yml` playbook.

> TODO

## Overriding the Default Bind Configuration

The default configuration for the BIND server has sane defaults to work well in a container for a lab environment out of the box.  However, you can override the default configuration by creating a file called `named.conf` in the `site_configs/{inventory_hostname}/{service_name}/vendor-config` directory.  This file will be copied into the container at `/opt/app-root/vendor/bind/named.conf` and will override the default configuration.

There is at least one line of crucial configuration that is required at the end of your `named.conf` file.  This line is:

```
# This line is required to be in your named.conf file - it loads the GoZones generated configuration for the zone files...or don't load it and just use this whole thing to start BIND/named your own way!
include "/opt/app-root/generated-conf/config/go-zones-bootstrap.conf";
```

The default file can be found here: [named.conf](https://github.com/kenmoini/go-zones/blob/main/container_root/opt/app-root/vendor/bind/named.conf)