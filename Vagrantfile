# -*- mode: ruby -*-
# vi: set ft=ruby :

# find ssh pub key
def guess_public_key
  %w(ecdsa rsa dsa).each do |method|
    path = File.expand_path "~/.ssh/id_#{method}.pub"
    return IO.read path if File.exist? path
  end
  fail 'Public key not found.'
end

# provision script
provision = <<SCRIPT
# SSH
mkdir -p /root/.ssh
echo \"#{guess_public_key}\" >> /root/.ssh/authorized_keys
# Timezone
timedatectl set-timezone Europe/Paris

# Kernel Options
mkdir -p /etc/default/grub.d
echo 'GRUB_CMDLINE_LINUX_DEFAULT="$GRUB_CMDLINE_LINUX_DEFAULT apparmor=1 security=apparmor"' | tee /etc/default/grub.d/apparmor.cfg
update-grub

SCRIPT

# All Vagrant configuration is done below. The "2" in Vagrant.configure
# configures the configuration version (we support older styles for
# backwards compatibility). Please don't change it unless you know what
# you're doing.
Vagrant.configure("2") do |config|
  # The most common configuration options are documented and commented below.
  # For a complete reference, please see the online documentation at
  # https://docs.vagrantup.com.

  # Ensure plugins
  required_plugins = %w(vagrant-vbguest vagrant-disksize vagrant-hostmanager)
  if ARGV[0] == 'up'
    required_plugins.each do |plugin|
      system "vagrant plugin install #{plugin}" unless Vagrant.has_plugin? plugin
    end
  end

  # Plugins config
  config.hostmanager.enabled = true
  config.hostmanager.manage_host = true
  config.vbguest.auto_update = true

  # Ensure SSH forward agent
  config.ssh.forward_agent = true

  # Every Vagrant development environment requires a box. You can search for
  # boxes at https://vagrantcloud.com/search.
  config.vm.box = "debian/stretch64"

  # Disable automatic box update checking. If you disable this, then
  # boxes will only be checked for updates when the user runs
  # `vagrant box outdated`. This is not recommended.
  config.vm.box_check_update = true

  # Disk size
  config.disksize.size = "10GB"

  # Hostname
  config.vm.hostname = "ansible-dockers.example.com"

  # Example for VirtualBox:
  config.vm.provider "virtualbox" do |vb, override|
    # Name
    vb.name = "Ansible-Dockers"

    # Display the VirtualBox GUI when booting the machine
    vb.gui = false

    # Customize the amount of memory on the VM:
    vb.memory = "512"

    # CPUs
    vb.cpus = 1

    # IP
    override.vm.network :private_network, ip: "192.142.42.42"

  end

  # Provision
  config.vm.provision :shell, inline: provision

end
