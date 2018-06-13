# -*- mode: ruby -*-
# vi: set ft=ruby :

Vagrant.require_version ">= 1.8.1"

BASEPATH = "/home/vagrant/go/src/github.com/appswitch/"
GOPATH = "/home/vagrant/go/"

# Shell provisioner script.
$script = <<-SCRIPT

yum check-update
yum install -y iperf3 net-tools nmap-ncat vim

# Install docker
#
curl -fsSL https://get.docker.com/ | sh
systemctl enable docker
systemctl start docker
usermod -aG docker vagrant

# Install docker-compose
#
curl -L https://github.com/docker/compose/releases/download/1.21.2/docker-compose-$(uname -s)-$(uname -m) -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose

# Stop and disable firewall
#
systemctl stop firewalld
systemctl disable firewalld

SCRIPT


Vagrant.configure(2) do |config|
  config.vm.box = "CentosBox/Centos7-v7.3-Minimal"
  config.vm.synced_folder ".", "/home/vagrant/appswitch"

  config.ssh.pty = true
  # Use the same key for each machine.
  config.ssh.insert_key = false

  config.vm.provision "shell", inline: $script, privileged: true

  #
  # Naming convention: hostXY
  #  X: cluster number
  #  Y: node number
  #
  #              cluster0                                          cluster1
  #     host01              host00                        host10              host11
  #
  #                        10.0.0.10  -----------------  10.0.0.11
  # 192.168.0.11  -----  192.168.0.10                  192.168.1.10  -----  192.168.1.11

  nclusters = 2
  nnodes = 2
  nclusters.times do |clustern|
    nnodes.times do |noden|
      name = "host#{clustern}#{noden}"
      config.vm.define name do |node|
        node.vm.hostname = name
        node.vm.network "private_network", ip: "192.168.#{clustern}.#{noden+10}",  virtualbox__intnet: true

        # Add second interface for federation connectivity.
        if noden == 0
          node.vm.network "private_network", ip: "10.0.0.#{clustern+10}",  virtualbox__intnet: true
        end

        node.vm.provider "virtualbox" do |vb|
          vb.name = "ax_" + name
	end
      end
    end
  end
end
