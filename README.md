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
      TASK TAGS: [osa-deploy-prep, att-full-dist-upgrade, att-install-prereqs]

  play #2 (all): 	TAGS: [osa-deploy-tools]
      TASK TAGS: [osa-deploy-tools]

  play #3 (deploy-host): 	TAGS: [osa-deploy-host]
      TASK TAGS: [osa-deploy-host]

  play #4 (target-host): 	TAGS: [osa-target-host]
      TASK TAGS: [osa-target-host]

  play #5 (target-host): 	TAGS: [osa-deploy-networking]
      TASK TAGS: [networking-dir-create, osa-deploy-networking, att-network-deploy]

  play #6 (target-host): 	TAGS: [osa-deploy-networking]
      TASK TAGS: [networking-dir-create, osa-deploy-networking, att-network-deploy]

  play #7 (logging-host): 	TAGS: [osa-deploy-networking]
      TASK TAGS: [networking-dir-create, osa-deploy-networking, att-network-deploy]

$
```
This command will show you all plays and associated tags within the project. This will help as we clean up our deployment. You can use it like so:
```
$ ./setup.sh -i inventory.ini --tags att-network-deploy --limit=host-logs01
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


## TODO:
* Automate distribution the `/root/id_rsa.pub` key on the deployment server to each of the target hosts. More information can be found [HERE](http://docs.openstack.org/developer/openstack-ansible/install-guide/targethosts-prepare.html#deploying-ssh-keys). During this deployment, I manually did this task to verify it worked as expected:  `scp ~/.ssh/id_rsa.pub host-logs01:/root/.ssh/`.
* Use secrets management like [HashiCorp Vault](https://github.com/hashicorp/vault).
* Start exploring [NAPALM](https://github.com/napalm-automation/napalm) for network automation of ISO, JunOS, FortiOS, PANOS and other gear.
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
