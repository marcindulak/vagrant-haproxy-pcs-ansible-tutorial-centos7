# -*- mode: ruby -*-
# vi: set ft=ruby :

require 'digest/md5'
require 'yaml'

# Read ansible inventory
# https://stackoverflow.com/questions/41094864/is-it-possible-to-write-ansible-hosts-inventory-files-in-yaml
ansible_inventory_file = 'ansible/hosts.yml'
ansible_inventory = YAML::load_file(ansible_inventory_file)

# Base OS configuration scripts
# disable IPv6 on Linux
$linux_disable_ipv6 = <<SCRIPT
sysctl -w net.ipv6.conf.default.disable_ipv6=1
sysctl -w net.ipv6.conf.all.disable_ipv6=1
sysctl -w net.ipv6.conf.lo.disable_ipv6=1
SCRIPT
# setenforce 1
$setenforce_1 = <<SCRIPT
if test `getenforce` != 'Enforcing'; then setenforce 1; fi
sed -Ei 's/^SELINUX=.*/SELINUX=Enforcing/' /etc/selinux/config
SCRIPT

Vagrant.configure(2) do |config|
  ansible_inventory.keys.sort.each do |group|
    ansible_inventory[group]['hosts'].keys.sort.each do |host|
      config.vm.define host do |machine|
        machine.vm.box = "bento/centos-7.3"
        machine.vm.box_url = machine.vm.box
        machine.vm.hostname = host
        ipaddress = ansible_inventory[group]['hosts'][host]['ipaddress']
        # generate almost unique MAC address
        macaddress = '080027' + Digest::MD5.hexdigest(host + ipaddress).slice(0, 6)
        machine.vm.network 'private_network', ip: ipaddress, mac: macaddress, auto_config: true
        if host == ansible_inventory['mgt']['hosts'].keys.first
          machine.vm.network :forwarded_port, guest: 9090, host: 49090, id: 'prometheus', auto_correct: false
          machine.vm.network :forwarded_port, guest: 80, host: 40080, id: 'app', auto_correct: false
        end
        machine.vm.provider 'virtualbox' do |provider|
          provider.memory = 256 
          provider.cpus = 1
        end
        # Base OS configuration
        machine.vm.provision :shell, :inline => 'hostnamectl set-hostname ' + host
        machine.vm.provision :shell, :inline => $linux_disable_ipv6, run: 'always'
        machine.vm.provision :shell, :inline => $setenforce_1
        # MDTMP: todo firewall for landrush
        #machine.vm.provision :shell, :inline => 'if ! `systemctl is-active firewalld > /dev/null`; then systemctl start firewalld; fi'
        #machine.vm.provision :shell, :inline => 'if ! `systemctl is-enabled firewalld > /dev/null`; then systemctl enable firewalld; fi'
        # DNS handled by landrush
        machine.landrush.enabled = true
        machine.landrush.tld = 'mydomain'
        machine.landrush.upstream '8.8.8.8'  # landrush behaves unpredictably, sometimes DNS is not resolving
        machine.landrush.host_redirect_dns = false
        ansible_inventory.keys.sort.each do |dnsgroup|
          ansible_inventory[dnsgroup]['hosts'].keys.sort.each do |dnshost|
            dnsipaddress = ansible_inventory[dnsgroup]['hosts'][dnshost]['ipaddress']
            machine.landrush.host dnshost, dnsipaddress
          end
        end
        # highly-available loadbalancer
        config.landrush.host 'loadbalancer.' + machine.landrush.tld, '192.168.123.20'
        # trying to make landrush work - no idea still what is the problem
        # https://ma.ttias.be/centos-7-networkmanager-keeps-overwriting-etcresolv-conf
        machine.vm.provision :shell, :inline => 'if `grep "PEERDNS=yes" /etc/sysconfig/network-scripts/ifcfg-enp0s3 > /dev/null`; then sed -i "s/PEERDNS=yes/PEERDNS=no/" /etc/sysconfig/network-scripts/ifcfg-enp0s3; fi'
        # https://bugzilla.redhat.com/show_bug.cgi?id=1405431
        # [main]
        #dns=none
        machine.vm.provision :shell, :inline => 'if ! `grep "dns=none" /etc/NetworkManager/NetworkManager.conf > /dev/null`; then sed -i "/\[main\]/adns=none" /etc/NetworkManager/NetworkManager.conf; fi'
        machine.vm.provision :shell, :inline => 'systemctl restart NetworkManager'
        # Install ansible on all machines
        machine.vm.provision :shell, :inline => 'if ! rpm -q epel-release; then yum -y install https://dl.fedoraproject.org/pub/epel/7/x86_64/e/epel-release-7-9.noarch.rpm; fi'
        # disable yum fastestmirror plugin
        machine.vm.provision :shell, :inline => 'sed -i "s/^enabled=1/enabled=0/" /etc/yum/pluginconf.d/fastestmirror.conf'
        machine.vm.provision :shell, :inline => 'yum -y install ansible'
        # Key-based password-less authentication for vagrant user
        machine.vm.provision :file, source: '~/.vagrant.d/insecure_private_key', destination: '~vagrant/.ssh/id_rsa'
        machine.vm.provision :shell, :inline => 'yum -y install wget rsync python-dns'  # common tools
        machine.vm.provision :shell, :inline => 'if ! `grep "vagrant insecure public key" ~vagrant/.ssh/authorized_keys > /dev/null`; then wget --no-check-certificate https://raw.github.com/mitchellh/vagrant/master/keys/vagrant.pub -qO- >> ~vagrant/.ssh/authorized_keys; fi'
      end
    end
  end
  # Once all the machies are provisioned, run ansible from mgt
  # https://www.vagrantup.com/docs/provisioning/ansible.html
  config.vm.define ansible_inventory['mgt']['hosts'].keys.first do |machine|
    # the very first ssh as vagrant user from mgt to all running nodes
    ansible_inventory.keys.sort.each do |sshgroup|
      ansible_inventory[sshgroup]['hosts'].keys.sort.each do |sshhost|
        machine.vm.provision :shell, :inline => 'if `ping -W 1 -c 4 ' + sshhost + ' > /dev/null`; then sudo -u vagrant ssh -o StrictHostKeyChecking=no vagrant@' + sshhost + '; fi'
      end
    end
    machine.vm.provision :ansible_local do |ansible|
      ansible.inventory_path = ansible_inventory_file
      ansible.verbose = ''  # 'vvv'
      ansible.limit = 'all'
      ansible.playbook = 'ansible/playbook.yml'
    end
  end
end
