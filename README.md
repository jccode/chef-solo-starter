Chef solo starter
=================

## Prerequisites ##

1. Make sure you already install vagrant.
2. Below vagrant plugins are also required.

        $ vagrant plugin install vagrant-cachier
        $ vagrant plugin install vagrant-omnibus
        $ vagrant plugin install vagrant-berkshelf
        
3. Make sure chef is installed
4. As well as knifo-solo, berkshelf

        $ gem install knife-solo
        $ gem install berkshelf

## Steps ##

### Initialize project structure ###

1. Create project directory.

        $ mkdir <project-dir>
        $ cd <project-dir>

2. Init vagrant, which is used for local testing.

        $ vagrant init hashicorp/precise64

   This will create a `Vagrantfile` in your project folder. Open it and edit:


        Vagrant.configure("2") do |config|
          config.vm.box = "hashicorp/precise64"

          config.cache.scope = :box
          config.omnibus.chef_version = :latest
          config.berkshelf.enabled = true

          VAGRANT_JSON = JSON.parse(Pathname(__FILE__).dirname.join('nodes', 'vagrant.json').read)
          config.vm.provision "chef_solo" do |chef|
            #chef.installer_download_path = "file:///Users/IT/soft/chef_client/chef_13.0.118-1_amd64.deb"
            #chef.install = false

            chef.cookbooks_path = ["site-cookbooks", "cookbooks"]
            chef.roles_path = "roles"
            chef.data_bags_path = "data_bags"
            chef.nodes_path = "nodes"
            chef.provisioning_path = "/tmp/vagrant-chef"

            chef.run_list = VAGRANT_JSON.delete('run_list')
            chef.json = VAGRANT_JSON
          end

        end

3. Init cookbooks folder structure by `knife solo`

        $ knife solo init .

   After that, our directory looks like, 

        .
        ├── Berksfile
        ├── README.md
        ├── Vagrantfile
        ├── cookbooks
        ├── data_bags
        ├── environments
        ├── nodes
        ├── roles
        └── site-cookbooks

4. Create our project cookbook in `site-cookbooks`. Let's take "my-project" as example.

        $ knife cookbook create -o site-cookbooks my-project

   After this command, our project directory looks like.

        .
        ├── Berksfile
        ├── README.md
        ├── Vagrantfile
        ├── cookbooks
        ├── data_bags
        ├── environments
        ├── nodes
        ├── roles
        └── site-cookbooks
            └── my-project
                ├── CHANGELOG.md
                ├── README.md
                ├── attributes
                ├── definitions
                ├── files
                │   └── default
                ├── libraries
                ├── metadata.rb
                ├── providers
                ├── recipes
                │   └── default.rb
                ├── resources
                └── templates
                    └── default

Things below are about how to cook you cookbooks.

### Cook your cookbook ###

Suppose we want to install some useful tools in our enviroment. And let's take this as example. Belows are things we need to do.

- Update apt repository before we install package. We can achieve this by add `apt` cookbook dependency.
- Sometimes, the official `source.list` are too slow. We can use some faster one to replace it.
- After that, write a recipe to install our favorite tools.

1. Add `'apt', '~> 6.1.0'` cookbook dependency.

  - Edit `site-cookbooks/my-project/metadata.rb`

        depends 'apt', '~> 6.1.0'

  - Edit `Berksfile`

        cookbook 'apt', '~> 6.1.0'

  - Update berks dependencies

        $ berks install

2. Create a `sources.list.erb` in `site-cookbooks/my-project/templates` folder, which defines the new `sources.list`.

        deb http://mirrors.aliyun.com/ubuntu/ precise main restricted universe multiverse
        deb http://mirrors.aliyun.com/ubuntu/ precise-security main restricted universe multiverse
        deb http://mirrors.aliyun.com/ubuntu/ precise-updates main restricted universe multiverse
        deb http://mirrors.aliyun.com/ubuntu/ precise-proposed main restricted universe multiverse
        deb http://mirrors.aliyun.com/ubuntu/ precise-backports main restricted universe multiverse
        deb-src http://mirrors.aliyun.com/ubuntu/ precise main restricted universe multiverse
        deb-src http://mirrors.aliyun.com/ubuntu/ precise-security main restricted universe multiverse
        deb-src http://mirrors.aliyun.com/ubuntu/ precise-updates main restricted universe multiverse
        deb-src http://mirrors.aliyun.com/ubuntu/ precise-proposed main restricted universe multiverse
        deb-src http://mirrors.aliyun.com/ubuntu/ precise-backports main restricted universe multiverse

3. Edit `recipes/default.rb`.

        # update source.list
        # sudo mv /etc/apt/source.list /etc/apt/source.list.orig

        bash "backup origin sources.list" do
          code <<-EOL
          mv /etc/apt/sources.list /etc/apt/sources.list.orig 
          EOL
        end

        # chef generate template . source.list
        template '/etc/apt/sources.list' do
          source 'sources.list.erb'
        end


        # apt-update
        include_recipe 'apt::default'

        # install your favorite packages
        node['pkgs'].each do |pkg|
          package pkg
        end

4. Well, `pkgs` above are the packages we want to install, which are define in `attributes/default.rb`. Edit this file (create it if it's not exist).

        default['pkgs'] = %w(git emacs)

5. Create `vagrant.json` in `nodes`, define the bootstrap entry.

        {
            "run_list": [
                "recipe[my-project]"
            ]
        }

6. Our cookbook is already done. Project directory looks like:

        .
        ├── Berksfile
        ├── Berksfile.lock
        ├── README.md
        ├── Vagrantfile
        ├── cookbooks
        ├── data_bags
        ├── environments
        ├── nodes
        │   └── vagrant.json
        ├── roles
        └── site-cookbooks
            └── my-project
                ├── CHANGELOG.md
                ├── README.md
                ├── attributes
                │   └── default.rb
                ├── definitions
                ├── files
                │   └── default
                ├── libraries
                ├── metadata.rb
                ├── providers
                ├── recipes
                │   └── default.rb
                ├── resources
                └── templates
                    ├── default
                    └── sources.list.erb

### Bootstrap our server ###

We can take it as "vagrant" project, use `vagrant up` to bootstrap it.
Also we can treat it as an chef project, use `knife solo` to bootstrap it.

#### By vagrant ####

        $ vagrant up --provision

#### By knife solo ####

1. Create a `fabfile.py` in project folder, to shorten our typings.


        from fabric.api import env, local, task, run, put, sudo

        def remote_server():
            env.user = 'root'
            env.hosts = [ '120.25.145.61' ]   #
            env.port = 22
            env.key_filename = '~/.ssh/id_rsa'

        def vagrant():
            """USAGE:
            fab vagrant uname

            Note that the command to run Fabric might be different on different
            platforms.
            """
            # change from the default user to 'vagrant'
            env.user = 'vagrant'

            # find running VM (assuming only one is running)
            result = local('vagrant global-status | grep running', capture=True)
            machineId = result.split()[0]

            result = local('vagrant ssh-config {} | grep Port'.format(machineId), capture=True)
            env.port = result.split()[1]
            # env._host = "127.0.0.1:{}".format(env.port)
            env.hosts = [ "127.0.0.1" ]

            # use vagrant ssh key for the running VM
            result = local('vagrant ssh-config {} | grep IdentityFile'.format(machineId), capture=True)

            env.key_filename = result.split()[1]

        @task
        def uname():
            run("uname -a")
            run("lsb_release -a")

        @task
        def prepare():
            local("knife solo prepare {user}@{hosts[0]} -p {port} -i {key_filename}".format(**env))

        @task
        def cook():
            local("knife solo cook {user}@{hosts[0]} -p {port} -i {key_filename}".format(**env))

        @task
        def bootstrap():
            local("knife solo bootstrap {user}@{hosts[0]} -p {port} -i {key_filename}".format(**env))


        # Setup env
        vagrant()


2. `fab cook` to bootstrap our server.

- if the run list is empty, then we need to update `127.0.0.1.json` in `nodes`.

        {
          "run_list": [
              "recipe[my-project]"
          ],
          "automatic": {
            "ipaddress": "127.0.0.1"
          }
        }

- if the chef client isn't install on the server, we need to `fab prepare` to install chef client first.

## Tricks ##

