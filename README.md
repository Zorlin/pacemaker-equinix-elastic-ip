# pacemaker-equinix-resource-agents

This project adds Equinix Metal Elastic IP address support to Pacemaker.

## Configuration
Global configuration:

* Edit equinix-elastic-ip and replace EQUINIX_PROJECT= as appropriate

* If you need to use public IP addresses instead of private ones, search and replace "private_ipv4" with "public_ipv4" (or remove that "type" specification if you need to use both)

For each machine you want to participate in Pacemaker:

* Install jq and crudini

`apt install -y jq crudini`

* Create the folder /usr/lib/ocf/resource.d/equinix/

* Copy equinix-elastic-ip to /usr/lib/ocf/resource.d/equinix/equinix-elastic-ip 

Set up a file /etc/ansible/facts.d/uuid.fact as such:

```
equinix_uuid=this-is-not-a-real-uuid
equinix_token=this-is-not-a-real-token
```

where equinix_uuid is the UUID of the machine in question (this is necessary to be able to move the IP to it)

and equinix_token is a valid token with read/write scope for your Equinix Metal account.

## Using the resource agent

This resource is best used *in combination* with the ocf:heartbeat:IPaddr2 resource agent. 

This agent will handle moving the elastic IP between machines in the API, while the ocf:heartbeat:IPaddr2 
will handle adding the IP itself to an interface on whichever machine you failover to.

Create an ocf:heartbeat:IPaddr2 resource (replace ip with your needed IP, bond0 with the interface it should live on):

`pcs resource create virtual_ip ocf:heartbeat:IPaddr2 ip=10.70.197.254 cidr_netmask=32 nic=bond0`

Create the Equinix Metal Elastic IP resource (replace ip with your needed IP)

`pcs resource create elastic_ip_metal ocf:equinix:equinix-elastic-ip ip=10.70.197.254

Create a colocation rule that says they must always "live together"

`pcs constraint colocation add virtual_ip with elastic_ip_metal INFINITY`
