# Ansible playbooks for RDQM

This directory contains two playbooks relating to RDQM:

1. an ansible playbook `rdqm.yml` to configure three systems so that they are ready to become an RDQM HA Group

2. an ansible playbook `createqm.yml` to create a sample queue manager

The systems must be running either RHEL 7, RHEL 8 or RHEL 9.

You will need to update a set of variables files in the group_vars folder before using the rdqm.yml playbook to configure RHEL systems.

Before running either playbook, you will need to update the hosts file to reflect the systems you wish to configure, both RDQM and DR.

There are two groups of hosts in the hosts file:

1. rdqm
2. dr

There must be exactly three hosts in the rdqm group and optionally exactly three hosts in the dr group for the rdqm playbook to run. For each host, at least one IP address must be specified:

1. ansible_host must be the address which ansible can use to connect to the host
2. rdqm_ha_gateway will be used as the internet gateway for static route communication between the RDQM hosts
3. [optional] rdqm_ha_replication must be the address of the host that is to be used for HA replication. If omitted, ansible_host will be used for replication
4. [optional] rdqm_dr_replication must be the address of the host that is to be used for DR replication. If omitted, ansible_host or rdqm_ha_replication will be used for replication 

Currently, this Playbook allows for two different IP addresses to be specified for each host. In the future, I would like to support specifying one or two additional RDQM IP addresses to be specified so that all of the options for the rdqm.ini file can be supported, but for the moment the generated rdqm.ini file will only have one IP address per host. If you want to specify more, you can edit the generated rdqm.ini file before configuring the RDQM HA Group.

If you want to use the same IP address for both ansible and RDQM, at the moment the address has to be specified twice. I hope to remove the requirement for this in the future.

## group_vars/all.yaml

### mq_release

The MQ release you are installing.
The value of this variable is used to create a directory to hold the unpacked image.

### base_dir

The base directory in which installation files are stored.

### mq_server_image

The full path to the file containing the MQ server product to be installed.
This file is copied to each of the RDQM nodes and unpacked.

### mqm_home

Home directory for the mqm user.

### mqm_ssh

Boolean for whether or not the mqm user to be allowed passwordless ssh.

### gpg_check

Indicate whether or not to require checking the signature of rpm packages with a gpg signing key.

## group_vars/mqm-user.yaml

### mqm_configure

Boolean check whether to configure mqm user or to let the installation do it for you.

### mqm_gid

The group id to be used for the mqm group. The mqm group is created explicitly before MQ is installed.

### mqm_uid

The user id to be used for the mqm user. The mqm user is created explicitly before MQ is installed, to guarantee that the same value is used on all hosts.

### mqm_password

The encrypted password used temporarily while configuring passwordless ssh for the mqm user.

### mqm_ssh_key

SSH Key for the mqm user.

## group_vars/drbd.yaml

### DRBD_device

The device which should be used to create a volume group for DRBD/RDQM.

## group_vars/mq.yaml

### mq_gpg_location

Location of gpg signing key for IBM MQ.

## group_vars/rdqm.yaml

### rdqm_admin_user

The user for the rdqmadmin account on the RDQM nodes.

### rdqm_admin_password

The encrypted password for the rdqmadmin account on the RDQM nodes.

### ssh_key_type

The type of ssh key for the mqm user.

## RDQM Playbook

Once the `hosts` file has been updated and any variables, the RDQM playbook can be run with the ansible.builtin.command:

```bash
ansible-playbook -i hosts rdqm.yml
```

## Accept MQ license

When the playbook is run on each of the RDQM hosts, the MQ license is accepted as follows:

On each of the rdqm hosts, as root, run `/opt/mqm/bin/mqlicense -accept` which should produce:

```bash
5724-H72 (C) Copyright IBM Corp. 1994, 2021.
License agreement accepted. Run the dspmqlic command to view the MQ
license agreement.
```

## Configure RDQM HA Group

If the mqm user does not have passwordless ssh, run the following command on all nodes. If the user does have passwordless ssh, only run on the primary node of each HA group: 

```bash
su - rdqmadmin
rdqmadm -c
```

which should produce something like:

```bash
Configuring the replicated data subsystem.
The replicated data subsystem has been configured on this node.
The replicated data subsystem has been configured on
'colgrave-vsi-rdqm-eu-gb-2'.
The replicated data subsystem has been configured on
'colgrave-vsi-rdqm-eu-gb-3'.
The replicated data subsystem configuration is complete.
Command '/opt/mqm/bin/rdqmadm' run with sudo.
```

You can check the status by running `rdqmstatus -n` which should produce something like:

```bash
Node colgrave-vsi-rdqm-eu-gb-1 is online
Node colgrave-vsi-rdqm-eu-gb-2 is online
Node colgrave-vsi-rdqm-eu-gb-3 is online
```

## Create RDQM HA Queue Manager

As rdqmadmin on the first rdqm node, run `crtmqm -p 1414 -sx QM1` which should produce something like:

```bash
Creating replicated data queue manager configuration.
Secondary queue manager created on 'colgrave-vsi-rdqm-eu-gb-2'.
Secondary queue manager created on 'colgrave-vsi-rdqm-eu-gb-3'.
IBM MQ queue manager created.
Directory '/var/mqm/vols/qm1/qmgr/qm1' created.
The queue manager is associated with installation 'Installation1'.
Creating or replacing default objects for queue manager 'QM1'.
Default objects statistics : 84 created. 0 replaced. 0 failed.
Completing setup.
Setup completed.
Enabling replicated data queue manager.
Replicated data queue manager enabled.
Command '/opt/mqm/bin/crtmqm' run with sudo.
```

You can check the status of this queue manager by running `rdqmstatus -m QM1` which should produce something like:

```bash
Node:                                   colgrave-vsi-rdqm-eu-gb-1
Queue manager status:                   Running
CPU:                                    0.00%
Memory:                                 136MB
Queue manager file system:              58MB used, 2.9GB allocated [2%]
HA role:                                Primary
HA status:                              Normal
HA control:                             Enabled
HA current location:                    This node
HA preferred location:                  This node
HA blocked location:                    None
HA floating IP interface:               None
HA floating IP address:                 None

Node:                                   colgrave-vsi-rdqm-eu-gb-2
HA status:                              Normal

Node:                                   colgrave-vsi-rdqm-eu-gb-3
HA status:                              Normal
```

## Testing

The approach I usually use to test a new deployment is described [here](Testing.md)
