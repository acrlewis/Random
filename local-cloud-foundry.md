### Install Ruby and RVM and git Bosh-Lite
	brew install cask
	brew install git
	brew install vagrant
	brew cask install virtualbox
	brew install ruby
	\curl -L https://get.rvm.io | bash -s stable
	sudo rvm remove 1.9.3
	
	rvm install 1.9.3
	rvm use ruby-1.9.3
	mkdir ~/workspace
	cd workspace
	git clone https://github.com/cloudfoundry/bosh-lite.git
	cd bosh-lite
	bundle
	gem install bosh_cli
	

### Start Vagrant from the base directory of this repository. This uses the Vagrantfile.

    
    vagrant up
    

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
    ./scripts/make_manifest_spiff

If you want to change the jobs properties for this bosh-lite deployment, e.g. number of nats servers, you can change it in the template located under cf-release/templates/cf-infrastructure-warden.yml.


### Deploy CF with bosh-lite

    bosh deployment manifests/cf-manifest.yml 
    # This will be done for you by make_manifest_spiff
    bosh deploy
    #enter yes to confirm
    
### Install CLoudFoundry CLI and login
	
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
	
vi the ~/workspace/cf-mysql-release/bosh-lite/manifests/cf-mysql-manifest.yml to change the following:
  stemcell:
    name: bosh-warden-boshlite-ubuntu-lucid-go_agent
    version: 60

	bosh deployment ~/workspace/cf-mysql-release/bosh-lite/manifests/cf-mysql-manifest.yml
	bosh deploy
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

vi the tmp/contrib-services-warden-manifest.yml to change the following in three places:
  stemcell:
    name: bosh-warden-boshlite-ubuntu-lucid-go_agent
    version: 60

	bosh -n deploy

	cf create-service-auth-token rabbitmq core c1oudc0w

	cf cs rabbitmq default test-rabbit
	
For some reason, when I rebuild everything and try to push an app, I get an error about a failed buildpack. When I reset the deployment to CF, the app push works

	cf apps
	cf delete <app you deployed and it failed>
	cd ~/workspace/bosh-lite
	bosh deployment manifests/cf-manifest.yml

### If you run into trouble

Sometimes, you have to start over with the local cf install. In that case, this has worked for me:
	
	cd ~/workspace/bosh-lite
	vagrant destroy -f
	git pull
	vagrant up
	scripts/add-route
    bosh upload stemcell latest-bosh-stemcell-warden.tgz
    cd ~/workspace/cf-release
    bosh upload release releases/cf-173.yml
    cd ~/workspace/bosh-lite
    ./scripts/make_manifest_spiff
    bosh deployment manifests/cf-manifest.yml 
    bosh deploy
    #enter yes to confirm
    cf api http://api.10.244.0.34.xip.io --skip-ssl-validation
	cf login -u admin -p admin
	#hit enter to skip Org selection
	cf create-org me
	cf target -o me
	cf create-space development
	cf target -s development

Then follow the instructions starting with 'Deploy Additional Services to Local CloudFoundry Deployment'


