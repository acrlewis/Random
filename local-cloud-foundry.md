### Install Ruby and RVM and git Bosh-Lite
	brew tap pivotal/tap

	brew install cask
	brew install git
	brew cask install vagrant
	brew install ruby
	\curl -L https://get.rvm.io | bash -s stable
	mkdir ~/workspace
	cd workspace
	git clone https://github.com/cloudfoundry/bosh-lite.git
	cd bosh-lite
	bundle
	gem install bosh_cli
	bundle install
	bundle update


### Start Vagrant from the base directory of this repository. This uses the Vagrantfile.


    vagrant up --provider vmware_fusion


### Target the BOSH Director and login with admin/admin.


    bosh target 192.168.50.4 lite
    #Your username: admin
    #Enter password: admin


Add a set of route entries to your local route table to enable direct Warden container access every time your networking gets reset (e.g. reboot or connect to a different network). Your sudo password may be required.


    bin/add-route


### Clone a copy of cf-release:

    cd ~/workspace
    git clone https://github.com/cloudfoundry/cf-release
    cd ~/workspace/cf-release
		rvm install ruby-1.9.3-p547
    ./update
		bundle install


### Deploy CF with bosh-lite

	cd ~/workspace/bosh-lite
	./bin/provision_cf


### Install CloudFoundry CLI and login

	brew install cloudfoundry-cli
	cf api http://api.10.244.0.34.xip.io --skip-ssl-validation
	cf login -u admin -p admin

	cf create-org me
	cf target -o me
	cf create-space development
	cf target -s development

### Deploy MySQL Services to local CloudFoundry Deployment

	cd ~/workspace
	git clone https://github.com/cloudfoundry/cf-mysql-release.git
	cd ~/workspace/cf-mysql-release
	./update
	bosh upload release releases/cf-mysql-15.yml
	./bosh-lite/make_manifest_spiff_mysql

change "Latest" to 389 for the stemcell version

	nano ~/workspace/cf-mysql-release/bosh-lite/manifests/cf-mysql-manifest.yml

	bosh deployment ~/workspace/cf-mysql-release/bosh-lite/manifests/cf-mysql-manifest.yml
	bosh -n deploy
	bosh run errand broker-registrar

Create a MySQL service for pet clinic

	cf cs p-mysql 100mb-dev petclinic-mysql

### Deploy Additional Services to Local CloudFoundry Deployment

	cd ~/workspace
	git clone https://github.com/cloudfoundry-community/cf-services-contrib-release.git
	cd cf-services-contrib-release
	bundle install
	bosh upload release releases/cf-services-contrib-6.yml
	templates/make_manifest warden

change "Latest" to 389 for the stemcell version
chang 10.244.1. to 10.244.3.

	nano tmp/contrib-services-warden-manifest.yml

  bosh deployment tmp/contrib-services-warden-manifest.yml
	bosh -n deploy

	cf create-service-auth-token rabbitmq core c1oudc0w
	cf create-service-auth-token postgresql core c1oudc0w
	cf create-service-auth-token redis core c1oudc0w
	cf create-service-auth-token mongodb core c1oudc0w
	cf create-service-auth-token elasticsearch core c1oudc0w
	cf create-service-auth-token swift core c1oudc0w

	cf cs rabbitmq default test-rabbit

### Deploy Admin-UI

	bosh upload release https://admin-ui-boshrelease.s3.amazonaws.com/boshrelease-admin-ui-3.tgz
	git clone https://github.com/cloudfoundry-community/admin-ui-boshrelease.git
	cd admin-ui-boshrelease
	git checkout v3
	./make_manifest warden
	bosh -n deploy
	cf create-service-auth-token admin-ui core c1oudc0w

Now you can browse to http://admin.10.244.0.34.xip.io and login with your cloud foundry admin user.

### Deploy Riak-CS
	cd ~/workspace
	git clone https://github.com/cloudfoundry/cf-riak-cs-release.git
	rvm install ruby-2.0.0-p353
	rvm use ruby-2.0.0-p353
	cd ~/workspace/cf-riak-cs-release
	gem install bosh_cli
	./update
	git checkout v4
	bosh upload release releases/cf-riak-cs-4.yml
	./bosh-lite/make_manifest
	bosh upload release bosh-lite/manifests/cf-riak-cs-manifest.yml
	bosh -n deploy
	bosh run errand broker-registrar

### Deploy Docker Services Capabilities to Local CloudFoundry Deployment (WORK IN PROGRESS)

	git clone https://github.com/cf-platform-eng/docker-boshrelease.git
	cd docker-boshrelease
	wget https://s3.amazonaws.com/bosh-jenkins-artifacts/bosh-stemcell/aws/light-bosh-stemcell-2624-aws-xen-ubuntu-trusty-go_agent.tgz
	bosh upload stemcell light-bosh-stemcell-2624-aws-xen-ubuntu-trusty-go_agent.tgz
	bosh upload release releases/docker-9.yml

### Deployments

	bosh deployments

	+---------------------+-----------------------+-----------------------------------------------+
	| Name                | Release(s)            | Stemcell(s)                                   |
	+---------------------+-----------------------+-----------------------------------------------+
	| cf-riak-cs          | cf-riak-cs/4          | bosh-warden-boshlite-ubuntu-lucid-go_agent/60 |
	+---------------------+-----------------------+-----------------------------------------------+
	| cf-services-contrib | cf-services-contrib/5 | bosh-warden-boshlite-ubuntu-lucid-go_agent/60 |
	+---------------------+-----------------------+-----------------------------------------------+
	| cf-warden           | cf/176                | bosh-warden-boshlite-ubuntu-lucid-go_agent/60 |
	+---------------------+-----------------------+-----------------------------------------------+
	| cf-warden-mysql     | cf-mysql/8            | bosh-warden-boshlite-ubuntu-lucid-go_agent/60 |
	+---------------------+-----------------------+-----------------------------------------------+

### Marketplace of Services

	➜  cf marketplace
	Getting services from marketplace in org me / space development as admin...
	OK

	service      plans       description
	mongodb      default     MongoDB NoSQL database
	p-mysql      100mb-dev   A MySQL service for application development and testing
	p-riakcs     developer   An S3-compatible object store based on Riak CS
	postgresql   default     PostgreSQL database
	rabbitmq     default     RabbitMQ message queue
	redis        default     Redis key-value store


### If you run into trouble

Sometimes you need to restart the director

	vagrant ssh -c "sudo sv restart director"
