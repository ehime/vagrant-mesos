## Vagrant Multi-Mesosphere

Spin up your [Mesos](http://mesos.apache.org) cluster with [Vagrant](http://www.vagrantup.com)! (Both Virtualbox and AWS are supported.)

#### About

This spins up Mesos 0.22.1 cluster and also spins up a framework server node in which [Marathon](https://github.com/mesosphere/marathon) (0.8.2) and [Chronos](http://github.com/mesos/chronos) (2.1.0) are running.

Which means you can build your own __Mesos+Marathon+Chronos+Docker__ PaaS with `vagrant up`!!  Marathon works as distributed `init.d` and Chronos works as distributed `cron`!!  _If you wanted to deploy docker containers, please refer to the chapter "Deploy Docker Container with Marathon" in [this blog entry](http://frankhinek.com/deploy-docker-containers-on-mesos-0-20/)._

* [Mesos Cluster on VirtualBox](#clvb)
* [Mesos Cluster on EC2 (VPC)](#clec2)


The mesos installation is powered by Everpeace's Mesos chef cookbook.  Please see [everpeace/cookbook-mesos](http://github.com/everpeace/cookbook-mesos).

Base boxes used in `Vagrantfile`s are Mesos pre-installed boxes,
[ehime/mesos](https://vagrantcloud.com/ehime/boxes/mesos) shared on Vagrant Cloud.


#### Prerequisites

* Vagrant 1.6.5+: <http://www.vagrantup.com/>
* VirtualBox: <https://www.virtualbox.org/> (not required if you use ec2.)
* Vagrant plugins
    * [vagrant-omnibus](https://github.com/schisamo/vagrant-omnibus)
    * [vagrant-berkshelf](https://github.com/berkshelf/vagrant-berkshelf) (>=4.0.0) [__NOTE:__ To use berkself, you will have to install [ChefDK](https://downloads.chef.io/chef-dk/)]
    * [vagrant-hosts](https://github.com/adrienthebo/vagrant-hosts)
    * [vagrant-cachier](https://github.com/fgrehm/vagrant-cachier) (optional)
    * [vagrant-aws](https://github.com/mitchellh/vagrant-aws) (only if you use ec2.)

To install the above requirements you can simply run my [dependancies  script](https://github.com/ehime/vagrant-mesos/blob/master/vagrant_plugin_installer.sh)

<a name="clvb"></a>
#### Mesos Cluster on VirtualBox

##### Cluster Configuration
Cluster Configuration is defined at  `cluster.yml`.  You can edit the file to configure cluster settings.

```bash
## Mesos cluster configurations
mesos_version : 0.22.1

## The numbers of servers
##############################
zk_n          : 1    # hostname will be zk1, zk2, …
master_n      : 1    # hostname will be master1,master2,…
slave_n       : 1    # hostname will be slave1,slave2,…

## Memory and Cpus setting(only for virtualbox)
##########################################
zk_mem        : 256
zk_cpus       : 1
master_mem    : 256
master_cpus   : 1
slave_mem     : 512
slave_cpus    : 2

## private ip bases
## When ec2, this should be matched with
## private addresses defined by subnet_id below.
################################################
zk_ipbase     : "172.31.0."
master_ipbase : "172.31.1."
slave_ipbase  : "172.31.2."
```


##### Launch Cluster
This takes several minutes(10 to 20 min.).  It's time to go grabbing some coffee.

```bash
$ vagrant up
```

At default setting, after all the boxes are up, you can see services running at:

* [Mesos](https://github.com/apache/mesos) Web UI on: <http://172.31.1.11:5050>
* [Marathon](https://github.com/mesosphere/marathon) Web UI on: <http://172.31.3.11:8080>
* [Chronos](https://github.com/mesos/chronos) Web UI on: <http://172.31.3.11:8081>

##### Destroy Cluster
this operations all VM instances forming the cluster.

```bash
$ vagrant destroy
```

<a name="clec2"></a>
#### Mesos Cluster on EC2 (VPC)

Because we assign private IP addreses to VM instances, this Vagrantfile requires Amazon VPC (you'll have to set subnet_id and security grooups both of which associates to the same VPC instance).

_Note: Using default VPC is highly recommended.  If you used non-default VPC, you should make sure to activate "DNS resolution" and "DNS hostname" feature in the VPC._

##### Cluster Configuration
You have to configure some additional stuffs in `cluster.yml` which are related to EC2.  Please note that

* `subnet_id` should be a VPC subnet
* `security_groups` should be ones associated to the VPC instance.
	* `security_groups` should allow accesses to ports 22(SSH), 2181(zookeeper) and 5050--(mesos).

```bash
(cont.)
## EC2 Configurations
## please choose one region from
## ["ap-northeast-1", "ap-southeast-1", "eu-west-1", "sa-east-1",
##  "us-east-1", "us-west-1", "ap-southeast-2", "us-west-2"]
## NOTE: if you used non-default vpc, you should make sure that
##       limit of the elastic ips is no less than (zk_n + master_n + slave_n).
## In EC2, the limit default is 5.
########################
access_key_id         : EDIT_HERE
secret_access_key     : EDIT_HERE
default_vpc           : true              # default vpc or not.
subnet_id             : EDIT_HERE         # VPC subnet id
security_groups       : ["EDIT_HERE"]     # array of VPN security groups. e.g. ['sg*** ']
keypair_name          : EDIT_HERE
ssh_private_key_path  : EDIT_HERE
region                : EDIT_HERE

## see http://aws.amazon.com/ec2/instance-types/#selecting-instance-types
zk_instance_type      : m1.small
master_instance_type  : m1.small
slave_instance_type   : m1.small
```

##### Launch Cluster
After editing configuration is done, you can just hit regular command.

```bash
$ vagrant up --provider=aws --no-parallel
```

_NOTE: `--no-parallel` is highly recommended because vagrant-berkshelf plugin is prone to failure in parallel provisioning._

After instances are all up, you can see

* [Mesos](https://github.com/apache/mesos) Web UI on:           `http://#_public_dns_of_the_master_N_#:5050`
* [Marathon](https://github.com/mesosphere/marathon) Web UI on: `http://#_public_dns_of_marathon_#:8080`
* [Chronos](https://github.com/mesos/chronos) Web UI on:        `http://#_public_dns_of_chronos#:8081`

if everything went well.

_Tips: you can get public dns of the vms by:_

```bash
$ vagrant ssh master1 -- 'echo http://`curl --silent http://169.254.169.254/latest/meta-data/public-hostname`:5050'
http://ec2-54-193-24-154.us-west-1.compute.amazonaws.com:5050
```

If you wanted to make sure that a specific master (e.g. `master1`) will be the initial leader, you can control the order of spinning up VMs like below.

```bash
# spin up an zookeeper ensemble
$ vagrant up --provider=aws /zk/

# spin up master1. master1 will be an initial leader
$ vagrant up --provider=aws master1

# spin up remained masters
$ vagrant up --provider=aws /master[2-9]/

# spin up slaves
$ vagrant up --provider=aws /slave/

# spin up marathon
$ vagrant up --provider=aws marathon
```

##### Stop your Cluster
```bash
$ vagrant halt
```

##### Resume your Cluster
```bash
$ vagrant reload --provision
```

##### Destroy your Cluster
This operations terminates all VMs instances forming the cluster.

```bash
$ vagrant destroy
```

<a name="ctst1"></a>
#### Testing and upgrading binary packages for compatibility

###### Basic command for tearing down a stack

In order to test a clean build run the following, this will completely tear down the stack and replay it.

```bash
vagrant destroy --force ; vagrant up --provider=virtualbox
```

If you would like to remove and reissue only a portion of the stack you can use the folllowing as well.

```bash
vagrant destroy /zk/ --force ; vagrant up /zk/ --provider=virtualbox
```

This will destroy and rebuild the zookeeper ensemble. Please note though that when removing portions of the stack that you will need to re-apply any hook or registration that is required by the infrastructure.
