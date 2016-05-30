# osa-prep
OSA-Prep is a starter collection of playbooks for preparation of the OSA Project. It will take bare-metal (or virtual hosts if you choose), and prepare them for an OSA multi-node deployment. It will also [eventually] provision side-car initiatives such as the deployment of Ceph-Ansible, Elasticsearch/Logstash/Kibana, Nagios and other tooling.


## Quickstart: Steps to deploy
  1. Create your base-machine with Ubuntu 14.04 LTE
  2. Edit the inventory file according to your deployment
  3. Create `/etc/hosts` for each of your servers by running `./setup-hosts.sh -i inventory.ini`
  4. Run the playbooks `./setup.sh -i inventory.ini`
  5. At this point, if you are setting up bridges, bonds or vlan interfaces, it is best to reboot your machine after step 4.
  6. Log into the deployment host.
  7. `sudo su -`, and create a `root` ssh-key with the following command: `ssh-keygen -t rsa -b 4096 -C "{{deploy-host}}@{{domain}}"`. Add the `/root/.ssh/id_rsa.pub` contents to each of your target hosts in `/root/.ssh/authorized_keys`.
  8. Add id_rsa.pub to each of the hosts (see notes below).
  9. Configure all of your options according to lengthy list at:
  [Mitaka - Chapter 4. Configuration Guides](http://docs.openstack.org/developer/openstack-ansible/mitaka/install-guide/configure.html).
  10. Once completed, run the syntax check tool against your configuration files: `openstack-ansible setup-infrastructure.yml --syntax-check`
  11. Set up the hosts according to your environment configuration (from `/opt/openstack-ansible/playbooks` directory): `openstack-ansible setup-hosts.yml`.
  12. If using HAproxy (as opposed to a hardware LB), make sure it is HA with the following command (from `/opt/openstack-ansible/playbooks` directory): `ansible-galaxy install -r ../ansible-role-requirements.yml`.
  13. Run the playbooks to deploy HAproxy (from `/opt/openstack-ansible/playbooks` directory): `openstack-ansible haproxy-install.yml`.
  14. Run the infrastructure playbooks (from `/opt/openstack-ansible/playbooks` directory): `openstack-ansible setup-infrastructure.yml`.
  15. Verify the database clusters:
  ```
  ansible galera_container -m shell -a "mysql \
  -h localhost -e 'show status like \"%wsrep_cluster_%\";'"
  ```
  16. Run the Openstack Setup playbooks: `openstack-ansible setup-openstack.yml`


## Using This Project
These playbooks are designed to build infrastructure within an Openstack environment for testing various components of the OSA project (a highly customizable version Openstack within Openstack). If you'd like to use this project for your own purposes, then you should prepare your Ansible deployment environment like so:

Create entries in your ~/.ssh/config file for each of your servers like so:
  ```
  Host osad-compute01
    User ubuntu
    Hostname osad-compute01
    IdentityFile <home-dir>/user/.ssh/<ssh-key>
    StrictHostKeyChecking no
  ```

Next create a corresponding entry in `/etc/hosts` on your deployment host. If your deployment host is also a target host, than `/etc/hosts` will be generated for you. Lastly, create DNS entries in your production environment and ensure that DNS is resolving for each of your hosts.

One helpful command will be the `ansible-playbook --list-tags` command. Use as shown below:
```
$ ansible-playbook site.yml --list-tags

playbook: site.yml

  play #1 (all): 	TAGS: [osa-deploy-prep]
      TASK TAGS: [osa-deploy-prep, vworks-full-dist-upgrade, vworks-install-prereqs]

  play #2 (all): 	TAGS: [osa-deploy-tools]
      TASK TAGS: [osa-deploy-tools]

  play #3 (deploy-host): 	TAGS: [osa-deploy-host]
      TASK TAGS: [osa-deploy-host]

  play #4 (target-host): 	TAGS: [osa-target-host]
      TASK TAGS: [osa-target-host]

  play #5 (target-host): 	TAGS: [osa-deploy-networking]
      TASK TAGS: [networking-dir-create, osa-deploy-networking, vworks-network-deploy]

  play #6 (target-host): 	TAGS: [osa-deploy-networking]
      TASK TAGS: [networking-dir-create, osa-deploy-networking, vworks-network-deploy]

  play #7 (logging-host): 	TAGS: [osa-deploy-networking]
      TASK TAGS: [networking-dir-create, osa-deploy-networking, vworks-network-deploy]

$
```
This command will show you all plays and associated tags within the project. This will help as we clean up our deployment. You can use it like so:
```
$ ./setup.sh -i inventory.ini --tags vworks-network-deploy --limit=astlatl-vwls01
```

If you would like `/etc/hosts` generated for each member of the inventory file (`inventory.ini`), then you can run the following command once:
```
$ ./setup-hosts -i inventory.ini
```


### Create Host Variable Files
Use the templates included in the `host_vars` directory for the type of deployment you are using. Currently there are two supported deployments: Multi-Node with Multi-NIC servers (mm), and Multi-Node with Single-NIC servers (ms). Others will be included in the future to include Single-Node with Multi-NIC deployment types and finally Single-Node with Single-NIC (better referred to as AIO, or all-in-one deployments).

Copy the deployment template for your server type (you can have mixed types depending on your switch/routing scenario), and complete the required information. Once completed, move to the next section.


### Testing your deployment
Use the following command to test your deployment:
`ansible all -m ping -i inventory.ini`


## Server Side Networking
If you need to test networking, use the following command to down and up the interfaces (note: this does not always work): `ifdown {{interface}} && ifup {{interface}}`. The {{interface}} declaration can be different when downing say vlan.202 in favor of vlan.102 (in the case of replacement).

Interfaces appeared to be flapping in the vWorks environment. I did a survey of the environment first:

ICMP Testing:
```
astlatl-vwoc02 > astlatl-vwcn02 = SUCCESS (w/delay)
astlatl-vwoc02 > astlatl-vwcn03 = SUCCESS (w/delay)
astlatl-vwoc02 > astlatl-vwcn04 = FAILURE
astlatl-vwoc03 > astlatl-vwcn02 = FAILURE
astlatl-vwoc03 > astlatl-vwcn03 = FAILURE
astlatl-vwoc03 > astlatl-vwcn04 = SUCCESS (w/delay)
astlatl-vwoc04 > astlatl-vwcn02 = SUCCESS (PERFECT)
astlatl-vwoc04 > astlatl-vwcn03 = SUCCESS
astlatl-vwoc04 > astlatl-vwcn04 = FAILURE
```


### Pinning the Primary Bond Interface Correctly
For the 10G links, Ubuntu expects the slave interface to be declared with the primary physical bond member. So I changed the following (example):
```
auto {{vworks_eth4}}
  iface {{vworks_eth4}} inet manual
    bond-master {{vworks_bond1}}

auto {{vworks_eth5}}
  iface {{vworks_eth5}} inet manual
    bond-master {{vworks_bond2}}

auto {{vworks_eth6}}
  iface {{vworks_eth6}} inet manual
    bond-master {{vworks_bond1}}
    bond-primary {{vworks_eth6}}

auto {{vworks_eth7}}
  iface {{vworks_eth7}} inet manual
    bond-master {{vworks_bond2}}
    bond-primary {{vworks_eth7}}
```

To this...
```
auto {{vworks_eth4}}
  iface {{vworks_eth4}} inet manual
    bond-master {{vworks_bond1}}
    bond-primary {{vworks_eth6}}

auto {{vworks_eth5}}
  iface {{vworks_eth5}} inet manual
    bond-master {{vworks_bond2}}
    bond-primary {{vworks_eth7}}

auto {{vworks_eth6}}
  iface {{vworks_eth6}} inet manual
    bond-master {{vworks_bond1}}

auto {{vworks_eth7}}
  iface {{vworks_eth7}} inet manual
    bond-master {{vworks_bond2}}
```

ICMP Testing:
```
astlatl-vwoc02 > astlatl-vwcn02 = SUCCESS (zero delay/no drops)
astlatl-vwoc02 > astlatl-vwcn03 = SUCCESS (zero delay/no drops)
astlatl-vwoc02 > astlatl-vwcn04 = SUCCESS (zero delay/no drops)
astlatl-vwoc03 > astlatl-vwcn02 = SUCCESS (zero delay/no drops)
astlatl-vwoc03 > astlatl-vwcn03 = SUCCESS (zero delay/no drops)
astlatl-vwoc03 > astlatl-vwcn04 = SUCCESS (zero delay/no drops)
astlatl-vwoc04 > astlatl-vwcn02 = SUCCESS (zero delay/no drops)
astlatl-vwoc04 > astlatl-vwcn03 = SUCCESS (zero delay/no drops)
astlatl-vwoc04 > astlatl-vwcn04 = SUCCESS (zero delay/no drops)
```


### Add default routes for each of the OSAD interfaces:
I would like to treat the OSAD interfaces (especially br-mgmt) as treated as separate interfaces, with their own routing tables. As such, they need a working default gateway, and in the case of `br-mgmt` they need to speak to the internet to obtain packages from Ubuntu, etc. If the following is **not** declared, your deployment will fail during `openstack-ansible setup-infrastructure.yml`. So these routes need to exist on the repo declarations at a minimum. I verified this during the following test(s):

```
root@astlatl-vwoc03:~# traceroute -i br-mgmt 8.8.8.8
traceroute to 8.8.8.8 (8.8.8.8), 30 hops max, 60 byte packets
 1  * * *
 2  * * *
 3  * * *
 4  * * *
 5  * * *
 6  * * *
 7  * * *
 8  * * *
 9  * * *
10  * * *
11  * *^C FAILED
root@astlatl-vwoc03:~#
```

To correct this, I needed to make the following changes (taken from [this article](https://www.thomas-krenn.com/en/wiki/Two_Default_Gateways_on_One_System)). I have automated this task in this project, so we don't need to do these manual tasks in the future.
```
vi /etc/iproute2/rt_tables # Add line `1 rt2`
# ip route add 10.2.31.0/24 dev br-mgmt src 10.2.31.9 table rt2
# ip route add default via 10.2.31.1 dev br-mgmt table rt2
# ip rule add from 10.2.31.9/32 table rt2
# ip rule add to 10.2.31.9/32 table rt2
```

I ran the same test again to verify a working state:
```
root@astlatl-vwoc03:~# traceroute -i br-mgmt 8.8.8.8
traceroute to 8.8.8.8 (8.8.8.8), 30 hops max, 60 byte packets
 1  10.2.31.1 (10.2.31.1)  52.433 ms  52.415 ms  52.405 ms
 2  * * *
 3  * * *
 4  * * *
 5  * * *
 6  * 216.239.51.53 (216.239.51.53)  1.993 ms google-public-dns-a.google.com (8.8.8.8)  0.373 ms
root@astlatl-vwoc03:~# exit
```

Making it persistent (again, this part is now automated):
```
auto br-mgmt
iface br-mgmt inet static
    bridge_stp off
    bridge_waitport 0
    bridge_fd 0
    # Bridge port references tagged interface
    bridge_ports {{vworks_vlan_2200}}
    address {{osad_br_mgmt_ipaddr}}
    netmask {{osad_br_mgmt_submsk}}
    gateway {{osad_br_mgmt_gatewy}}
    dns-nameservers 8.8.8.8 # {{vworks_host_namesrv}}
    dns-search {{vworks_host_sdomain}}
    post-up ip route add {{osad_br_mgmt_netwrk}}/{{osad_br_mgmt_bitmsk}} dev br-mgmt src {{osad_br_mgmt_ipaddr}} table rt2
    post-up ip route add default via {{osad_br_mgmt_gatewy}} dev br-mgmt table rt2
    post-up ip rule add from {{osad_br_mgmt_ipaddr}}/32 table rt2
    post-up ip rule add to {{osad_br_mgmt_ipaddr}}/32 table rt2
```

## TODO:
* Automate distributing the `/root/id_rsa.pub` key on the deployment server to each of the target hosts. More information can be found [HERE](http://docs.openstack.org/developer/openstack-ansible/install-guide/targethosts-prepare.html#deploying-ssh-keys). During this deployment, I manually did this task to verify it worked as expected:  `scp ~/.ssh/id_rsa.pub astlatl-vwls01:/root/.ssh/`.
* vWorks needs to find a Secrets Management solution in order to replace the password generation process described in [Openstack-Ansible project](http://docs.openstack.org/developer/openstack-ansible/install-guide/configure-creds.html). The recommendation is to use [HashiCorp Vault](https://github.com/hashicorp/vault), even over [Ansible Vault](http://docs.ansible.com/ansible/playbooks_vault.html) because of it's strength and extreme portability/reusability for cloud environments.
* I strongly suggest looking at [this project](https://projectme10.wordpress.com/2015/12/07/adding-cisco-ios-support-to-napalm-network-automation-and-programmability-abstraction-layer-with-multivendor-support/) for complete automation and abstraction of complex networking tasks. This makes networking tasks useable and simple for everyone on our team. Unless we want to [build our own solution](https://github.com/Juniper/py-junos-eznc), this project allows us to automate networking infrastructure tasks across [ISO, JunOS, PanOS, Fortigate and other networking gear commonly found throughout AT&T](https://github.com/napalm-automation/napalm/blob/master/docs/support/index.rst).
* Gather Ceph Configuration facts and add that into our deployment.
* Configure other services automatically, such as Swift, Ceph, Alarming, etc.
* Enable the following settings for each user (`/home/{{user}}/.bashrc`)
        ```
        force_color_prompt=yes
        alias dir='dir --color=auto'
        alias vdir='vdir --color=auto'
        ```
Why(?): Besides the aesthetics, colors can prevent mistakes.
* Add ssh keys for root automatically to deployment host.
* Add deployment host root public key to `/root/.ssh/authorized_keys` automatically.

## Opportunity
Would you like to contribute to the Openstack-Ansible project? These are some low-hanging-fruit items that can get you started quickly!

* bj916b: Noticed that these packages are not described or installed automatically for the deployment host: libssl-dev, libffi-dev. These items have been added to our deployment on 29-05-2016, but needs to be included in the OSAD documentation still. Contact odyssey4me (current PTL) or cloudnull (previous PTL) on how to get started.

## Potential Blockers (AT&T Specific)
* Sometimes Java will error in the background, which will cause blades to *appear* faulty. Log out of the console, and reopen a new window. You may need to power off the server and start over (painful, we know).
* Interface `bond2.2098` never came up under any conditions, no matter what I tried. The interfaces are configured exactly as the others. Will talk to Robert about this on Monday/Tuesday (hopefully he is feeling better).
* OSAD expects that x.x.x.1/y of the `br-mgmt` CIDR.
