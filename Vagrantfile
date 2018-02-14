# -*- mode: ruby -*-
# vi: set ft=ruby :

require 'fileutils'

VAGRANTFILE_API_VERSION = "2"

$num_instances = 3
$instance_name_prefix = "k8s"
$vm_memory = 2048

# The first three nodes are etcd servers
$etcd_instances = $num_instances
# The first two nodes are kube masters
$kube_master_instances = $num_instances == 1 ? $num_instances : ($num_instances - 1)
# All nodes are kube nodes
$kube_node_instances = $num_instances
# The following only works when using the libvirt provider
$kube_node_instances_with_disks = false
$kube_node_instances_with_disks_size = "20G"
$kube_node_instances_with_disks_number = 2

host_vars = {}

$inventory= File.join(File.dirname(__FILE__), "inventory")

if ! File.exist?(File.join(File.dirname($inventory), "hosts"))
    $vagrant_ansible = File.join(File.dirname(__FILE__), ".vagrant",
                        "provisioners", "ansible")
    FileUtils.mkdir_p($vagrant_ansible) if ! File.exist?($vagrant_ansible)

    if ! File.exist?(File.join($vagrant_ansible, "inventory"))
        FileUtils.ln_s($inventory, File.join($vagrant_ansible, "inventory"))
    end
end

$ansible_playbook = File.join(File.dirname(__FILE__), ".vagrant", "test.yml")
if ! File.exist?($ansible_playbook)
    File.open($ansible_playbook, "w") do |file|
        text = <<EOM
---
- hosts: all
  tasks:
    - name: test all machines
      debug: msg="test"
EOM
        file.puts text
    end
end

Vagrant.configure(VAGRANTFILE_API_VERSION) do |config|

    config.vm.box = "centos/7"

    (1..$num_instances).each do |i|
        config.vm.define vm_name="%s-%02d" % [$instance_name_prefix, i] do |config|
            config.vm.hostname = vm_name

            config.vm.provider :libvirt do |lv|
                lv.driver = "kvm"
                lv.memory = $vm_memory
            end

            config.ssh.insert_key = false
            config.ssh.private_key_path = ["~/.vagrant.d/insecure_private_key", "~/.ssh/id_rsa"]

            config.vm.provision :shell, privileged: false do |s|
                ssh_pub_key = File.readlines("#{Dir.home}/.ssh/id_rsa.pub").first.strip
                s.inline = <<-SHELL
                    echo #{ssh_pub_key} >> /home/$USER/.ssh/authorized_keys
                    sudo mkdir -p /root/.ssh
                    sudo bash -c "echo #{ssh_pub_key} >> /root/.ssh/authorized_keys"
                SHELL
            end

            if i == $num_instances
                config.vm.provision "ansible" do |ansible|
                    ansible.playbook = $ansible_playbook
                    if File.exist?(File.join(File.dirname($inventory), "hosts"))
                      ansible.inventory_path = $inventory
                    end
                    ansible.groups = {
                        "etcd" => ["#{$instance_name_prefix}-0[1:#{$etcd_instances}]"],
                        "kube-master" => ["#{$instance_name_prefix}-0[1:#{$kube_master_instances}]"],
                        "kube-node" => ["#{$instance_name_prefix}-0[1:#{$kube_node_instances}]"],
                        "k8s-cluster:children" => ["kube-master", "kube-node"],
                    }
                end
            end
        end
    end
end
