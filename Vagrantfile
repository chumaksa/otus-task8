# -*- mode: ruby -*-
# vi: set ft=ruby :

MACHINES = {
  :otus=> {
        :box_name => "ubuntu/focal64",
        :vm_name => "otus",
  },

  :otuscentos => {
        :box_name => "centos/7",
        :vm_name => "otuscentos",
  }

}

Vagrant.configure("2") do |config|

  MACHINES.each do |boxname, boxconfig|
    
    config.vm.define boxname do |box|
   
      box.vm.box = boxconfig[:box_name]
      box.vm.host_name = boxconfig[:vm_name]

     end
  end
end

