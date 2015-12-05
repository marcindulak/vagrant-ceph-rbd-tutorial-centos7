-----------
Description
-----------

Configures Ceph cluster consisting of 3 CentOS 7 nodes and presents a Rados Block Device (RBD)
to a CentOS 7 client.

Tested on Ubuntu 14.04 host.


----------------------
Configuration overview
----------------------

The private 192.168.10.0/24 network is associated with the Vagrant eth1 interfaces.
eth0 is used by Vagrant.  Vagrant reserves eth0 and this cannot be currently changed
(see https://github.com/mitchellh/vagrant/issues/2093).

                 ---------------------------------------------------------
                 |                     Vagrant HOST                      |
                 ---------------------------------------------------------
                  |     |                 |     |                 |     |
              eth1| eth0|             eth1| eth0|             eth1| eth0|
    192.168.10.100|     |    192.168.10.10|     |    192.168.10.1N|     |
                 -----------             -----------             -----------
                 | client0 |             | server1 |     ...     | serverN |
                 -----------             | OSD     |             | OSD     |
                                         | MON     |             | MON     |
                                         | ADMIN   |             | ADMIN   |
                                         -----------             -----------


------------
Sample Usage
------------

Install VirtualBox https://www.virtualbox.org/ and Vagrant
https://www.vagrantup.com/downloads.html

Start three servers and install Ceph packages with::

        $ git clone https://github.com/marcindulak/vagrant-ceph-rbd-tutorial-centos7.git
        $ cd vagrant-ceph-rbd-tutorial-centos7
        $ vagrant up

The above setup follows loosely the instructions from http://docs.ceph.com/docs/master/rados/deployment/

After having the servers running configure a basic Ceph installation.
Altough the documentation does not recommend comingling monitors and OSDs on the same host,
this is what is done below. The commands are executed as **ceph** user on
only one server (**server0** here, though all the servers in this setup are equivalent)::

- create a cluster:

            $ vagrant ssh server0 -c "sudo su - ceph -c 'ceph-deploy new server{0,1,2}'"

- edit the `~ceph/ceph.conf` generated by the above command to define the public and cluster Ceph networks
  for the cluster and reduce the OSD journal size to the size of the used block device (/dev/sdc):

            $ vagrant ssh server0 -c "sudo su - ceph -c 'echo public_network = 192.168.10.0/24 >> ceph.conf'"
            $ vagrant ssh server0 -c "sudo su - ceph -c 'echo cluster_network = 192.168.10.0/24 >> ceph.conf'"
            $ vagrant ssh server0 -c "sudo su - ceph -c 'echo [osd] >> ceph.conf'"
            $ vagrant ssh server0 -c "sudo su - ceph -c 'echo osd_journal_size = 0 >> ceph.conf'"

  The reduction of the journal size is done for the purpose of this setup.
  Normally you may increase the default journal size. In a production setup the public
  and cluster networks should be different, the cluster network being a "high" performance one.
  See http://docs.ceph.com/docs/master/rados/configuration/network-config-ref/

- add monitors (MONs)::

            $ vagrant ssh server0 -c "sudo su - ceph -c 'ceph-deploy mon create server{0,1,2}'"
            $ sleep 30  # wait for the monitors for form quorum

- gather keys::

            $ vagrant ssh server0 -c "sudo su - ceph -c 'ceph-deploy gatherkeys server0'"

- synchronize `/etc/ceph/ceph.conf` and keys on all the admin (ADMIN) servers::

            $ vagrant ssh server0 -c "sudo su - ceph -c 'ceph-deploy admin server{0,1,2}'"

- list the available disks::

            $ vagrant ssh server0 -c "sudo su - ceph -c 'ceph-deploy disk list server{0,1,2}'"

- zap the /dev/sdb disks (delete the partition table)::

            $ vagrant ssh server0 -c "sudo su - ceph -c 'ceph-deploy disk zap server0:sdb server1:sdb server2:sdb'"

- prepare the OSDs (Object Storage Daemon)::

            $ vagrant ssh server0 -c "sudo su - ceph -c 'ceph-deploy osd prepare server0:sdb:/dev/sdc server1:sdb:/dev/sdc server2:sdb:/dev/sdc'"

- activate the OSDs::

            $ vagrant ssh server0 -c "sudo su - ceph -c 'ceph-deploy osd activate server0:/dev/sdb1:/dev/sdc server1:/dev/sdb1:/dev/sdc server2:/dev/sdb1:/dev/sdc'"

- modify the permissions for `ceph.client.admin.keyring`::

            $ vagrant ssh server0 -c "sudo su - ceph -c 'sudo chmod +r /etc/ceph/ceph.client.admin.keyring'"
            $ vagrant ssh server1 -c "sudo su - ceph -c 'sudo chmod +r /etc/ceph/ceph.client.admin.keyring'"
            $ vagrant ssh server2 -c "sudo su - ceph -c 'sudo chmod +r /etc/ceph/ceph.client.admin.keyring'"

- check some aspects of the cluster status from all the admin servers::

            $ vagrant ssh server0 -c "sudo su - ceph -c 'ceph health'"
            $ vagrant ssh server1 -c "sudo su - ceph -c 'ceph status'"
            $ vagrant ssh server2 -c "sudo su - ceph -c 'ceph df'"

  See http://docs.ceph.com/docs/master/rados/operations/monitoring/ for more information.

When an OSD server goes down Ceph will adjust the status of the cluster::

            $ vagrant ssh server1 -c "sudo su - -c 'shutdown -h now'"
            $ sleep 30
            $ vagrant ssh server2 -c "sudo su - ceph -c 'ceph status'"
            $ vagrant up server1
            $ sleep 30
            $ vagrant ssh server1 -c "sudo su - ceph -c 'ceph status'"

The cluster should report now *HEALTH_OK*.

A Ceph cluster offers object storage, block device and filesystem.
Here we create a Ceph block device, called RBD (Rados Block Device), and mount it on a client.
See http://docs.ceph.com/docs/master/start/quick-rbd/ for more information about RBD::

- from an ADMIN node install Ceph on the client::

            $ vagrant ssh server0 -c "sudo su - ceph -c 'ceph-deploy install client0'"

- copy `/etc/ceph/ceph.conf` and keys to the client::

            $ vagrant ssh server0 -c "sudo su - ceph -c 'ceph-deploy admin client0'"

- create a 128MB block device image rbd0::

            $ vagrant ssh client0 -c "sudo su - ceph -c 'sudo chmod +r /etc/ceph/ceph.client.admin.keyring'"
            $ vagrant ssh client0 -c "sudo su - ceph -c 'rbd create rbd0 --size 128 -m server0,server1,server2 -k /etc/ceph/ceph.client.admin.keyring'"

- map the image to the block device::

            $ vagrant ssh client0 -c "sudo su - ceph -c 'sudo rbd map rbd0 --name client.admin -m server0,server1,server2 -k /etc/ceph/ceph.client.admin.keyring'"

- create xfs filesystem and mount the device on the client::

            $ vagrant ssh client0 -c "sudo su - -c 'mkfs.xfs -L rbd0 /dev/rbd0'"
            $ vagrant ssh client0 -c "sudo su - -c 'mkdir /mnt/rbd0'"
            $ vagrant ssh client0 -c "sudo su - -c 'mount /dev/rbd0 /mnt/rbd0'"

When done, destroy the test machines with::

        $ vagrant destroy -f


------------
Dependencies
------------

None


-------
License
-------

BSD 2-clause


----
Todo
----
