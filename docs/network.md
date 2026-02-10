## Network Design

The environment is deployed in a custom VPC with separate public and private subnets.

- Public subnet is used only for internet-facing resources.
- Private subnet hosts EC2 instances without public IPs.
- Internet access is controlled through route tables and gateways.
