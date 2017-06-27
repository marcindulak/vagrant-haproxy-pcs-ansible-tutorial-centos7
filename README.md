-----------
Description
-----------

An example of a https://projects.spring.io/spring-boot/ stateless web-application deployed using `vagrant` and `virtualbox`.

The setup consists of three application `app` servers, two highly-available http://www.haproxy.org/ load-balancers `lb`, and a management server.
The high-availability is achieved using a `pacemaker`/`corosync` cluster http://clusterlabs.org/.
Application deployments and system configurations are automated with https://www.ansible.com/ and monitoring is performed using https://prometheus.io/.

All servers run CentOS 7. This setup assumes a `vagrant` box with the shared folder enabled by default, mounted on the guests as `/vagrant`.

The setup requires 2GB RAM.

Tested on:

- Ubuntu 16.04 x86_64, Vagrant 1.9.5


----------------------
Configuration overview
----------------------

`vagrant` reserves enp0s3 (the first interface) and this cannot be currently changed (see https://github.com/mitchellh/vagrant/issues/2093).

The application servers and load-balancers provide services on the private network on the enp0s8 (the second) interfaces.
The Virtual IP assigned by the `pacemaker` cluster to one of the load-balancers
is used as the backend for an `apache` reverse proxy running on the management server port 80.
`vagrant` port forwarding feature is used to expose the `apache` port 80 on the management server to the `vagrant` host.

In a real-life scenarios, apart from separate networks handing different traffic (management, application),
there would be no need for an `apache` reverse proxy on the management server.
In the current setup the management server is the SPOF (single point of failure), in addition to the obvious SPOF of the `vagrant` host.



                                                    Active
                                                 load-balancer

                                               -----------------
                                               | lb1           |
                                               |               |
                                               | haproxy       |
                             Virtual IP        | node_exporter |
                         192.168.123.20/24-----| pacemaker     |
                                               | corosync      |
                                               -----------------
                                                   |       |
          Management                               |       |                              Application
            server              enp0s3 10.0.2.0/24 |       | enp0s8 192.168.123.21/24       servers
                                                   |       |
       -----------------                           |       |                           -----------------
       | mgt1          |    enp0s3 10.0.2.0/24    -----------   enp0s3 10.0.2.0/24     | appX: X=1..3  |
       |               |--------------------------|         |--------------------------|               |
       | apache        |                          | Vagrant |                          | spring-boot   |
       | prometheus    | enp0s8 192.168.123.31/24 |  HOST   | enp0s8 192.168.123.1X/24 | node_exporter |
       | node_exporter |--------------------------|         |--------------------------|               |
       |               |                          -----------                          |               |
       |               |                           |       |                           -----------------
       -----------------                           |       |
                                enp0s3 10.0.2.0/24 |       | enp0s8 192.168.123.22/24
                                                   |       |
                                                   |       |
                                               -----------------
                                               | lb2           |
                                               |               |
                                               | haproxy       |
                                               | node_exporter |
                                               | pacemaker     |
                                               | corosync      |
                                               -----------------

                                                    Passive
                                                 load-balancer


------------
Sample Usage
------------

Install `virtualbox`, including "VirtualBox Extension Pack" https://www.virtualbox.org/wiki/Downloads and
`vagrant` https://www.vagrantup.com/downloads.html

Checkout the project repo, and the external `ansible` modules:

        $ git clone https://github.com/marcindulak/vagrant-haproxy-pcs-ansible-tutorial-centos7.git
        $ cd vagrant-haproxy-pcs-ansible-tutorial-centos7
        $ cd ansible/roles/external
        $ git clone https://github.com/marcindulak/ansible-pacemaker styopa.pacemaker
        $ # ansible-galaxy install --roles-path . -r requirements.yml  # if you have ansible on the Vagrant host
        $ cd -

Install the required `vagrant` plugins and bring up the VMs:

        $ vagrant plugin install landrush
        $ vagrant up || :
        $ sleep 30  # wait for the corosync quorum

This will start the VMs, install `ansible` on them, setup key-based authentication for the `vagrant` user used for `ansible`,
and deploy https://github.com/spring-guides/gs-spring-boot/ application on the `app` VMs.

The health of the VMs can be monitored with `prometheus` API on `mgt`:9090:

        $ vagrant ssh mgt1.mydomain -c "curl -sgG --data-urlencode query='up{instance=~\"app.*\", job=\"node_exporter\"}' localhost:9090/api/v1/query | python -m json.tool"

`prometheus` port 9090 exposes also a web-gui service which provides the same information as API in so called console, and additionally allows one to create simple graphs.
The `prometheus` port is also configured to be forwarded by `vagrant` and is available at `localhost`:49090.
The screenshots below show how the same API data are presented in the console and graph.

![up_console](https://raw.github.com/marcindulak/vagrant-haproxy-pcs-ansible-tutorial-centos7/master/screenshots/up_console.png)

![up_graph](https://raw.github.com/marcindulak/vagrant-haproxy-pcs-ansible-tutorial-centos7/master/screenshots/up_graph.png)

For an introduction to `prometheus` see "An introduction to monitoring and alerting with time-series at scale, with Prometheus" https://www.youtube.com/watch?v=gNmWzkGViAY. `prometheus` provides it's own alerting system, but it is not configured here.
See "PromCon 2016: Alerting in the Prometheus Universe - Fabian Reinartz" https://www.youtube.com/watch?v=yrK6z3fpu1E for more details about alerting in `prometheus`.

The initial setup of the VMs and `ansible` playbook use internet downloads (one should use a local proxy of software mirrors instead) and most likely the initial configuration of some components will fail.
Further `ansible` runs should be performed from the `mgt` server to hopefully correct the initial problems:

        $ vagrant ssh mgt1.mydomain -c "ansible-playbook -i /vagrant/ansible/hosts.yml /vagrant/ansible/playbook.yml"

Alternatively, this `ansible` playbook could be let run by `vagrant` by:

        $ vagrant provision mgt1.mydomain

The application should be accesible now on `mgt`:80 which points to the Virtual IP managed by
the `pacemaker`/`corosync` cluster, and the corresponding port forwarded by `vagrant` to `localhost`:40080:

        $ vagrant ssh mgt1.mydomain -c "curl loadbalancer:80"
        $ curl localhost:40080

Note that the `ansible` [springboot](ansible/roles/internal/springboot) task modifies any `build.gradle` in order to build
a "fully executable" jar https://docs.spring.io/spring-boot/docs/current/reference/html/deployment-install.html
and to expose a basic `prometheus` metrics on the `app` servers using https://moelholm.com/2017/02/06/spring-boot-prometheus-actuator-endpoint/.
This is a bad practice, but it's fine for the purpose of this tutorial, instead of forking https://github.com/spring-guides.
Exposing `spring-boot` metrics to `prometheus` is very experimental, but an effort in this direction has been noticed by the developers of `spring-boot`
https://github.com/spring-projects/spring-boot/issues/8130.

Due to the fact that all VMs share a single private network, the individual `app` servers, as well as the `lb` (load-balancers) can be queried from the `mgt` server:

        $ vagrant ssh mgt1.mydomain -c "curl app1:8080"
        $ vagrant ssh mgt1.mydomain -c "curl lb1:80"

Let's deploy a different (https://github.com/spring-guides/gs-testing-web) spring boot application to a subset of `app` nodes.
First, modify the `ansible` playbook to point to the new application:

        $ cp -pf ansible/playbook.yml ansible/playbook.yml.rollback
        $ sed -i 's/gs-spring-boot/gs-testing-web/' ansible/playbook.yml

then run the playbook on the subset of nodes (here just a single node `app1.mydomain`), disabling the old service first:

        $ vagrant ssh mgt1.mydomain -c "ansible app1.mydomain -i /vagrant/ansible/hosts.yml -a 'systemctl stop gs-spring-boot' --become"
        $ vagrant ssh mgt1.mydomain -c "ansible app1.mydomain -i /vagrant/ansible/hosts.yml -a 'systemctl disable gs-spring-boot' --become"
        $ vagrant ssh mgt1.mydomain -c "ansible-playbook -i /vagrant/ansible/hosts.yml /vagrant/ansible/playbook.yml -l app1.mydomain"
        $ sleep 30

Verify the new application (which prints "Hello World") is deployed on the given node:

        $ vagrant ssh mgt1.mydomain -c "curl app1:8080"

One could alternatively completely destroy the given `app` server and make a fresh deployment. This time using the rollback playbook:

        $ vagrant destroy -f app1.mydomain
        $ vagrant up app1.mydomain
        $ vagrant ssh mgt1.mydomain -c "ansible-playbook -i /vagrant/ansible/hosts.yml /vagrant/ansible/playbook.yml.rollback"
        $ sleep 30

Verify the old application (which prints "Greetings from Spring Boot!") is deployed on the given node:

        $ vagrant ssh mgt1.mydomain -c "curl app1:8080"

Let's simulate a failure of the active load-balancer to see whether the service continues to be available.

First discover which of the `lb` servers actively holds the Virtual IP, and destroy that VM:

        $ ACTIVE_LB=`vagrant ssh lb1.mydomain -c "sudo su - -c \"pcs status | grep virtual-ip | cut -d: -f5 | cut -d' ' -f2 | tr -d '\n'\""`
        $ if `echo $ACTIVE_LB | grep -q lb1`; then PASSIVE_LB=lb2.mydomain; else PASSIVE_LB=lb1.mydomain; fi
        $ vagrant destroy -f $ACTIVE_LB

Verify that the loadbalancer has been moved to the remaining `lb` VM and the application is still available:

        $ vagrant ssh $PASSIVE_LB -c "sudo su - -c 'pcs status'"
        $ vagrant ssh mgt1.mydomain -c "curl loadbalancer:80"

Bring back, and configure the failed load-balancer. We need to copy the `/etc/corosync/corosync.conf` otherwise the `ansible` pacemaker
module will try to `pcs cluster setup` again (it's a missing logic in the `ansible` module). After copying the file one has to handle the SELinux context.

        $ vagrant up $ACTIVE_LB
        $ vagrant ssh $PASSIVE_LB -c "scp -o StrictHostKeyChecking=no -p /etc/corosync/corosync.conf $ACTIVE_LB:/tmp"
        $ vagrant ssh $ACTIVE_LB -c "sudo su - -c 'mkdir /etc/corosync&& mv /tmp/corosync.conf /etc/corosync&& chown root.root /etc/corosync/corosync.conf'"
        $ vagrant ssh $ACTIVE_LB -c "sudo su - -c 'semanage fcontext -a -s system_u -t etc_t /etc/corosync/corosync.conf'"

        $ vagrant ssh mgt1.mydomain -c "ansible-playbook -i /vagrant/ansible/hosts.yml /vagrant/ansible/playbook.yml.rollback"
        $ vagrant ssh $ACTIVE_LB -c "sudo su - -c 'pcs status'"

A note about STONITH (Shoot The Other Node In The Head): the application is stateless and the only resource managed by
the `pacemaker`/`corosync` cluster is the assignment of the Virtual IP to one of the `lb` servers.
In case where `corosync` detects a failure of the active `lb` node and `pacemaker` moves the Virtual IP to the other `lb` server
there is no risk of any data corruption due to the failed `lb` server being still alive.

When done, destroy the test machines with:

        $ vagrant destroy -f


------------
Dependencies
------------

https://github.com/vagrant-landrush/landrush


-------
License
-------

BSD 2-clause


----
Todo
----

1. few of the `ansible` modules are still used in a non-idempotent way.

2. the template files created by ansible-galaxy init should be edited.


--------
Problems
--------

