## -*- mode: ruby -*-
## vi: set ft=ruby :
require 'yaml'
require './lib/gen_node_infos'
require './lib/predicates'

def is_plugin(name)
  if Vagrant.has_plugin?(name)
    puts "using #{name}"
  else
    puts "please run vagrant plugin install #{name}"
    exit(1)
  end
end

base_dir  = File.expand_path(File.dirname(__FILE__))
conf      = YAML.load_file(File.join(base_dir, "cluster.yml"))
ninfos    = gen_node_infos(conf)

## vagrant plugins required:
## vagrant-aws, vagrant-berkshelf, vagrant-omnibus, vagrant-hosts, vagrant-cachier
Vagrant.configure("2") do | config |
  if !conf["custom_ami"] then
    ## https://vagrantcloud.com/ehime/boxes/mesos
    config.vm.box = "ehime/mesos"
    config.vm.box_version = "1.0.0"
  end

  ## enable plugins
  config.berkshelf.enabled = true
  config.berkshelf.berksfile_path ="./Berksfile"
  config.omnibus.chef_version = :latest

  ## if you want to use vagrant-cachier,
  ## please install vagrant-cachier plugin.
  if Vagrant.has_plugin?("vagrant-cachier")
    config.cache.enable :apt
    config.cache.enable :yum
    config.cache.enable :chef
  end

  is_plugin("vagrant-berkshelf")
  is_plugin("vagrant-omnibus")
  is_plugin("vagrant-hosts")

  ## define VMs. all VMs have identical configuration.
  [ninfos[:zk], ninfos[:master], ninfos[:slave]].flatten.each_with_index do | ninfo, i |
    config.vm.define ninfo[:hostname] do | cfg |

      cfg.vm.provider :virtualbox do | vb, override |
        override.vm.hostname = ninfo[:hostname]
        override.vm.network :private_network, :ip => ninfo[:ip]
        override.vm.provision :hosts

        vb.name = 'vagrant-mesos-' + ninfo[:hostname]
        vb.customize ["modifyvm", :id, "--memory", ninfo[:mem], "--cpus", ninfo[:cpus] ]

        override.vm.provision :shell do | s |
          s.path = "scripts/populate_sshkey.sh"
          s.args = "/root root"
        end

        override.vm.provision :shell do | s |
          s.path = "scripts/populate_sshkey.sh"
          s.args = "/home/vagrant vagrant"
        end
      end

      cfg.vm.provider :aws do | aws, override |
        aws.access_key_id         = conf["access_key_id"]
        aws.secret_access_key     = conf["secret_access_key"]

        aws.region = conf["region"]
        if conf["custom_ami"] then
            override.vm.box       = "dummy"
            override.vm.box_url   = "https://github.com/mitchellh/vagrant-aws/raw/master/dummy.box"
            aws.ami               = conf["custom_ami"]
        end

        ## workaround for https://github.com/mitchellh/vagrant-aws/issues/275
        aws.ami = ""

        aws.instance_type         = ninfo[:instance_type]
        aws.keypair_name          = conf["keypair_name"]
        aws.subnet_id             = conf["subnet_id"]
        aws.security_groups       = conf["security_groups"]
        aws.private_ip_address    = ninfo[:ip]
        aws.tags = {
          Name: "vagrant-mesos-#{ninfo[:hostname]}"
        }
        if !conf["default_vpc"] then
          aws.associate_public_ip = true
        end

        override.ssh.username = "ubuntu"
        override.ssh.private_key_path = conf["ssh_private_key_path"]

        override.vm.provision :shell do | s |
          s.path = "scripts/populate_sshkey.sh"
          s.args = "/home/ubuntu ubuntu"
        end

        if master?(ninfo[:hostname]) || slave?(ninfo[:hostname]) then
          override.vm.provision :shell , :inline => <<-SCRIPT
            PUBLIC_DNS=`wget -q -O - http://169.254.169.254/latest/meta-data/public-hostname`
            hostname $PUBLIC_DNS
            echo $PUBLIC_DNS > /etc/hostname
            HOSTNAME=$PUBLIC_DNS  ## Fix the bash built-in hostname variable too
            SCRIPT
        end

        if master?(ninfo[:hostname]) then
          override.vm.provision :shell , :inline => 'restart mesos-master'
        end

        if slave?(ninfo[:hostname]) then
          override.vm.provision :shell , :inline => 'restart mesos-slave'
        end
      end

      ## mesos-master doesn't create its work_dir.
      master_work_dir = "/var/run/mesos"
      if master?(ninfo[:hostname]) then
        cfg.vm.provision :shell, :inline => "mkdir -p #{master_work_dir}"
      end

      cfg.vm.provision :chef_solo do | chef |
##       chef.log_level = :debug
        chef.add_recipe "apt"
        if master?(ninfo[:hostname]) then
          chef.add_recipe "mesos::master"
          chef.json  = {
            :mesos=> {
              :type          => "mesosphere",
              :version       => conf["mesos_version"],
              :master_ips    => ninfos[:master].map { |m| "#{m[:ip]}" },
              :slave_ips     => ninfos[:slave].map { | s | "#{s[:ip]}" },
              :mesosphere    => if conf["mesos_build"].length > 0 then
                {
                  :build_version => conf["mesos_build"]
                }
              else
                {
                  :build_version => ""
                }
              end,

              :master        => if ninfos[:zk].length > 0 then
                {
                  :cluster  => "MyCluster",
                  :quorum   => "#{(ninfos[:master].length.to_f/2).ceil}",
                  :work_dir => master_work_dir,
                  :zk       => "zk://"+ninfos[:zk].map{|zk| zk[:ip]+":2181"}.join(", ")+"/mesos",
                  :ip       => "#{ninfo[:ip]}"
                }
              else
                {
                  :cluster  => "MyCluster",
                  :quorum   => "#{(ninfos[:master].length.to_f/2).ceil}",
                  :work_dir => master_work_dir,
                  :ip       => "#{ninfo[:ip]}"
                }
              end
            }
          }
        elsif slave?(ninfo[:hostname]) then
          chef.add_recipe "mesos::slave"
          chef.json = {
            :mesos => {
              :type         => "mesosphere",
              :version      => conf["mesos_version"],

              :mesosphere    => if conf["mesos_build"].length > 0 then
                {
                  :build_version => conf["mesos_build"]
                }
              else
                {
                  :build_version => ''
                }
              end,

              :slave        => {
                :master         => if ninfos[:zk].length > 0 then
                                    "zk://"+ninfos[:zk].map{|zk| zk[:ip]+":2181"}.join(", ")+"/mesos"
                                  else
                                    "#{ninfos[:master][0][:ip]}:5050"
                                  end,
                :ip              => "#{ninfo[:ip]}",
                :containerizers  => "docker,mesos",
                :isolation       => "cgroups/cpu,cgroups/mem",
                :executor_registration_timeout => "5mins",
              }
            }
          }
        end
      end

      if zk?(ninfo[:hostname]) then
        myid = (/zk([0-9]+)/.match ninfo[:hostname])[1]
        cfg.vm.provision :shell, :inline => <<-SCRIPT
          sudo mkdir -p /tmp/zookeeper
          sudo chmod 755 /tmp/zookeeper
          sudo chown zookeeper /tmp/zookeeper
          sudo -u zookeeper echo #{myid} > /tmp/zookeeper/myid
          sudo -u zookeeper /opt/chef/embedded/bin/ruby /vagrant/scripts/gen_zoo_conf.rb > /etc/zookeeper/conf/zoo.cfg
          sudo restart zookeeper
        SCRIPT
      end

      ## If you wanted use `.dockercfg` file
      ## Please place the file simply on this directory
      if File.exist?(".dockercfg")
        config.vm.provision :shell, :priviledged => true, :inline => <<-SCRIPT
          cp /vagrant/.dockercfg /root/.dockercfg
          chmod 600 /root/.dockercfg
          chown root /root/.dockercfg
        SCRIPT
      end
    end
  end

  if conf["marathon_enable"] then
    config.vm.define :marathon do | cfg |
      marathon_ip = conf["marathon_ipbase"]+"11"
      cfg.vm.provider :virtualbox do | vb, override |
        override.vm.hostname = "marathon"
        override.vm.network :private_network, :ip => marathon_ip
        override.vm.provision :hosts

        vb.name = 'vagrant-mesos-' + "marathon"
        vb.customize ["modifyvm", :id, "--memory", conf["marathon_mem"], "--cpus", conf["marathon_cpus"] ]

        if conf["marathon_version"] != "0.8.2" and conf["marathon_version"] != ""
          override.vm.provision :shell , :inline => <<-SCRIPT
            sudo apt-key adv --keyserver keyserver.ubuntu.com --recv E56151BF

            DISTRO=$(lsb_release -is | tr '[:upper:]' '[:lower:]')
            CODENAME=$(lsb_release -cs)
            echo "deb http://repos.mesosphere.com/${DISTRO} ${CODENAME} main" |\
            sudo tee /etc/apt/sources.list.d/mesosphere.list

            sudo apt-get -y install equivs
            equivs-control dummy-jdk-8u91
            echo -e "### Commented entries have reasonable defaults.\n### Uncomment to edit them.\nSection: misc\nPriority: optional\nStandards-Version: 3.9.2\n\nPackage: dummy-oracle-jdk\nVersion: 1.8.91\nProvides: java8-runtime-headless,default-jre-headless,java-runtime-headless\nDescription: dummy package to satisfy java8-runtime-headless dep for manual install\n long description and info\n .\n second paragraph" >| dummy-jdk-8u91
            equivs-build dummy-jdk-8u91
            sudo dpkg -i dummy-oracle-jdk_1.8.91_all.deb

            sudo apt-get -y update
            sudo apt-get install -y marathon=$(apt-cache madison marathon |grep #{conf["marathon_version"]} |awk -F'|' '{print$2}' |tr -d ' ')
          SCRIPT
        end


        override.vm.provision :shell do | s |
          s.path = "scripts/populate_sshkey.sh"
          s.args = "/root root"
        end

        override.vm.provision :shell do | s |
          s.path = "scripts/populate_sshkey.sh"
          s.args = "/home/vagrant vagrant"
        end


      end

      cfg.vm.provider :aws do | aws, override |
        aws.access_key_id       = conf["access_key_id"]
        aws.secret_access_key   = conf["secret_access_key"]

        aws.region = conf["region"]
        if conf["custom_ami"] then
          override.vm.box       = "dummy"
          override.vm.box_url   = "https://github.com/mitchellh/vagrant-aws/raw/master/dummy.box"
          aws.ami               = conf["custom_ami"]
        end

        ## workaround for https://github.com/mitchellh/vagrant-aws/issues/275
        aws.ami=""

        aws.instance_type       = conf["marathon_instance_type"]
        aws.keypair_name        = conf["keypair_name"]
        aws.subnet_id           = conf["subnet_id"]
        aws.security_groups     = conf["security_groups"]
        aws.private_ip_address  = marathon_ip
        aws.tags = {
          Name: "vagrant-mesos-marathon"
        }
        if !conf["default_vpc"] then
          aws.associate_public_ip = true
        end

        override.ssh.username = "ubuntu"
        override.ssh.private_key_path = conf["ssh_private_key_path"]

        override.vm.provision :shell do | s |
          s.path = "scripts/populate_sshkey.sh"
          s.args = "/home/ubuntu ubuntu"
        end

        override.vm.provision :shell , :inline => <<-SCRIPT
          PUBLIC_DNS=`wget -q -O - http://169.254.169.254/latest/meta-data/public-hostname`
          hostname $PUBLIC_DNS
          echo $PUBLIC_DNS > /etc/hostname
          HOSTNAME=$PUBLIC_DNS  ## Fix the bash built-in hostname variable too
          SCRIPT
      end

      cfg.vm.provision :shell, :privileged => true, :inline => <<-SCRIPT
        mkdir -p /var/log/marathon
        kill -KILL `ps augwx | grep marathon | tr -s " " | cut -d' ' -f2`
        LIBPROCESS_IP=#{marathon_ip} nohup /opt/marathon/bin/start --master #{"zk://"+ninfos[:zk].map{|zk| zk[:ip]+":2181"}.join(",")+"/mesos"} --zk_hosts #{ninfos[:zk].map{|zk| zk[:ip]+":2181"}.join(",")} --event_subscriber http_callback > /var/log/marathon/nohup.log 2> /var/log/marathon/nohup.log < /dev/null &
        echo LIBPROCESS_IP=#{marathon_ip} nohup /opt/marathon/bin/start --master #{"zk://"+ninfos[:zk].map{|zk| zk[:ip]+":2181"}.join(",")+"/mesos"} --zk_hosts #{ninfos[:zk].map{|zk| zk[:ip]+":2181"}.join(",")} --event_subscriber http_callback >| ~/marathon.runner
        SCRIPT

      if conf["chronos_enable"] then
        cfg.vm.provision :shell, :privileged => true, :inline => <<-SCRIPT
          ## Install and run Chronos based on:
          ## https://mesosphere.io/learn/run-chronos-on-mesos/
          mkdir -p /var/log/chronos
          LIBPROCESS_IP=#{marathon_ip} nohup /opt/chronos/bin/start-chronos.bash --master #{"zk://"+ninfos[:zk].map{|zk| zk[:ip]+":2181"}.join(",")+"/mesos"} --zk_hosts #{"zk://"+ninfos[:zk].map{|zk| zk[:ip]+":2181"}.join(",")+"/mesos"} --http_port 8081 > /var/log/chronos/nohup.log 2> /var/log/chronos/nohup.log < /dev/null &
          echo LIBPROCESS_IP=#{marathon_ip} nohup /opt/chronos/bin/start-chronos.bash --master #{"zk://"+ninfos[:zk].map{|zk| zk[:ip]+":2181"}.join(",")+"/mesos"} --zk_hosts #{"zk://"+ninfos[:zk].map{|zk| zk[:ip]+":2181"}.join(",")+"/mesos"} --http_port 8081 >| ~/chronos.runner
          SCRIPT
      end

      ## If you wanted use `.dockercfg` file
      ## Please place the file simply on this directory
      if File.exist?(".dockercfg")
        config.vm.provision :shell, :priviledged => true, :inline => <<-SCRIPT
          cp /vagrant/.dockercfg /root/.dockercfg
          chmod 600 /root/.dockercfg
          chown root /root/.dockercfg
          SCRIPT
      end
    end
  end
end
