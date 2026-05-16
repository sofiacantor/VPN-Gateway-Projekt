# -*- mode: ruby -*-
# vi: set ft=ruby :

Vagrant.configure("2") do |config|
  config.vm.box = "ubuntu/jammy64"

  # ============================================
  # VM 1: VPN-Gateway
  # ============================================
  config.vm.define "gateway" do |gw|
    gw.vm.hostname = "gateway"

    gw.vm.network "private_network", ip: "192.168.56.10", virtualbox__intnet: "internal-net"
    gw.vm.network "forwarded_port", guest: 51820, host: 51820, protocol: "udp"

    gw.vm.provider "virtualbox" do |vb|
      vb.name = "vpn-gateway"
      vb.memory = 1024
      vb.cpus = 1
    end

    # Ansible provisionering - WireGuard
    gw.vm.provision "ansible_local" do |ansible|
      ansible.playbook = "ansible/site.yml"
      ansible.limit = "gateway"
      ansible.inventory_path = "ansible/inventory.ini"
      ansible.compatibility_mode = "2.0"
      ansible.install_mode = "default"
    end
  end

  # ============================================
  # VM 2: Intern tjanst
  # ============================================
  config.vm.define "internal" do |int|
    int.vm.hostname = "internal"

    int.vm.network "private_network", ip: "192.168.56.20", virtualbox__intnet: "internal-net"

    int.vm.provider "virtualbox" do |vb|
      vb.name = "internal-service"
      vb.memory = 512
      vb.cpus = 1
    end

    # Ansible provisionering - nginx
    int.vm.provision "ansible_local" do |ansible|
      ansible.playbook = "ansible/site.yml"
      ansible.limit = "internal"
      ansible.inventory_path = "ansible/inventory.ini"
      ansible.compatibility_mode = "2.0"
      ansible.install_mode = "default"
    end
  end
end