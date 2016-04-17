[![Circle CI](https://circleci.com/gh/daniel-rhoades/aws-security-groups-role.svg?style=svg&circle-token=9e73196b1138c36a3ec90938a2ebae505408ecc1)](https://circleci.com/gh/daniel-rhoades/aws-security-groups-role)

aws-security-group-role
=======================

Ansible role for simplifying the provisioning and decommissioning of EC2 Security Groups within an AWS account.

For more detailed on information on the creating VPCs with Ansible see the offical documentation for that module: http://docs.ansible.com/ansible/ec2_group_module.html.

Requirements
------------

Requires the latest Ansible EC2 support modules along with Boto.

You will also need to configure your Ansible environment for use with AWS, see http://docs.ansible.com/ansible/guide_aws.html.

Role Variables
--------------

Defaults:

* security_group_resource_tags: Defaults to the name of the security group;
* ec2_inbound_group_state: State of the security groups list as "inbound", defaults to `present`;
* ec2_internal_inbound_group_state: State of the security groups list as "internal inbound", defaults to `present`;
* ec2_outbound_group_state: State of the security groups list as "outbound", defaults to `present`.

Required variables:

* vpc_region: You must specify the region in which you created the VPC, e.g. eu-west-1;
* vpc_id: You must specify the VPC ID that you wish to create the security groups within, e.g. vpc-xxxxxxxx;
* ec2_group_inbound_sg: You must specify a list of inbound security groups to create, see example playbook section below for further information;
* ec2_group_internal_inbound_sg: You must specify a list of internal inbound security groups to create, see example playbook section below for further information;
* ec2_group_outbound_sg: You must specify a list of outbound security groups to create, see example playbook section below for further information.

Inbound security groups are expected to be those which you would apply to public facing services, e.g. a load balancer.  The internal inbound security groups are expected to be allocated to instanced behind a load balancer or internal to the VPC.  Outbound security groups are the ports which services can call upon from inside the network.

Outputs:

* ec2_group_inbound_sg: The AWS EC2 Group object created as a result of running the `ec2_group_module` with the supplied "inbound" variables;
* ec2_group_internal_inbound_sg: The AWS EC2 Group object created as a result of running the `ec2_group_module` with the supplied "internal inbound" variables;
* ec2_group_outbound_sg: The AWS EC2 Group object created as a result of running the `ec2_group_module` with the supplied "outbound" variables.

The role processes the security groups in the order of:

* inbound;
* internal inbound;
* outbound.

This means you can reference security groups that will be created within the "inbound" list in the "internal inbound" list, this is especially useful when you want to route traffic only between the inbound and internal inbound groups, e.g. like a load balancer and web server.

Dependencies
------------

No dependencies on other roles, but an VPC must already exist or be created before applying this role.

Example Playbook
----------------

Before using this role you will need to install the role, the simplist way to do this is: `ansible-galaxy install daniel-rhoades.aws-security-group-role`.

The example playbook below ensures an EC2 Security Group is provisioned in AWS as specified, e.g. if one already matches the role does nothing, otherwise it gets created.  For completeness the examples also create a VPC into which the EC2 Security Groups would reside using the role:  `daniel-rhoades.aws-vpc`.

```
- name: My System | Provision all required infrastructure
  hosts: localhost
  connection: local
  gather_facts: no
  vars:
    my_vpc_name: "my_example_vpc"
    my_vpc_region: "eu-west-1"
    my_vpc_cidr: "172.40.0.0/16"
    everywhere_cidr: "0.0.0.0/0"

    # Subnets within the VPC
    my_vpc_subnets:
      - cidr: "172.40.10.0/24"
        az: "{{ my_vpc_region }}a"

      - cidr: "172.40.20.0/24"
        az: "{{ my_vpc_region }}b"

    # Allow the subnets to route to the outside world
    my_public_subnet_routes:
      - subnets:
          - "{{ my_vpc_subnets[0].cidr }}"
          - "{{ my_vpc_subnets[1].cidr }}"
        routes:
          - dest: "{{ everywhere_cidr }}"
            gw: igw

    # Inbound security groups, e.g. public facing services like a load balancer
    my_inbound_security_groups:
      - sg_name: inbound-web
        sg_description: allow http and https access (public)
        sg_rules:
          - proto: tcp
            from_port: 80
            to_port: 80
            cidr_ip: "{{ everywhere_cidr }}"

          - proto: tcp
            from_port: 443
            to_port: 443
            cidr_ip: "{{ everywhere_cidr }}"

      # Only allow SSH access from within the VPC, to access any services within the VPC you would need to create a
      # temporary bastion host
      - sg_name: inbound-ssh
        sg_description: allow ssh access
        sg_rules:
         - proto: tcp
           from_port: 22
           to_port: 22
           cidr_ip: "{{ my_vpc_cidr }}"

    # Internal inbound security groups, e.g. services which should not be directly accessed outside the VPC, such as
    # the web servers behind the load balancer
    my_internal_inbound_security_groups:
      - sg_name: inbound-web-internal
        sg_description: allow http and https access (from load balancer only)
        sg_rules:
          - proto: tcp
            from_port: 80
            to_port: 80
            group_id: "{{ ec2_group_inbound_sg.results[0].group_id }}"

    # Outbound rules, e.g. what services can the web servers access by themselves
    my_outbound_security_groups:
      - sg_name: outbound-all
        sg_description: allows outbound traffic to any IP address
        sg_rules:
          - proto: all
            cidr_ip: "{{ everywhere_cidr }}"
  roles:
    # Provision networking
    - {
        role: daniel-rhoades.aws-vpc,
        vpc_name: "{{ my_vpc_name }}",
        vpc_region: "{{ my_vpc_region }}",
        vpc_cidr_block: "{{ my_vpc_cidr }}",
        vpc_subnets: "{{ my_vpc_subnets }}",
        public_subnet_routes: "{{ my_public_subnet_routes }}"
      }

    # Provision security groups
    - {
        role: daniel-rhoades.aws-security-groups,
        vpc_region: "{{ my_vpc_region }}",
        vpc_id: "{{ vpc.vpc_id }}",
        ec2_group_inbound_sg: "{{ my_inbound_security_groups }}",
        ec2_group_internal_inbound_sg: "{{ my_internal_inbound_security_groups }}",
        ec2_group_outbound_sg: "{{ my_outbound_security_groups }}"
      }
```

To decommission the groups:

```
- name: My System | Decommission all required infrastructure
  hosts: localhost
  connection: local
  gather_facts: no
  vars:
    my_vpc_name: "my_example_vpc"
    my_vpc_region: "eu-west-1"
    my_vpc_cidr: "172.40.0.0/16"
    everywhere_cidr: "0.0.0.0/0"

    # Subnets within the VPC
    my_vpc_subnets:
      - cidr: "172.40.10.0/24"
        az: "{{ my_vpc_region }}a"

      - cidr: "172.40.20.0/24"
        az: "{{ my_vpc_region }}b"

    # Allow the subnets to route to the outside world
    my_public_subnet_routes:
      - subnets:
          - "{{ my_vpc_subnets[0].cidr }}"
          - "{{ my_vpc_subnets[1].cidr }}"
        routes:
          - dest: "{{ everywhere_cidr }}"
            gw: igw

    # Inbound security groups, e.g. public facing services like a load balancer
    my_inbound_security_groups:
      - sg_name: inbound-web
        sg_description: allow http and https access (public)
        sg_rules:
          - proto: tcp
            from_port: 80
            to_port: 80
            cidr_ip: "{{ everywhere_cidr }}"

          - proto: tcp
            from_port: 443
            to_port: 443
            cidr_ip: "{{ everywhere_cidr }}"

      # Only allow SSH access from within the VPC, to access any services within the VPC you would need to create a
      # temporary bastion host
      - sg_name: inbound-ssh
        sg_description: allow ssh access
        sg_rules:
         - proto: tcp
           from_port: 22
           to_port: 22
           cidr_ip: "{{ my_vpc_cidr }}"

    # Internal inbound security groups, e.g. services which should not be directly accessed outside the VPC, such as
    # the web servers behind the load balancer
    my_internal_inbound_security_groups:
      - sg_name: inbound-web-internal
        sg_description: allow http and https access (from load balancer only)
        sg_rules:
          - proto: tcp
            from_port: 80
            to_port: 80
            group_id: "{{ ec2_group_inbound_sg.results[0].group_id }}"

    # Outbound rules, e.g. what services can the web servers access by themselves
    my_outbound_security_groups:
      - sg_name: outbound-all
        sg_description: allows outbound traffic to any IP address
        sg_rules:
          - proto: all
            cidr_ip: "{{ everywhere_cidr }}"
  roles:
    # Provision networking
    - {
        role: daniel-rhoades.aws-vpc,
        vpc_name: "{{ my_vpc_name }}",
        vpc_region: "{{ my_vpc_region }}",
        vpc_cidr_block: "{{ my_vpc_cidr }}",
        vpc_subnets: "{{ my_vpc_subnets }}",
        public_subnet_routes: "{{ my_public_subnet_routes }}"
      }

    # Decommission security groups
    - {
        role: daniel-rhoades.aws-security-groups,
        vpc_region: "{{ my_vpc_region }}",
        vpc_id: "{{ vpc.vpc_id }}",
        ec2_group_inbound_sg: "{{ my_inbound_security_groups }}",
        ec2_group_internal_inbound_sg: "{{ my_internal_inbound_security_groups }}",
        ec2_group_outbound_sg: "{{ my_outbound_security_groups }}",

        ec2_inbound_group_state: "absent",
        ec2_internal_inbound_group_state: "absent",
        ec2_outbound_group_state: "absent"
      }
```

License
-------

MIT

Author Information
------------------

Daniel Rhoades (https://github.com/daniel-rhoades)
