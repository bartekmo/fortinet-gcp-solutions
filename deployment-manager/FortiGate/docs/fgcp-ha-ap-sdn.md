# Active-Passive HA FortiGate Pair with Fabric (SDN) Connector Failover
This template deploys an Active-Passive HA cluster of 2 FortiGate instances together with the required cloud resources. The cluster is preconfigured with the FGCP configuration synchronization, GCP Fabric Connector, and proper HA configuration for external IP and route failover.

HA multi-zone deployments provide 99.99% Compute Engine SLA vs. 99.5-99.9% for single instances. See [Google Compute Engine SLA](https://cloud.google.com/compute/sla) for details.

This architecture is suitable for deployments where only a single Public IP is required.

Template file: [modules/fgcp-ha-ap-sdn.jinja](../modules/fgcp-ha-ap-sdn.jinja)
Schema file: [modules/fgcp-ha-ap-sdn.jinja.schema](../modules/fgcp-ha-ap-sdn.jinja.schema)

## Overview
FGCP protocol natively does not work in L3 overlay networks. For cloud deployments it must be configured to use unicast communication, which slightly limits its functionality (e.g. only Active-Passive between 2 peers is possible) and enforces use of dedicated management interface. In this template port3 is used as heartbeat and FGCP sync interface and port4 is used as dedicated management interface.

As cloud networks do not allow any network mechanisms below IP layer (e.g. gratuitous arp) usually used in HA scenarios, this template configures FortiGates to interact with Google Cloud Compute API upon failover event. SDN connector will re-attach the public IP to port1 of newly active instance and rewrite the route on internal network to point to the new instance's port2 IP address.

## Active-Passive HA Design Options Comparison
Fortinet recommends two base building blocks when designing GCP network with an Active-Passive FortiGate cluster:
1. [A-P HA with SDN Connector](fgcp-ha-ap-sdn.md)
2. [A-P HA in LB Sandwich](fgcp-ha-ap-elbilb.md)

Make sure you understand differences between them and choose your architecture properly. Also remember that templates and designs provided here are the base building blocks and you can modify or mix them to match your individual use case.

| Feature | with SDN Connector | in LB Sandwich |
| --------|--------------------|----------------|
| Failover time | >40 secs | ~10 secs |
| Protocols supported | UDP, TCP, ICMP, ESP, AH, SCTP | UDP, TCP |
| Max. external addresses | 1 (per external NIC) | unlimited |
| Tag-based internal GCP routes| supported | not supported |

## Design
As unicast FGCP clustering of FortiGate instances requires dedicated heartbeat and management NICs, 2 additional VPC Networks need to be created (or indicated in configuration file). This design features 4 separate VPCs for external, internal, heartbeat and management NICs. Both instances are deployed in separate zones indicated in **zones** property to enable GCP 99.99% SLA.

Additional resources deployed include:
- default route for the internal VPC Network pointing to the internal IP of primary FortiGate - this route will be re-written using Fabric Connector during failover to direct outbound traffic from internal network via the active instance. More granular routing can be deployed by template using the routes property
- 3 external IPs - one floating IP for incoming traffic from Internet and 2 management IPs
- Cloud Router/Cloud NAT service is used to allow outbound traffic from port1 of secondary FortiGate instance (e.g. for license activation)

![A-P HA Diagram](https://www.lucidchart.com/publicSegments/view/9fb2009b-32fa-4404-9009-4eb4529c988c/image.png)

## Prerequisites
1. Two VPC Networks created for external and protected roles
1. Two empty subnets created in the external and protected VPCs.

## Failover automation
Deployed FortiGates integrate with GCP fabric using an SDN Connector. Upon failover 2 actions are performed:
- named route is switched to the IP of the now active node
- named external IP is re-assigned to the now active node

## Dependencies
This template uses [singlevm.jinja](singlevm.md) template and helpers in utils directory.

## How to deploy
See [README](../README.md) for details on how to deploy this template and on available parameters.

See [ha-ap-sdn.yaml](../examples/ha-ap-sdn.yaml) in [examples](../examples) directory for a basic configuration file to start with.

## Post-deployment Steps
After your firewalls are deployed, connect to the primary instance and change the default password. The initial password is set to the primary instance ID.

## See also
[Other FortiGate deployments](./README.md)
