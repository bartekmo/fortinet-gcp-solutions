# Active-Passive HA FortiGate cluster in LB Sandwich
This template deploys 2 FortiGate instances in an Active-Passive HA cluster between two load balancers ("load balancer sandwich" pattern). LB Sandwich design enables use of multiple public IPs and provides faster, configurable failover times when compared to SDN-connector based. Due to Google Cloud Load Balancer limitations only UDP and TCP traffic is supported and tag-based GCP routes cannot be configured with ILB as next hop.

HA multi-zone deployments provide 99.99% Compute Engine SLA vs. 99.5-99.9% for single instances. See [Google Compute Engine SLA](https://cloud.google.com/compute/sla) for details.

Template file: [modules/fgcp-ha-ap-elbilb.jinja](../modules/fgcp-ha-ap-elbilb.jinja)
Schema file: [modules/fgcp-ha-ap-elbilb.jinja.schema](../modules/fgcp-ha-ap-elbilb.jinja.schema)

## Overview
FGCP protocol natively does not work in L3 overlay networks. For cloud deployments it must be configured to use unicast communication, which slightly limits its functionality (e.g. only Active-Passive between 2 peers is possible) and enforces use of dedicated management interface. In this template port3 is used as heartbeat and FGCP sync interface and port4 is used as dedicated management interface.

As cloud networks do not allow any network mechanisms below IP layer (e.g. gratuitous arp) usually used in HA scenarios, this template adds a pair of load balancers and configures health probes to detect currently active instance. Passive peer will be detected as unhealthy by the load balancers and will not receive any traffic. External load balancer is configured to forward all UDP and all TCP ports, while internal load balancer is used as a next hop in the custom route (note that despite using a single port in the ILB, it will route all UDP and TCP traffic).

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

## Diagram
As unicast FGCP clustering of FortiGate instances requires dedicated heartbeat and management NICs, 2 additional VPC Networks need to be created (or indicated in configuration file). This design features 4 separate VPCs for external, internal, heartbeat and management NICs. Both instances are deployed in separate zones indicated in **zones** property to enable GCP 99.99% SLA.

Additional resources deployed include:
- default route for the internal VPC Network pointing to the internal load balancer rule
- external IPs - by default one floating IP for incoming traffic from Internet and 2 management IPs, to add more external IPs simply list them in the **publicIPs** property
- Cloud Router/Cloud NAT service is used to allow outbound traffic from port1 of secondary FortiGate instance (e.g. for license activation)

![ELBILB Sandwich diagram](https://app.lucidchart.com/publicSegments/view/b1ee079a-3c64-4e75-acb7-a42e3b6f8982/image.png)

## Deployed Resources
- 2 FortiGate VM instances with 4 NICs each
- 2 VPC Networks: heartbeat and management (unless provided)
- External Load Balancer
    - External addresses
    - Target pool
    - Legacy HTTP Health Check
    - 2 Forwarding Rules for each IP (UDP and TCP)
- Internal Load Balancer
    - 2 unmanaged Instance Groups (one in each zone)
    - Backend Service
    - Internal Forwarding Rule (using ephemeral internal IP)
    - HTTP Health Check
    - route(s) via Forwarding Rule
- Cloud NAT

## Prerequisites and Requirements
You MUST create the external and protected VPC networks and subnets before using this template. External and protected subnets MUST be in the same region where VMs are deployed.

All VPC Networks already created before deployment and provided to the template using `networks.*.vpc` and `networks.*.subnet` properties, SHOULD have first 2 IP addresses available for FortiGate use. Addresses are assigned statically and it's the responsibility of administrator to make sure they do not overlap.

## Dependencies
This template uses [fgcp-ha-ap-sdn.jinja](fgcp-ha-ap-sdn.md) template and helpers in utils directory.

## How to deploy
For detailed instructions on how to deploy as well as list of available properties, check this [README](../README.md)

## See also
[Other FortiGate designs](../README.md)
