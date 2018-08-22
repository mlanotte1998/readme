# WA Monitoring for Glusterfs on a 4-Node RHEL7 VM setup
Use this ReadMe to quickly create and set up a 4-Node RHEL7 WA setup. The 4 nodes consist of one server node and 3 storage nodes that each include 1 brick. 
The only tools you should need are the three installs in Step 1 and the full WA Automated Setup folder. 

#### Based heavily off README by julim found [here](https://github.com/julienlim/multinode-glusterfs-with-tendrl-vagrant)

## Initial Setup

### Step 0 
Make sure to be on the company VPN or a company hard wire. 

Certain repos will not be downloaded without this. 

### Step 1
Install Ansible, VirtualBox, and Vagrant

You may have to run the command `$ sudo yum install kernel-devel dkms kernel-headers`

### Step 2
Copy all of the files into a directory

Edit the "conf.yml" file to include: 

* the bootstrap for your desired configuration (WA 3.3 is bootstrap_3.3.sh and WA 3.4 is bootstrap_3.4.sh) 
* Next add in the host for your desired ntp clock
* Lastly, fill in the username and password fields with your Red Hat subscription username and password 

### Step 3
Within the WA directory, run `$ vagrant up --provider virtualbox`

After this is completeed, you should have 4 VMs (node0...node3) up and running, you should be able to passwordless SSH between all nodes, and your gluster cluster should be up and running.

### Step 4
Vagrant ssh into node0 and run `$ cd /usr/share/doc/tendrl-ansible-VERSION` such that VERSION is your version of tendrl-ansible.

Then create a new file "inventory_file" that looks as follows (IP address may vary, but should be the inet ip for node0):

```text
[gluster_servers]
node1
node2
node3
[tendrl_server]
node0
[all:vars]
# Mandatory variables. In this example, 192.0.2.1 is ip address of tendrl
# server, tendrl.example.com is a hostname of tendrl server and
# tendrl.example.com hostname is translated to 192.0.2.1 ip address.
etcd_ip_address=172.28.128.3
etcd_fqdn=172.28.128.3
graphite_fqdn=172.28.128.3
configure_firewalld_for_tendrl=false
# when direct ssh login of root user is not allowed and you are connecting via
# non-root cloud-user account, which can leverage sudo to run any command as
# root without any password
#ansible_become=yes
#ansible_user=cloud-user
```

### Step 4.5 
If using the bootstrap for WA 3.3 then ...

Create a new file "site.yml" in the same directory as the inventory_file that looks as follows:

```text
- hosts: tendrl_server
  user: root
  pre_tasks:
    - name: Check that mandatory variables are defined
      assert:
        that:
          - etcd_fqdn is defined
          - etcd_ip_address is defined
          - graphite_fqdn is defined
        msg:
          - "You need to define all mandatory ansible variables to run this"
          - "playbook, see README file for guidance."
  roles:
    - tendrl-ansible.tendrl-server

- hosts: gluster_servers
  user: root
  pre_tasks:
    - name: Check that mandatory variables are defined
      assert:
        that:
          - etcd_fqdn is defined
          - graphite_fqdn is defined
        msg:
          - "You need to define all mandatory ansible variables to run this"
          - "playbook, see README file for guidance."
  roles:
    - tendrl-ansible.tendrl-storage-node
```

### Step 5
Run `$ ansible-playbook -i inventory_file site.yml`. If you run into issues try running `$ ansible -i inventory_file -m ping all` and ensure all nodes are able to communicate with one another.

You should now be able to access the Tendrl dashboard from you machine via a browser at this URL: `http://<node0-ip-address>/`
