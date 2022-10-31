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
3. If you'll be running the Playbook manually with the CLI then set up your inventory file - see the [examples/inventory](examples/inventory) file for an example of how to do this.  If you plan on using this with Ansible Tower you can skip creating the inventory file.
4. Next, you'll need to create the directory structure for your configuration files.  This allows you to deploy DNS services to multiple hosts, that can each run multiple instances of the DNS service.  The directory structure is as follows:

```
site_configs/
├── {inventory_hostname} (the name of the Host in your Ansible Inventory)
│   ├── {dns_service_name} (the name of the DNS SystemD service)
│   │   ├── etc-config
│   │   │   ├── server.yml (GoZones DNS Server Configuration)
│   │   ├── vendor-config (optional)
│   │   │   ├── named.conf (BIND/named configuration to override defaults)
```

You can find examples of the `server.yml` and `named.conf` files in the [examples](examples) directory.

5. Once you have your directory structure created, you can run the Playbook to deploy the DNS services to your hosts.  If you're using the CLI, you can run the Playbook like this:

```bash=
ansible-playbook -i inventory deploy.yml
```

> If you're using Ansible Tower or the AAP2 Controller, continue to the next step.

## Setting Up Ansible Tower for GitOps-y goodness

Ideally as soon as you commit a change to the `main` branch of this repo, the changes will be synced to the actual services running on the hosts.  To do this, we'll use Ansible Tower to run the `deploy.yml` playbook.  Assuming you already have Ansible Tower or the AAP2 Controller installed and ready to go, you can follow these steps to set things up:

1. Create a **Credential(s)** for the systems that you will be connecting to.  You can use the `Machine` credential type for this.  You can also use the `Source Control` credential type for the Git repo that you'll be using for the configuration files.
2. Create an **Inventory** for the hosts that you'll be deploying the DNS services to.
3. Add the **Hosts** to the **Inventory**.  Make sure to add the needed `gozones_services` variables to the Hosts.  See the [examples/inventory](examples/inventory) file for an example of the `gozones_services` variable structure.
4. Create a **Project** for your fork of this Git repo.  I like to keep the names the same as the repo, so I named mine `lab-go-zones-dns`.  Make sure to check the `Update Revision on Launch` box.
5. Create a **Template** for the `deploy.yml` Playbook, add the Credentials you'll need to connect to the hosts.  Important: Make sure to check the `Enable Webhook` box.  This will allow you to trigger the Playbook from a GitHub/GitLab webhook.
6. Once the Template is created, you can get the `Webhook URL` and the `Webhook Key` from the **Details** page of the Template.  You'll need these to set up the Webhook in GitHub/GitLab.

Assuming you'll be using this with GitHub, you can follow these steps to set up the Webhook: https://docs.ansible.com/ansible-tower/latest/html/userguide/webhooks.html

> With this, now when you perform a `git push` to the repo to update your DNS zone files, the Playbook will be triggered and the changes will be automatically deployed to the hosts!

## Overriding the Default Bind Configuration

The default configuration for the BIND server has sane defaults to work well in a container for a lab environment out of the box.  However, you can override the default configuration by creating a file called `named.conf` in the `site_configs/{inventory_hostname}/{service_name}/vendor-config` directory.  This file will be copied into the container at `/opt/app-root/vendor/bind/named.conf` and will override the default configuration.

There is at least one line of crucial configuration that is required at the end of your `named.conf` file.  This line is:

```
# This line is required to be in your named.conf file - it loads the GoZones generated configuration for the zone files...or don't load it and just use this whole thing to start BIND/named your own way!
include "/opt/app-root/generated-conf/config/go-zones-bootstrap.conf";
```

The default file can be found in the [examples](examples) directory.

> You'll also need to set the `enable_vendor_overrides` to `true` when executing the `deploy.yml` Playbook