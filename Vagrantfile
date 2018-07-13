# -*- mode: ruby -*-
# vi: set ft=ruby :

# check it: http://bertvv.github.io/notes-to-self/2015/10/05/one-vagrantfile-to-rule-them-all/
require 'yaml'
require 'fileutils'

#lets get started with a dir to save stuff into and add stuff to
FileUtils::mkdir_p './vagrant'
FileUtils::mkdir_p './logs'

KUBE_STACK = ENV['KUBE_STACK'] ? ENV['KUBE_STACK'] : $kube_stack ? $kube_stack : "hypercluster"
$local_release_dir = "/vagrant/temp"
STACK = YAML.load_file("stacks/#{KUBE_STACK}.yml")
inventory_name = STACK.key?('inventory') ? STACK['inventory'] : KUBE_STACK
$inventory = File.join(File.dirname(__FILE__), "inventory", inventory_name)

Vagrant.require_version ">= 2.0.0"

CONFIG = File.join(File.dirname(__FILE__), "vagrant/config.rb")

COREOS_URL_TEMPLATE = "https://storage.googleapis.com/%s.release.core-os.net/amd64-usr/current/coreos_production_vagrant.json"

# Uniq disk UUID for libvirt
DISK_UUID = Time.now.utc.to_i

SUPPORTED_OS = {
  "coreos-stable" => {box: "coreos-stable",      bootstrap_os: "coreos", user: "core", box_url: COREOS_URL_TEMPLATE % ["stable"]},
  "coreos-alpha"  => {box: "coreos-alpha",       bootstrap_os: "coreos", user: "core", box_url: COREOS_URL_TEMPLATE % ["alpha"]},
  "coreos-beta"   => {box: "coreos-beta",        bootstrap_os: "coreos", user: "core", box_url: COREOS_URL_TEMPLATE % ["beta"]},
  "ubuntu"        => {box: "bento/ubuntu-16.04", bootstrap_os: "ubuntu", user: "vagrant"},
  "centos"        => {box: "akx/centos",           bootstrap_os: "centos", user: "vagrant"},
  "opensuse"      => {box: "opensuse/openSUSE-42.3-x86_64", bootstrap_os: "opensuse", use: "vagrant"},
  "opensuse-tumbleweed" => {box: "opensuse/openSUSE-Tumbleweed-x86_64", bootstrap_os: "opensuse", use: "vagrant"},
}

GUEST_ADDITIONS_ISO = ENV['GUEST_ADDITIONS_ISO'] ? ENV['GUEST_ADDITIONS_ISO'] : $guest_additions_iso ? $guest_additions_iso : nil

# the project name
STACK['project'] = 'kubernetes' if !STACK.key?('project')
STACK['cache'] = false if !STACK.key?('cache')
  
VAGRANTFILE_API_VERSION = STACK.key?('version') ? STACK['version'] : ENV['VAGRANTFILE_API_VERSION'] ? ENV['VAGRANTFILE_API_VERSION'] : $vagrantfile_api_version ? $vagrantfile_api_version : '2'
BASE_OS = STACK.key?('base_os') ? STACK['base_os'] : ENV['BASE_OS'] ? ENV['BASE_OS'] : $base_os ? $base_os : 'centos'

# get the computers default gateway and adapter and base ip based on the gateway
#PUBLIC_GATEWAY = `ip route | awk '/default/ { print $3 }'`.strip!
PUBLIC_GATEWAY = STACK.key?('public_gateway') ? STACK['public_gateway'] : ENV['PUBLIC_GATEWAY'] ? ENV['PUBLIC_GATEWAY'] : $public_gateway ? $public_gateway : false
PUBLIC_IP_BASE = PUBLIC_GATEWAY[0...PUBLIC_GATEWAY.rindex('.')]

#do the same above for the private networking
#PRIVATE_GATEWAY = `ip route | grep vboxnet0 | awk '{print $9}'`.strip!
PRIVATE_GATEWAY = STACK.key?('private_gateway') ? STACK['private_gateway'] : ENV['PRIVATE_GATEWAY'] ? ENV['PRIVATE_GATEWAY'] : $private_gateway ? $private_gateway : false
PRIVATE_IP_BASE = PRIVATE_GATEWAY[0...PRIVATE_GATEWAY.rindex('.')]

# retrieve the default network adapters name
ADAPTER = STACK.key?('adapter') ? STACK['adapter'] : ENV['ADAPTER'] ? ENV['ADAPTER'] : $adapter ? $adapter : false

# get the default folder where vbox stores it's images
VB_MACHINE_FOLDER = `VBoxManage list systemproperties | grep "Default machine folder"`.split(':')[1].strip!

# these are arrays of grouped machines for ansible inventory
STACK['groups'] = {} if !STACK.key?('groups')
STACK['groups']["k8s-cluster:children"] = ["kube-master", "kube-node"]
  
# handle any plugin dependencies
required_plugins = %w(vagrant-vbguest vagrant-hostmanager sahara)
plugins_to_install = required_plugins.select { |plugin| not Vagrant.has_plugin? plugin }
if not plugins_to_install.empty?
  puts "Installing plugins: #{plugins_to_install.join(' ')}"
  if system "vagrant plugin install #{plugins_to_install.join(' ')}"
    exec "vagrant #{ARGV.join(' ')}"
  else
    abort "Installation of one or more plugins has failed. Aborting."
  end
end

# if $inventory has a hosts file use it, otherwise copy over vars etc
# to where vagrant expects dynamic inventory to be.
if ! File.exist?(File.join(File.dirname($inventory), "hosts"))
  $vagrant_ansible = File.join(File.dirname(__FILE__), ".vagrant",
                       "provisioners", "ansible")
  FileUtils.mkdir_p($vagrant_ansible) if ! File.exist?($vagrant_ansible)
  if ! File.exist?(File.join($vagrant_ansible,"inventory"))
    FileUtils.ln_s($inventory, File.join($vagrant_ansible,"inventory"))
  end
end

def design_ports(box_def,box) 
  ports = []
  box_def['ports'].each do |port_def|
    
    guest_port = port_def['expose']
    host_port = guest_port
    while $host_ports.include? host_port
      host_port = host_port + 1
    end
    
    protocol = port_def.key?('protocol') ? port_def['protocol'] : 'tcp'
    port = {
      'name' => port_def.key?('name') ? port_def['name'] : "#{protocol}#{guest_port}",
      'host' => host_port,
      'guest' => guest_port,
      'host_ip' => port_def.key?('host_ip') ? port_def['host_ip'] : "127.0.0.1",
      'guest_ip' => port_def.key?('guest_ip') ? port_def['guest_ip'] : nil,
      'protocol' => protocol
    }
    
    $host_ports.push(host_port)
    ports.push( port )
    
  end
  box['ports'] = ports
end

def design_storage(box_def,box)
  STACK['groups']['brick'] = [] if !STACK['groups'].key?('brick')
  STACK['groups']['brick'].push(box['name']) if !STACK['groups']['brick'].include? box['name']
  
  storage_controllers = []
  box_def['storage'].each do |controller_def|
    storage_controller = {
      'name' => controller_def['name'],
      'type' => controller_def['type'],
      'disks' => []
    }
    #now add the disks to the controller
    disk_def = controller_def['disks']
    (1..disk_def['count']).each do |ii|
      storage_disk = {
        'name' => "#{disk_def['prefix']}-#{ii}-#{box['name']}",
        'size' => disk_def['size'],
        'type' => disk_def.key?('type') ? disk_def['type'] : 'VDI'
      }
      storage_controller['disks'].push(storage_disk)
    end
    storage_controllers.push(storage_controller)
  end
  
  box['storage'] = storage_controllers
end

# kubernetes always has to be external or else nothing will get created
STACK['control_plane']['kube-master']['external'] = true
STACK['control_plane']['kube-node']['external'] = true
 
local_release_dir = "./vagrant/temp" 
$host_vars = {}
$host_ports = []
ip_end = Integer(STACK['ip_end'])
STACK['boxes'] = [] if !STACK.key?('boxes')
# only one is primary so set it to false after first round
is_primary = false 
# go through all the pieces of the control plane to define boxes
STACK['control_plane'].each do |item|
  
  # get the val on the second index
  box_def = item[1]
  
  if box_def['external']
    
    # get the group ready for adding to
    STACK['groups']["#{item[0]}"] = [] if !STACK['groups'].key?("#{item[0]}")
    host_group = STACK['groups']["#{item[0]}"]
    
    # create one box for each replica
    (1..box_def['replicas']).each do |i|
      
      box_os = box_def.key?('os') ? box_def['os'] : BASE_OS
        
      box = {
        'name' => (box_def['replicas'] == 1) ? "#{box_def['prefix']}" : "#{box_def['prefix']}-#{i}",
        'description' => box_def.key?('description') ? box_def['description'] : '',
        'image' => SUPPORTED_OS[box_os][:box],
        'image_url' => SUPPORTED_OS[box_os].key?('box_url') ? SUPPORTED_OS[box_os]['box_url'] : nil,
        'private_ip' => "#{PRIVATE_IP_BASE}.#{ip_end}",
        'public_ip' => "#{PUBLIC_IP_BASE}.#{ip_end}",
        'memory' => box_def['memory'],
        'cpus' => box_def['cpus'],
        'primary' => ((item[0] == 'kube-master') and (i == 1)) # check the first index if this is the master loop
      }
      
      box['volumes'] = box_def['volumes'] if box_def.key?('volumes')
      
      #build the storage disks if needed
      design_storage(box_def,box) if box_def.key?('storage')
        
      # add any port mappings needed
      design_ports(box_def, box) if box_def.key?('ports')
      
      $host_vars[box['name']] = {
        "ip": box['private_ip'],
        "bootstrap_os": SUPPORTED_OS[box_os][:bootstrap_os],
        "local_release_dir" => local_release_dir,
        "download_run_once": "False",
        "kube_network_plugin": STACK.key?('network_plugin') ? STACK['network_plugin'] : 'weave'
      }
      
      # add the box to the array of boxes
      STACK['boxes'].push(box)
        
      #add the box to the host group
      host_group.push(box['name'])
        
      # increment the ip
      ip_end = (ip_end + 1)
      
    end # end of box def loop
    
  end # end of check for ha clusters
  
end # end control plane loop

# now make a group for the nodes with the gluster bricks
#STACK['groups']['brick'] = STACK['groups'][BRICKS]
#DISK_CONTROLLER = STACK['control_plane'][BRICKS]['disks']['controller']['name']
  
# make sure all nodes and master is a kubelet
STACK['groups']['kubelet'] = [STACK['groups']['kube-node'],STACK['groups']['kube-master']].flatten

# base configuration for all of the boxes
Vagrant.configure(VAGRANTFILE_API_VERSION) do |config|
  
  config.ssh.insert_key = false
  if (GUEST_ADDITIONS_ISO != nil)
    config.vbguest.iso_path = GUEST_ADDITIONS_ISO
  end
  # plugin conflict
  if Vagrant.has_plugin?("vagrant-vbguest") then
    config.vbguest.auto_update = false
  end
  config.vm.synced_folder ".", "/vagrant", disabled: true
  
  # this makes the public network reachable from a remote system
  config.vm.provision "shell",
    run: "always",
    inline: "sudo route add default gw #{PUBLIC_GATEWAY} eth2"
    
  # initialize as true so first node becomes primary node
  STACK['boxes'].each do |box|
    
    define_vm(config.vm, box)
    
  end # end of box loop
  
end # end of vagrant configure

def define_vm(vm, box)
  
  vm.define box['name'], primary: box['primary'] do |node|
          
      node.vm.box = box['image']
      
      node.vm.box_url = box['image_url'] if (box[:image_url] != nil)
        
      node.vm.hostname = box['name']
       
      # create a private network 
      node.vm.network :private_network, ip: box['private_ip']
         
      # create a public network     
      node.vm.network "public_network", 
        bridge: [ADAPTER], 
        ip: box['public_ip'], 
        :use_dhcp_assigned_default_route => true,
        :gateway => PUBLIC_GATEWAY,
        auto_config: true
     
      
     define_provider(node.vm, box)
     
     # make any volumes if requested
     define_volumes(node.vm, box) if (box.key?('volumes')) 
     
     # add any port forwarding if needed
     define_ports(node.vm, box) if (box.key?('ports')) 
       
     # setup the ansible provisioner only on the primary master
     define_provisioner(node.vm, box) if box['primary']
     
    end # end of box define
  
end


def define_ports(vm, box)
  
  box['ports'].each do |port|
    vm.network "forwarded_port", 
      id: port['name'],
      guest: port['guest'], 
      host: port['host'], 
      host_ip: port['host_ip'],
      guest_ip: port['guest_ip'],
      protocol: port['protocol'],
      auto_correct: true
  end
  
end

def define_volumes(vm, box)
  
  box['volumes'].each do |vol|
    
    type = vol.key?('type') ? vol['type'] : 'VirtualBox'
    vm.synced_folder vol['src'], vol['dest'], 
      type: vol['type'], create: true
  
  end
  
end

# define how the virtual boxes are set up  
def define_provider(vm, box)
  
  vm.provider :virtualbox do |vb|
    vb.memory = box['memory']
    vb.cpus = box['cpus']
    # give the master a name
    vb.name = box['name']
      
    # add all the boxes to the project grouping in virtualbox
    vb.customize [
      'modifyvm', :id, 
      '--groups', "/#{STACK['project']}",
      '--description', box['description']
    ]
    
    # create any extra disks if any are defined in the box def  
    add_disks(vb, box) if box.key?('storage')
      
  end # end of provider block
  
  vm.provider :libvirt do |lv|
    lv.memory = box['memory']
  end
  
  ["vmware_fusion", "vmware_workstation"].each do |vmware|
    vm.provider vmware do |v|
      v.vmx['memsize'] = box['memory']
      v.vmx['numvcpus'] = box['cpus']
    end
  end
  
end

def add_disks(vb, box)
  
  box['storage'].each do |storage_controller|
    
    #has_controller = `if VBoxManage showvminfo #{box['name']} | grep -q '#{storage_controller['name']}'; then echo true; else echo false; fi`.strip!
  
    #unless (has_controller == 'true')
      # create a controller for the disks to attach to
      vb.customize [
        "storagectl", :id,
        "--name", storage_controller['name'], 
        "--add", storage_controller['type'],
        "--controller", "IntelAHCI"
      ]
    #end
    
    # create each of the disks
    #disk_def['count'] = Integer(disk_def['count'])
    i = 1
    storage_controller['disks'].each do |disk|
      
      new_disk = File.join(
         VB_MACHINE_FOLDER, 
         "#{STACK['project']}/#{box['name']}", 
         "#{disk['name']}.vdi"
       )
      unless File.exist?(new_disk)
        vb.customize [
          'createhd', 
          '--filename', new_disk, 
          '--format', disk['type'], 
          '--size', disk['size'] * 1024
        ]
        vb.customize [
          'storageattach', :id, 
          '--storagectl', storage_controller['name'], 
          '--port', 3+i, 
          '--device', 0, 
          '--type', 'hdd', 
          '--medium', new_disk
        ]
        i = i + 1
      end # end of file exists check
      
    end # end of disk loop
    
  end # end of controller loop
  
end

def define_provisioner(vm, box)
  
  stack_var_file = File.join(File.dirname(__FILE__), "vagrant/stack.yml")
  File.open(stack_var_file, "w") do |file|
    file.write({
      "vagrant_home"  => ENV['VAGRANT_HOME'] ? ENV['VAGRANT_HOME'] : "~/.vagrant.d",
      "vagrant_cache" => ENV['VAGRANT_CACHE'] ? ENV['VAGRANT_CACHE'] : STACK['cache'],
      "vagrant_master"  => box['private_ip'],
      "vagrant_master_name" => box['name'],
      "base_domain" => STACK['domain'],
      "disable_swap" => true,
      "stack" => STACK,
    }.to_yaml)
  end
  
  num_instances = STACK['control_plane']['kube-node']['replicas']
  
  # View the documentation for the provider you're using for more
  # information on available options.
  vm.provision :ansible do |ansible|
    ansible.playbook = "cluster-test.yml"
    ansible.verbose = 'vvvv'
    ansible.become = true
    ansible.limit = "all,localhost"
    ansible.host_key_checking = false
    ansible.raw_arguments = ["--forks=#{num_instances}", "--flush-cache"]
    ansible.host_vars = $host_vars
    ansible.groups = STACK['groups']
    ansible.extra_vars = stack_var_file
    if File.exist?(File.join(File.dirname($inventory), "hosts"))
      ansible.inventory_path = $inventory
    end
  end
  
end