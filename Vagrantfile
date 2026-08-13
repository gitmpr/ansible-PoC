# -*- mode: ruby -*-
# vi: set ft=ruby :
#
# Two target VMs for the Ansible PoC. Static private-network IPs so
# inventory/hosts.yml can point at them without depending on DHCP.
# See concepts.md for why these two specifically, and why they're each
# provisioned differently.
#
# Rocky Linux 10 (RHEL-family), not Ubuntu: FreeIPA is only officially
# packaged for Fedora/RHEL/CentOS (Ubuntu isn't on its supported-platforms
# list at all — see concepts.md). Using the same distro for both VMs keeps
# things simple; Keycloak has no distro-specific requirements either way,
# it just needs Podman.

Vagrant.configure("2") do |config|
  config.vm.box = "rockylinux/10"

  config.vm.define "ipa" do |ipa|
    ipa.vm.hostname = "ipa.poc.internal"
    ipa.vm.network "private_network", ip: "192.168.56.11"
    ipa.vm.provider :libvirt do |lv|
      lv.memory = 2048
      lv.cpus = 2
    end
  end

  config.vm.define "keycloak" do |kc|
    kc.vm.hostname = "keycloak.poc.internal"
    kc.vm.network "private_network", ip: "192.168.56.12"
    kc.vm.provider :libvirt do |lv|
      lv.memory = 1536
      lv.cpus = 1
    end
  end
end
