# osa-prep
OSA-Prep is a starter collection of playbooks for preparation of the OSA Project. It will take bare-metal (or virtual hosts if you choose), and prepare them for an OSA multi-node deployment. It will also [eventually] provision side-car initiatives such as the deployment of Ceph-Ansible, Elasticsearch/Logstash/Kibana, Nagios and other tooling.

## Steps to deploy
  1. Create your base-machine with Ubuntu 14.04 LTE
  2. Edit the inventory file according to your deployment
  3. Run the playbooks `./setup.sh -i inventory.ini`

Assumptions:
These playbooks are designed to build infrastructure within an Openstack environment for testing various components of the OSA project (a highly customizable version Openstack within Openstack). If you'd like to use this project for your own purposes, then you should prepare your Ansible deployment environment like so:

  * Create entries in your ~/.ssh/config file for each of your servers like so:

  ```
  Host osa-cm01
    User ubuntu
    Hostname osa-cm01
    IdentityFile <home-dir>/user/.ssh/<ssh-key>
    StrictHostKeyChecking no
  ```

  * Next create an entry in your hostfile, or ensure that DNS is resolving for each if your hosts.
