# -*- mode: ruby -*-
# vi: set ft=ruby :

VAGRANTFILE_API_VERSION = "2"

Vagrant.configure(VAGRANTFILE_API_VERSION) do |config|
  # Same box for all nodes.
  config.vm.box = "bento/ubuntu-16.04"

  # zookeeper and kafka, 3 of each.
  types = ["zoo", "kaf"]
  replicas = 3

  types.each do |type|
    (1..replicas).each do |machine_id|
      host = "#{type}#{machine_id}"
      ip = "192.168.77.1#{types.index(type)}#{machine_id}"

      config.vm.define "#{host}" do |machine|
        machine.vm.network "private_network", ip: "#{ip}"
        machine.vm.hostname = "#{host}"

        # Only execute once the Ansible provisioner,
        # when all the machines are up and ready.
        if machine_id == replicas && type == "kaf"
          machine.vm.provision :ansible do |ansible|
            # Disable default limit to connect to all the machines at once
            ansible.limit = "all"

            # Comment / uncomment to your convenience for verbosity
            ansible.verbose = "v"

            ansible.playbook = "ansible/playbook.yml"
            ansible.inventory_path = "./ansible/vagrant_inventory"

          end
        end
      end
    end
  end
end
