
# kube ansible

These ansible roles are meant to be a base setup example to deploy vanilla kubernetes on debian and fedora based systems.

# Requirements

- Linux based systems - debian or fedora based
  - supported package managers
    - apt-get
    - dnf
  - only systemd based linux distributions(sysvinit or openrc are not supported)
  - no swap - the linux machine must not have swap enabled
  - > 1.29 - the kube version must be 1.29 or higher
- container runtime
  - the script will install containerd by default
  - CRI-O is supported(in future version)
  - docker is not supported
  - podman is not supported


# Setting up the inventory

you can create multiple inventory file in the inventory directory. Some things are required for these roles to operate properly.

There must be both the control and worker groups to configure which nodes will be used for the control plane and worker nodes.

Each of those group should have a :vars section to setup the appropriate ansible_user and should also set ansible_become to true.

see the inventory/playground/hosts file for an example of how this can be done. 


# running ansible

There are three main roles with tags...

#TODO complete this section once the playbooks are done.


# troubleshooting examples

run the ansible setup module on one of the inventory groups to view information about the target system

```sh
ansible -i inventory/playground control -m setup -K
```
