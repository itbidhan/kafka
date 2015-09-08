System Integration & Performance Testing
========================================

This directory contains Kafka system integration and performance tests. 
[ducktape](https://github.com/confluentinc/ducktape) is used to run the tests.  
(ducktape is a distributed testing framework which provides test runner, 
result reporter and utilities to pull up and tear down services.) 

Local Quickstart
----------------
This quickstart will help you run the Kafka system tests on your local machine.
For a tutorial on how to setup and run the Kafka system tests, see 
https://cwiki.apache.org/confluence/display/KAFKA/tutorial+-+set+up+and+run+Kafka+system+tests+with+ducktape

* Install Virtual Box from [https://www.virtualbox.org/](https://www.virtualbox.org/) (run `$ vboxmanage --version` to check if it's installed).
* Install Vagrant >= 1.6.4 from [http://www.vagrantup.com/](http://www.vagrantup.com/) (run `vagrant --version` to check if it's installed).
* Install Vagrant Plugins:

        # Required
        # Note that vagrant-hostmanager v1.6.0 and up breaks our Vagrant scripts
        $ vagrant plugin install vagrant-hostmanager --plugin-version 1.5.0
        $ vagrant plugin install vagrant-cachier

* Build a specific branch of Kafka
       
        $ cd kafka
        $ git checkout $BRANCH
        $ gradle
        $ ./gradlew jar
      
* Setup a testing cluster with Vagrant. Configure your Vagrant setup by creating the file 
   `Vagrantfile.local` in the directory of your Kafka checkout. For testing purposes,
  `num_brokers` and `num_kafka` should be 0, and `num_workers` should be set high enough
  to run all of you tests. An example resides in kafka/vagrant/system-test-Vagrantfile.local

        # Example Vagrantfile.local for use on local machine
        # Vagrantfile.local should reside in the base Kafka directory
        num_zookeepers = 0
        num_kafka = 0
        num_workers = 9

* Bring up the cluster (note that the initial provisioning process can be slow since it involves
installing dependencies and updates on every vm.):

        $ vagrant up

* Install ducktape:
       
        $ pip install ducktape

* Run the system tests using ducktape:

        $ cd tests
        $ ducktape kafkatest/tests

* If you make changes to your Kafka checkout, you'll need to rebuild and resync to your Vagrant cluster:

        $ cd kafka
        $ ./gradlew jar
        $ vagrant rsync # Re-syncs build output to cluster
        
EC2 Quickstart
--------------
This quickstart will help you run the Kafka system tests on EC2. In this setup, all logic is run
on EC2 and none on your local machine. 

There are a lot of steps here, but the basic goals are to create one distinguished EC2 instance that
will be our "test driver", and to set up the security groups and iam role so that the test driver
can create, destroy, and run ssh commands on any number of "workers".

As a convention, we'll use "kafkatest" in most names, but you can use whatever name you want. 

Preparation
-----------
In these steps, we will create an IAM role which has permission to create and destroy EC2 instances, 
set up a keypair used for ssh access to the test driver and worker machines, and create a security group to allow the test driver and workers to all communicate via TCP.

* [Create an IAM role](http://docs.aws.amazon.com/IAM/latest/UserGuide/Using_SettingUpUser.html#Using_CreateUser_console). We'll give this role the ability to launch or kill additional EC2 machines.
 - Create role "kafkatest-master"
 - Role type: Amazon EC2
 - Attach policy: AmazonEC2FullAccess (this will allow our test-driver to create and destroy EC2 instances)
 
* If you haven't already, [set up a keypair to use for SSH access](http://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-key-pairs.html). For the purpose
of this quickstart, let's say the keypair name is kafkatest, and you've saved the private key in kafktest.pem

* Next, create a security group called "kafkatest". 
 - After creating the group, inbound rules: allow SSH on port 22 from anywhere; also, allow access on all ports (0-65535) from other machines in the kafkatest group.

Create the Test Driver
----------------------
* Launch a new test driver machine 
 - OS: Ubuntu server is recommended
 - Instance type: t2.medium is easily enough since this machine is just a driver
 - Instance details: Most defaults are fine.
 - IAM role -> kafkatest-master
 - Tagging the instance with a useful name is recommended. 
 - Security group -> 'kafkatest'
  

* Once the machine is started, upload the SSH key to your test driver:

        $ scp -i /path/to/kafkatest.pem \
            /path/to/kafkatest.pem ubuntu@public.hostname.amazonaws.com:kafkatest.pem

* Grab the public hostname/IP (available for example by navigating to your EC2 dashboard and viewing running instances) of your test driver and SSH into it:

        $ ssh -i /path/to/kafkatest.pem ubuntu@public.hostname.amazonaws.com
        
Set Up the Test Driver
----------------------
The following steps assume you have ssh'd into
the test driver machine.

* Start by making sure you're up to date, and install git and ducktape:

        $ sudo apt-get update && sudo apt-get -y upgrade && sudo apt-get install -y git
        $ pip install ducktape

* Get Kafka:

        $ git clone https://git-wip-us.apache.org/repos/asf/kafka.git kafka
        
* Install some dependencies:

        $ cd kafka
        $ kafka/vagrant/aws/aws-init.sh
        $ . ~/.bashrc

* An example Vagrantfile.local has been created by aws-init.sh which looks something like:

        # Vagrantfile.local
        ec2_instance_type = "..." # Pick something appropriate for your
                                  # test. Note that the default m3.medium has
                                  # a small disk.
        num_zookeepers = 0
        num_kafka = 0
        num_workers = 9
        ec2_keypair_name = 'kafkatest'
        ec2_keypair_file = '/home/ubuntu/kafkatest.pem'
        ec2_security_groups = ['kafkatest']
        ec2_region = 'us-west-2'
        ec2_ami = "ami-29ebb519"

* Start up the instances (note we have found bringing up machines in parallel can cause errors on aws):

        $ vagrant up --provider=aws --no-provision --no-parallel && vagrant provision

* Now you should be able to run tests:

        $ cd kafka/tests
        $ ducktape kafkatest/tests

* To halt your workers without destroying persistent state, run `vagrant halt`. Run `vagrant destroy -f` to destroy all traces of your workers.
