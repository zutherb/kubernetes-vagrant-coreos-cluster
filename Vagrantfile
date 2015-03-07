# -*- mode: ruby -*-
# vi: set ft=ruby :

require 'fileutils'
require 'net/http'
require 'open-uri'

Vagrant.require_version ">= 1.6.0"

MASTER_YAML = File.join(File.dirname(__FILE__), "master.yaml")
NODE_YAML = File.join(File.dirname(__FILE__), "node.yaml")

$num_node_instances = ENV['NUM_INSTANCES'] || 3
$update_channel = ENV['CHANNEL'] || 'alpha'
$coreos_version = ENV['COREOS_VERSION'] || 'latest'
$enable_serial_logging = (ENV['SERIAL_LOGGING'].to_s.downcase == 'true')
$vb_gui = (ENV['GUI'].to_s.downcase == 'true')
$vb_master_memory = ENV['MASTER_MEM'] || 512
$vb_master_cpus = ENV['MASTER_CPUS'] || 1
$vb_node_memory = ENV['NODE_MEM'] || 1536
$vb_node_cpus = ENV['NODE_CPUS'] || 2
$kubernetes_version = ENV['KUBERNETES_VERSION'] || '0.11.0'

if $update_channel != 'alpha'
	puts "============================================================================="
	puts "As this is a fastly evolving technology CoreOS' alpha channel is the only one"
	puts "expected to behave reliably. While one can invoke the beta or stable channels"
	puts "please be aware that your mileage may vary a whole lot."
	puts "So, before submitting a bug, in this project, or upstream  (either kubernetes"
	puts "or CoreOS) please make sure it (also) happens in the (default) alpha channel."
	puts "============================================================================="
end

upstream = "http://#{$update_channel}.release.core-os.net/amd64-usr/current"
if $coreos_version == "latest"
  url = "#{upstream}/version.txt"
  $coreos_version = open(url).read().scan(/COREOS_VERSION=.*/)[0].gsub('COREOS_VERSION=', '')
end

if $kubernetes_version == "latest"
  url = "https://get.k8s.io"
  $kubernetes_version = open(url).read().scan(/release=.*/)[0].gsub('release=v', '')
end

Vagrant.configure("2") do |config|
  config.vm.box = "coreos-%s" % $update_channel
  config.vm.box_version = ">= #{$coreos_version}"
  config.vm.box_url = "#{upstream}/coreos_production_vagrant.json"

  config.vm.provider :virtualbox do |v|
    # On VirtualBox, we don't have guest additions or a functional vboxsf
    # in CoreOS, so tell Vagrant that so it can be smarter.
    v.check_guest_additions = false
    v.functional_vboxsf     = false
  end

  # plugin conflict
  if Vagrant.has_plugin?("vagrant-vbguest") then
    config.vbguest.auto_update = false
  end

  (1..($num_node_instances.to_i + 1)).each do |i|
    if i == 1
      hostname = "master"
      cfg = MASTER_YAML
      memory = $vb_master_memory
      cpus = $vb_master_cpus
    else
      hostname = "node-%02d" % (i - 1)
      cfg = NODE_YAML
      memory = $vb_node_memory
      cpus = $vb_node_cpus
    end

    config.vm.define vmName = hostname do |kHost|
      config.ssh.insert_key = false
      kHost.vm.hostname = vmName

      if vmName == "master"
        kHost.vm.network :forwarded_port, guest: 8080, host: 8080
      end

      if $enable_serial_logging
        logdir = File.join(File.dirname(__FILE__), "log")
        FileUtils.mkdir_p(logdir)

        serialFile = File.join(logdir, "%s-serial.txt" % vmName)
        FileUtils.touch(serialFile)
      end

      ["virtualbox"].each do |h|
        kHost.vm.provider h do |n|
          n.memory = memory
          n.cpus = cpus
        end
      end

      kHost.vm.network :private_network, ip: "172.17.8.#{i+100}"
      # Uncomment below to enable NFS for sharing the host machine into the coreos-vagrant VM.
      #config.vm.synced_folder ".", "/home/core/share", id: "core", :nfs => true, :mount_options => ['nolock,vers=3,udp']
      kHost.vm.synced_folder ".", "/vagrant", disabled: true

      if File.exist?(cfg)
        kHost.vm.provision :file, :source => "#{cfg}", :destination => "/tmp/vagrantfile-user-data"
        kHost.vm.provision :shell, :privileged => true,
        inline: <<-EOF
          sed -i 's,__RELEASE__,v#{$kubernetes_version},g' /tmp/vagrantfile-user-data
          mv /tmp/vagrantfile-user-data /var/lib/coreos-vagrant/
        EOF
      end
    end
  end
end

