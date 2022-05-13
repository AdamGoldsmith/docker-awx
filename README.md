# NOTICE

This README is taken from another one of my repos and is therefore incorrect - needs big overhaul when I find time!

# AWX on Docker deployment using Ansible & Vagrant

Collection of playbooks and roles to prepare and deploy hosts to run AWX in a docker-compose development/test environment. A playbook is supplied to create a Vagrantfile from the Ansible inventory for creating an infrastructure using either the VirtualBox or libvirt hypervisor.

## Requirements

* __community.docker__ collection (install with `ansible-galaxy collection install community.docker`)
* __vagrant__ required for deploying a VM to host Docker for running AWX container services via docker-compose
* __VirtualBox__ required for deploying Vbox VM
* __libvirt__ required for deploying libvirt VM

## TL;DR

From root directory of repo:

1. Depending on your flavour of hypervisor (defaults to VirtualBox), edit `ansible.cfg` and update `inventory = ./inv.d/vbox` line (choices are `vbox`, `libvirt`)
1. `ansible-playbook playbooks/vagrant/prepare_vagrant.yml -e "vagrant_script_check=no"`
1. `cd vagrant; vagrant up; cd ..`
1. `./scripts/prepare_ansible_targets.sh`
1. `ansible-playbook playbooks/site.yml`
1. Point browser at https://10.20.88.11:8043 to access [AWX](https://github.com/ansible/awx)

To stop the services cleanly from the Ansible controller:

1. `ansible-playbook playbooks/site.yml --tags awx_stop`

## Overview of tasks

When run with default options, the following tasks will be applied in this order. Each main task can be run by supplying `--tags` - see [here](#running-the-deployment) for details.

1. Fixing references to unavailable repos for CentOS 8 hosts (see https://www.centos.org/centos-linux-eol/)
1. Preparing the hosts for Docker installation (Managed by the roles `ansible_role_user_prepare` & `ansible_role_dir_prepare`)
    1. Creating groups & users
    1. Configuring sudo access
    1. Installing LVM software
    1. Creating Volume Groups, Logical Volumes, Mountpoints & Filesystems
    1. Applying directory structure with correct ownership & permissions
1. Install Docker software packages (Managed by the role `ansible_role_docker_install`)
    1. Removing unused redundant Docker software packages
    1. Add Docker yum gpgkey & repository when not using private repositories
    1. Installing Docker software packages
    1. Start & enable Docker service
    1. Configure docker daemon properties
1. Install Ansible software
1. Install AWX software
    1. Install AWX dependencies
    1. Checkout AWX git repository
    1. Build the docker-compose development environment
    1. Start the AWX services
    1. Create the AWX UI

## Supported hypervisor host platofrms

| Platform   | Version |
|------------|---------|
| Linux Mint | 20.3    |

## Supported hypervisor guest platforms

| Platform | Version |
|----------|---------|
| CentOS   | 8       |

## Requirements

* Ansible 2.9+ (tested with 2.9.8, 2.10.0 & 2.12.1)
* Python's docker module (pip - `docker`)
* Vagrant (__*optional*__) (tested with 2.2.6 & 2.2.19)
* Python3 (__*optional*__) for creating a virtual env using supplied utility script (tested with 3.8.10)
* libvirt vagrant plugin (vagrant plugin install vagrant-libvirt)

## Inventory directories

There are two inventories that live under the main `inv.d` directory - `vbox` & `libvirt`. The default inventory is `vbox` as configued in `ansible.cfg` in the repo root directory (`inventory = ./inv.d/vbox`)

The following example shows the supplied inventory in `inv.d/vbox/inventory`

```ini
localhost ansible_host=127.0.0.1 ansible_connection=local

[awx]
awx-1 ansible_host=192.168.56.11
```

## Group & Host vars

In addition to the group & host variables required for each role (see their associated READMEs), the following group & host variables will need setting:

`inv.d/group_vars/awx/awx.yml`
```yaml
awx_git_version: 20.0.0
awx_git_dest: ~/awx
awx_inventory_data:
  pg_password: pg_password
  broadcast_websocket_secret: web_secret
  secret_key: secret_key
```
`inv.d/ds/group_vars/all/vars.yml`
```yaml
ansible_user: ansible                       # Ansible user on target host
verbosity_level: 1                          # Verbosity level when to display debug output (0 = always, 1 = -v, 2 = -vv, etc)
```

## Optional use of Vagrant

This repo supplies an Ansible playbook that locally creates a Vagrantfile and target preparation script using the contents of the host inventories. These can be used to provision and prepare a complete [VirtualBox](https://www.virtualbox.org) or [libvirt](https://libvirt.org/) test infrastructure using [Vagrant](https://www.vagrantup.com) ready for the Docker swarm deployment. Although it is designed to work out of the box by design, please ensure that the following vagrant variables are configured correctly.

`inv.d/host_vars/localhost.yml`
```yaml
vagrant_box_version: "centos/8"                           # Flavour and version of Vagran Box
vagrant_guest_additions: False                            # Boolean for installing guest additions software
vagrant_domain: ""                                        # Domain name for vagrant hosts
vagrant_script_check: True                                # Defaults the prepare_ansible_targets.sh script to run in check mode
vagrant_connect_user: vagrant                             # VM user created by vagrant
vagrant_ansible_user: ansible                             # User to create in VM by prepare_ansible_targets.sh script
vagrant_ansible_user_uid: 9999                            # UID for ansible user
vagrant_connect_user_privkey_file: ~/.ssh/id_rsa          # Path to private SSH key used to connect to VM
vagrant_ansible_user_pubkey_file: ~/.ssh/id_rsa.pub       # Path to publc SSH key pair file
```

1. Running the following Ansible playbook will
    1. Create the `vagrant` & `scripts` directories
    1. Create the `Vagrantfile` for the selected hypervisor
    1. Create the `prepare_ansible_targets.sh` script
    1. Copy the SSH public key into the `vagrant` directory

```
ansible-playbook playbooks/vagrant/prepare_vagrant.yml
```

__Note:__ *To override the default enabled script check mode, run with: `-e "vagrant_script_check=no"`*

2. Running the following command will bring the necessary infrastructure online

```
cd vagrant; vagrant up; cd ..
```

3. Running the `prepare_ansible_targets.sh` script will use the initial vagrant user to
    1. Create the ansible user on the target hosts
    1. Update the ansible user's `authorized_keys` file on the target hosts with the specified public SSH key
    1. Add the ansible user to sudo with passwordless access

```
./scripts/prepare_ansible_targets.sh
```

At this point, a complete test infrastructure will now be provisioned, ready to be configured!

## AWX deployment

The Docker services are deployed using `docker-compose` files located in the AWX repo - see their README for details.


## Running the deployment

1. To perform a full deployment of all components to the default environment
    ```
    ansible-playbook playbooks/site.yml
    ```
    __Note:__ *For additional debug output, set the verbosity level with: `-e "verbosity_level=0"`*


>__Note__: There is no option to destroy this configuration. You are advised to create a new environment and re-deploy this configuration

This can easily be achieved when using vagrant:

```
cd vagrant; vagrant destroy --force; vagrant up; cd ..
```

## Known Issues

1. Inactive libvirt networks

    It was noticed that when using libvirt provider for vagrant, the network was inactive at system bootup. There is an option to start it at system startup in the "Virtual Networks" tab of the "Connection Details" window in "Virtual Machine Manager"

## Author Information

Adam Goldsmith
