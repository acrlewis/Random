### Install Ruby and RVM and git Bosh-Lite
	brew tap pivotal/tap
	brew tap xoebus/homebrew-cloudfoundry

	brew install cask
	brew install git
	brew cask install virtualbox
	brew install ruby
	\curl -L https://get.rvm.io | bash -s stable
	rvm remove 1.9.3-p547
	
	rvm install 1.9.3-p547
	rvm use ruby-1.9.3-p547
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

    
    scripts/add-route
    

### Upload the Warden stemcell

A stemcell is a VM template with an embedded BOSH Agent. BOSH Lite uses the Warden CPI, so we need to use the Warden Stemcell which will be the root file system for all Linux Containers created by the Warden CPI.

### Download latest Warden stemcell

    
    wget http://bosh-jenkins-gems-warden.s3.amazonaws.com/stemcells/latest-bosh-stemcell-warden.tgz
    

### Upload the stemcell

    
    bosh upload stemcell latest-bosh-stemcell-warden.tgz

### Install [Spiff](https://github.com/cloudfoundry-incubator/spiff)

	brew install spiff
	
### Clone a copy of cf-release:
    
    cd ~/workspace
    git clone https://github.com/cloudfoundry/cf-release 
    cd ~/workspace/cf-release
    ./update
    git checkout v173
    

### Upload final release

    bosh upload release releases/cf-173.yml
    cd ~/workspace/bosh-lite
    ./update
    ./scripts/make_manifest_spiff

If you want to change the jobs properties for this bosh-lite deployment, e.g. number of nats servers, you can change it in the template located under cf-release/templates/cf-infrastructure-warden.yml.


### Deploy CF with bosh-lite

    bosh deployment manifests/cf-manifest.yml 
    bosh -n deploy
    
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
	git checkout v8	
	bosh upload release releases/cf-mysql-8.yml
	./bosh-lite/make_manifest_spiff_mysql
	
* vi the ~/workspace/cf-mysql-release/bosh-lite/manifests/cf-mysql-manifest.yml to change the following:
  stemcell:
    name: bosh-warden-boshlite-ubuntu-lucid-go_agent
    version: 60

	bosh deployment ~/workspace/cf-mysql-release/bosh-lite/manifests/cf-mysql-manifest.yml
	bosh -n deploy
	bosh run errand broker-registrar

### Create a MySQL service for pet clinic
	cf cs p-mysql 100mb-dev petclinic-mysql

### Deploy Additional Services to Local CloudFoundry Deployment

	cd ~/workspace
	git clone https://github.com/cloudfoundry-community/cf-services-contrib-release.git
	cd cf-services-contrib-release
	bundle install
	bosh upload release releases/cf-services-contrib-5.yml
	templates/make_manifest warden

* vi the tmp/contrib-services-warden-manifest.yml to change the following in three places:
  stemcell:
    name: bosh-warden-boshlite-ubuntu-lucid-go_agent
    version: 60

  Change 10.244.1 to 10.244.4 throughout

    bosh deployment tmp/contrib-services-warden-manifest.yml
	bosh -n deploy

	cf create-service-auth-token rabbitmq core c1oudc0w

	cf cs rabbitmq default test-rabbit

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
	bosh upload release releases/docker-4.yml

### Deployments

	➜  bosh deployments

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


