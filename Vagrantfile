Vagrant.configure("2") do |config|
  # Global settings
  config.vm.provider "virtualbox" do |vb|
    vb.memory = 512
    vb.cpus = 2
    vb.linked_clone = true
    # Uncomment to enable GUI mode for the VM
    # vb.gui = true
  end

  config.vm.define "node1" do |pa|
    pa.vm.box = "ubuntu/jammy64" # Set the Ubuntu 22 box
    pa.vm.box_check_update = false
    pa.vm.hostname = "node1"
    pa.vm.network "public_network", ip: "192.168.1.123"

    pa.disksize.size = '5GB'  # Requires `vagrant plugin install vagrant-disksize`

    pa.vm.provider "virtualbox" do |avm|
      avm.cpus = 2
      avm.memory = 4096
    end

    # Provision script
    pa.vm.provision "shell", inline: <<-SHELL
      mkdir -p /home/vagrant/.ssh
      cat /vagrant/*.pub >> /home/vagrant/.ssh/authorized_keys
      chown -R vagrant:vagrant /home/vagrant/.ssh
      chmod 700 /home/vagrant/.ssh
      chmod 600 /home/vagrant/.ssh/authorized_keys
    SHELL
  end 
  
  # =========================
  # REPLICA 1
  # =========================
  config.vm.define "node2" do |node2|

    node2.vm.box = "ubuntu/jammy64"

    node2.vm.hostname = "node2"

    node2.vm.network "public_network", ip: "192.168.56.11"

    node2.vm.provider "virtualbox" do |vb|
      vb.memory = 4096
      vb.cpus = 2
    end

  end

  # =========================
  # REPLICA 2
  # =========================
  config.vm.define "node3" do |node3|

    node3.vm.box = "ubuntu/jammy64"

    node3.vm.hostname = "node3"

    node3.vm.network "public_network", ip: "192.168.56.12"

    node3.vm.provider "virtualbox" do |vb|
      vb.memory = 4096
      vb.cpus = 2
    end

  end

end