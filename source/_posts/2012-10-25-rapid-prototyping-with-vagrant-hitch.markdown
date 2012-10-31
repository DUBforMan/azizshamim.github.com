---
layout: post
title: "Rapid prototyping with vagrant-hitch"
date: 2012-10-25 16:17
comments: true
categories: 
---
My former consulting partner [jfryman](http://github.com/jfryman) and [I](http://github.com/ashamim) had a few start-ups as clients who, inevitably, needed to deploy their application "by tuesday because we have a zillion new users".

The fast and furious nature of the start-up world and the consistent need to model new and relatively distinct environments in [Vagrant](http://vagrantup.com) lead to a pretty blatant bottleneck: the [Vagrantfile](http://vagrantup.com/v1/docs/vagrantfile.html).

Defining an environment with 20 different servers in some configuration that might overlap and might not is a complicated thing in the standard vagrant method:

    Vagrant::Config.run do |config|
      config.vm.define :web do |web_config|
        web_config.vm.box = "web"
        web_config.vm.network :hostonly, "10.1.1.1"
        web_config.vm.forward_port 80, 8080
        web_config.vm.provision :puppet do |puppet|
          puppet.module_path = "moduledir"
          puppet.manifest_file = "manifest/web_manifest.pp"
        end
      end

      config.vm.define :db do |db_config|
        db_config.vm.box = "db"
        db_config.vm.network :hostonly, "10.1.1.2"
        db_config.vm.forward_port 3306, 3306
        db_config.vm.provision :puppet do |puppet|
          puppet.module_path = "moduledir"
          puppet.manifest_file = "manifest/db_manifest.pp"
        end
      end
    end

Now do this 20 times! ZOMG!

So [jfryman](http://github.com/jfryman) and [I](http://github.com/azizshamim) came up with another way of doing this: [Hitch!](https://github.com/fup/vagrant-hitch)
<!--more-->
What hitch does is allow you to define an entire infrastructure as a YAML data block, making use of advanced YAML features like [relational trees](http://en.wikipedia.org/wiki/YAML#Relational_trees):

    default: &default                   # The default configuration for every box
      vbox: vbox-name
      orgname: vagrant.test
      puppet:
        manifest_file: site.pp
      ports:
        http:
          host: 10080
          guest: 80
        https:
          host: 10443
          guest: 443
        mysql:
          host: 13310
          guest: 3306
        tomcat:
          host: 18080
          guest: 8080
    test1:
      <<: *default
      vbox: vbox-dev-webserver          # here the default vbox is being overridden
    test2:
      <<: *default

What are some conventions we adopted in order to build/use [Hitch!](https://github.com/fup/vagrant-hitch)?

* The default config directory is **./config** relative to the Vagrantfile
* The node definition list is **nodes.yml**
* The provisioners use **provisioner_<name>.yml** to configure the provisioner
* The network is **:hostonly**
* The default manifest_file is in the manifest_path called **\<hostname\>.pp**

Lets take a more complicated data structure. We're modeling a simple 3 tier system with redundant databases for a startup called "SuperFastJellyfish:
{% gist 3988675 1_superfastjellyfish.yml %}
This configuration will load up a simple infrastructure of 3 webservers, 3 application servers and 2 database servers. Now lets try to get them slightly different:
{% gist 3988675 2_superfastjellyfish.yml %}
Now we've got different IP Addresses assigned to the nodes. Lets get a little more complexity:
{% gist 3988675 3_superfastjellyfish.yml %}

Now, using more YAML relational trees, we've defined some additional defaults and chef roles for those defaults, and even some distinct roles for the database servers.

Now lets take a look at the provisioner_chef.yml datafile and how it can be used:
{% gist 3988675 chef_provisioner.yml %}

This chef provisioner configuration defines the log level for running chef-solo, the environment (if you use them) and the default roles that are assigned to every system using this provisioner. Since *base* is in here, we could safely remove it from the nodes.yml file.

Now here's the environment:
{% gist 3988675 1_vagrant_status %}

You can do a similar thing with **provisioner_puppet.yml**. For an example, see the [example](https://github.com/fup/vagrant-hitch/tree/master/example) directory.

Future features that are under discussion are:

* Hosted Chef integration
* Full puppetmaster integration (run a puppetmaster inside a vagrant environment and make it a dependency)
* Bootstrapping for vagrant-hitch
* Full integration with Vagrant 2.0 (data driven infrastructure EVERYWHERE!!!)
