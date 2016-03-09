# osa-prep
OSA-Prep is a starter collection of playbooks for the OSA Project. It will take bare-metal (or virtual hosts if you choose), and prepare them for an OSA multi-node deployment. It will also [eventually] provision side-car initiatives such as the deployment of Ceph-Ansible, Elasticsearch/Logstash/Kibana, Nagios and other tooling.

## Steps to deploy
  1. Create your base-machine with Ubuntu 14.04 LTE
  2. Edit the inventory file according to your deployment
  3. Run the playbooks `./setup.sh -i inventory.ini`

Assumptions:

  * That you can access the hosts via ssh-keys or are set up in ~/.ssh/config correctly.
