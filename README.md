# NOTICE

This README is taken from another one of my repos and is therefore incorrect - needs big overhaul when I find time!

# AWX on Docker deployment using Ansible & Vagrant

Collection of playbooks and roles to prepare and deploy hosts to run AWX in a docker-compose development/test environment. A playbook is supplied to create a Vagrantfile from the Ansible inventory for creating an infrastructure using the VirtualBox hypervisor.

## Requirements

* __community.docker__ collection (install with `ansible-galaxy collection install community.docker`)

## TL;DR

From root directory of repo:

1. `./scripts/ansible_venv.sh && source ~/venvs/ansible_ds/bin/activate` (__Optional__ step that creates & sources an ansible-enabled python virtual environment [__Requires python3__])
1. `ansible-playbook playbooks/vagrant/prepare_vagrant.yml -e "vagrant_script_check=no"`
1. `cd vagrant; vagrant up; cd ..`
1. `./scripts/prepare_ansible_targets.sh`
1. `ansible-playbook playbooks/site.yml`
1. Point browser at https://10.20.88.11:8043 to access [AWX](https://github.com/ansible/awx)

To stop the services cleanly from the Ansible controller:

1. ansible awx -a "docker-compose -f ~/awx/tools/docker-compose/_sources/docker-compose.yml stop"

## Overview of tasks

When run with default options, the following tasks will be applied in this order. Each main task can be run by supplying `--tags` - see [here](#running-the-deployment) for details.

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

## Supported platforms

| Platform | Version |
|----------|---------|
| CentOS   | 8       |

## Requirements

* Ansible 2.9+ (tested with 2.9.8 & 2.10.0)
* Python's docker module (pip - `docker`)
* Vagrant (__*optional*__) (tested with 2.2.6)
* Python3 (__*optional*__) (for creating a virtual env using supplied utility script)

## Inventory directories

The inventory lives under the main `inv.d` directory.

The following example shows the supplied inventory in `inv.d/inventory`

```ini
localhost ansible_host=127.0.0.1 ansible_connection=local

[awx]
awx-1 ansible_host=10.20.88.11
```

## Group & Host vars

In addition to the group & host variables required for each role (see their associated READMEs), the following group & host variables will need setting:

`inv.d/group_vars/awx/awx.yml`
```yaml
awx_git_version: 19.3.0
awx_git_dest: ~/awx
awx_inventory_data:
  pg_password: pg_password
  broadcast_websocket_secret: web_secret
  secret_key: secret_key
```
`inv.d/ds/group_vars/all/vars.yml`
```yaml
ansible_user: ansible                       # Ansible user on target host
verbosity_level: 0                          # Verbosity level when to display debug output (0 = always, 1 = -v, 2 = -vv, etc)
```

## Optional use of Ansible in Python3 virtual environment

This repo supplies a utility script for creating an Ansible virtual environment with the necessary pip package dependencies installed. This is a convenient way to run Ansible in a dedicated environment without installing it directly onto the host operating system. Run the following from the repo root directory to create & activate the Ansible venv.

```
./scripts/ansible_venv.sh && source ~/venvs/ansible_ds/bin/activate
```

__Note:__ To deactivate the venv, run `deactivate` from any directory

## Optional use of Vagrant

This repo supplies an Ansible playbook that locally creates a Vagrantfile and target preparation script using the contents of the `ds` inventory. These can be used to provision and prepare a complete [VirtualBox](https://www.virtualbox.org) test infrastructure using [Vagrant](https://www.vagrantup.com) ready for the Docker swarm deployment. Although it is designed to work out of the box by design, please ensure that the following vagrant variables are configured correctly.

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
    1. Create the `Vagrantfile`
    1. Create the `prepare_ansible_targets.sh` script
    1. Copy the SSH public key into the `vagrant` directory

```
ansible-playbook playbooks/vagrant/prepare_vagrant.yml
```

__Note:__ *To override the default enabled script check mode, run with: `-e "vagrant_script_check=no"`*

2. Running the following command will bring the entire infrastructure online

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

## Author Information

Adam Goldsmith
