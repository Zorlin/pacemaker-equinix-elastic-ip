# Pacemaker Elastic IP on Equinix Metal
This project adds Equinix Metal Elastic IP address support to Pacemaker.

Demo:

[![asciicast](https://asciinema.org/a/X5bkycTrWOCod7bljEaRsN5Qv.svg)](https://asciinema.org/a/X5bkycTrWOCod7bljEaRsN5Qv)

## Getting started
For each machine you want to participate in Pacemaker:

* Install jq and crudini

`apt install -y jq crudini`

* Create the folder /usr/lib/ocf/resource.d/equinix/

* Copy equinix-elastic-ip and equinix-elastic-ip-public to /usr/lib/ocf/resource.d/equinix/

Set up a file /etc/ansible/facts.d/uuid.fact as such:

```
equinix_uuid=this-is-not-a-real-uuid
equinix_project=this-is-not-a-real-project
equinix_token=this-is-not-a-real-token
```

where equinix_uuid is the UUID of the machine in question (this is necessary to be able to move the IP to it)

equinix_project is the UUID of the project that machine lives in (to be able to find IP assignments)

and equinix_token is a valid token with read/write scope for your Equinix Metal account.

## Using the resource agent

This resource is best used *in combination* with the `ocf:heartbeat:IPaddr2` resource agent. 

This agent will handle moving the elastic IP between machines in the API, while the `ocf:heartbeat:IPaddr2` 
will handle adding the IP itself to an interface on whichever machine you failover to.

### Public Elastic IP
Create an `ocf:heartbeat:IPaddr2` resource (replace ip with your chosen IP, bond0 with the interface it should live on):

`pcs resource create virtual_ip_public ocf:heartbeat:IPaddr2 ip=198.70.197.254 cidr_netmask=32 nic=bond0`

Create the Equinix Metal Elastic IP resource, replacing `198.70.197.254` with your chosen IP.

`pcs resource create elastic_ip_metal_public ocf:equinix:equinix-elastic-ip-public ip=198.70.197.254`

Create a colocation rule that says they must always "live together"

`pcs constraint colocation add virtual_ip_public with elastic_ip_metal_public INFINITY`

### Private Elastic IP
Create an `ocf:heartbeat:IPaddr2` resource (replace ip with your chosen IP, bond0 with the interface it should live on):

`pcs resource create virtual_ip ocf:heartbeat:IPaddr2 ip=10.20.30.254 cidr_netmask=32 nic=bond0`

Create the Equinix Metal Elastic IP resource, replacing `10.20.30.254` with your chosen IP and "10.20.30.128" with your chosen subnet.

`pcs resource create elastic_ip_metal ocf:equinix:equinix-elastic-ip ip=10.20.30.254 subnet=10.20.30.128`

Create a colocation rule that says they must always "live together"

`pcs constraint colocation add virtual_ip with elastic_ip_metal INFINITY`

## Operation
Once you've installed the resource agents, added your configuration and created all the required resources, run `pcs status`. You should see something like this:

```
root@db-staging-lb01:~# pcs status
Cluster name: db-loadbalancer
Cluster Summary:
  * Stack: corosync
  * Current DC: db-staging-lb03 (version 2.1.2-ada5c3b36e2) - partition with quorum
  * Last updated: Mon Nov 28 05:26:40 2022
  * Last change:  Mon Nov 28 04:54:58 2022 by hacluster via crmd on db-staging-lb01
  * 3 nodes configured
  * 5 resource instances configured

Node List:
  * Online: [ db-staging-lb01 db-staging-lb02 db-staging-lb03 ]

Full List of Resources:
  * virtual_ip  (ocf:heartbeat:IPaddr2):         Started db-staging-lb01
  * Clone Set: loadbalancer-clone [loadbalancer]:
    * Started: [ db-staging-lb01 db-staging-lb02 db-staging-lb03 ]
  * elastic_ip_metal    (ocf:equinix:equinix-floating-ip):       Started db-staging-lb01

Daemon Status:
  corosync: active/enabled
  pacemaker: active/enabled
  pcsd: active/enabled
```

If you see a virtual_ip resource, and an elastic_ip_metal resource (or the public equivalents thereof) started on the same machine, and traffic is flowing, congratulations, you've set up a HA cluster on Equinix Metal! Try rebooting the active machine while pinging the elastic IP and see what happens - failover should happen within 4-12 seconds. Enjoy!

## Credits and License

Copyright (c) 2022 Benjamin Arntzen & Protocol Labs, licensed under the GNU General Public License v2.

Based on code of Markus Guertler and Mathieu GRZYBEK.
