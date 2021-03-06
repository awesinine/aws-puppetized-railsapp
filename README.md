# aws-puppetized-railsapp
Interested in getting a small scale puppetized rails app with passenger setup within AWS?  Here's a guide!

**There are some constraints for this project:**
 
1. Puppet Master must run on *Apache*, _not the built in server_
2. Our webserver node must run *Apache*, passenger, and a ruby on rails application
3. _We gotta hit it like we mean it when we're finished_

##I'll be working in steps from this guide
[Configuring a Puppet Master Server with Passenger and Apache](https://docs.puppetlabs.com/guides/passenger.html)

###Technology stack
* Infrastructure: AWS, EC2
* Operating system: CentOS 6.5 (AMI)
* Node 1: puppet master
* Node 2: webserver
* Apache
* Passenger
* Ruby on Rails : hello world

###AWS account setup
Sorry, you're on your own.  You'll need a :credit_card:, an :e-mail: and some _ambition_.

###AWS Puppet Master Setup:
* Select EC2
* Select a Region near you
* Select Community AMI
  * CentOS6.5
  * _I Went with T1.Large for my puppet master_
* Storage => 30 gigs `HEY: Note that the puppet tutorial says 100gigs`
* Tagging => `server : puppet-master`
* Security Group : `name`
  * Ports:
    * **SSH**       :  `22 : 0.0.0.0/0`
    * **HTTPS**     :  `443 : 0.0.0.0/0 : anywhere`
    * **Traffic**   :  `80, 8080 : 0.0.0.0/0`
    * **Active MQ** :  `61613 : 0.0.0.0/0  `
    * **Puppet**    :  `8140: 0.0.0.0/0`
    * **Pupper web**:  `3000: 0.0.0.0/0`  <--- you'll diable this later

Create a key or use an existing one if you have it.  Save that pem somewhere safe! 

launch your instance

check permissions on your key: if not 0400, then change permissions: `chmod 0400`

takes ~ 5 mins for everything to get launched and all that Jazz
go download your [enterprise puppet master](https://puppetlabs.com/download-puppet-enterprise-welcome)

Make note of your public DNS
[no public dns?](http://stackoverflow.com/questions/20941704/ec2-instance-has-no-public-dns)

**secure copy the puppet enterprise over to aws:** `scp -i example.pem ./puppet-enterprise-2015.2.3-el-7-x86_64.tar.gz root@publicDNS:/home/root`
**secure copy your pem over to aws:**
scp -i example.pem example.pem root@publicDNS:/home/`

**ssh in**
`ssh -i example.pem root@publicDNS`

**update / upgrade**
`yum upgrade -y`

* untar that badboy
  * `tar -xzvf puppet-enterprise-2015.2.3-el-7-x86_64.tar.gz`
  * `cd puppet-enterprise-2015.2.3-el-7-x86_64`

* run the puppet installer method 1 : installer
  * `./puppet-enterprise-installer`
  * go to the web console at https://name:3000
  * configure via web gui

* run the puppet installer method 2 : answers file
  * make some edits to the answer file: 
  * **replace** _hostnames_ which refer to the internal aws name with `curl http://169.254.169.254/latest/meta-data/public-hostname`
  * _this is an aws endpoint which will give you the hostname of the box_
  * **replace** _redacted_ with matching passwords
  * `./puppet-enterprise-installer -a answer.file`

_EITHER PATH_: Backup your answers file `cp /etc/puppetlabs/installer/answers.install /etc/puppetlabs/installer/answers.config`

puppet resource group puppet ensure=present
puppet resource user puppet ensure=present gid=puppet

[setup firewall](https://access.redhat.com/documentation/en-US/Red_Hat_Enterprise_Linux/3/html/Security_Guide/s1-firewall-ipt-basic.html) to let port 443 get and send traffic!

* `service iptables stop`
* `iptables -A OUTPUT -p tcp -m tcp --dport 443 -j ACCEPT`
* `iptables -A INPUT -p tcp -m tcp --dport 443 -j ACCEPT`
* `service iptables start`

check in master as a node
`/opt/puppetlabs/bin/puppet agent -t`

###AWS Puppet Node Setup:
* Select EC2
* Select a Region near you
* Select Community AMI
  * CentOS6.5
  * _I Went with T1.Large for my puppet master_
* Storage => 30 gigs `HEY: Note that the puppet tutorial says 100gigs`
* Tagging => `server : webserver`
* Security Group : `name-webserver`
  * Ports:
    * **SSH**       :  `22 : 0.0.0.0/0`
    * **Traffic**   :  `80, 8080 : 0.0.0.0/0`
    * **Active MQ** :  `61613 : 0.0.0.0/0  `
    * **Puppet**    :  `8140: 0.0.0.0/0`

Create a key or use an existing one if you have it.  Save that pem somewhere safe! 

launch your instance

/opt/puppetlabs/bin/puppet cert list
yum install epel-release -y
sudo yum install httpd httpd-devel mod_ssl ruby-devel rubygems gcc -y
gem install rack passenger
yum install gcc-c++
yum install libcurl-devel
passenger-install-apache2-module

apache edit conf???
LoadModule passenger_module /usr/lib/ruby/gems/1.8/gems/passenger-5.0.21/buildout/apache2/mod_passenger.so
<IfModule mod_passenger.c>
  PassengerRoot /usr/lib/ruby/gems/1.8/gems/passenger-5.0.21
  PassengerDefaultRuby /usr/bin/ruby
</IfModule>

mkdir -p /usr/share/puppet/rack/puppetmasterd
cd /usr/share/puppet/rack/
wget "https://raw.githubusercontent.com/puppetlabs/puppet/stable/ext/rack/config.ru" config.ru
mkdir /usr/share/puppet/rack/puppetmasterd/public /usr/share/puppet/rack/puppetmasterd/tmp
cp /usr/share/puppet/rack/config.ru /usr/share/puppet/rack/puppetmasterd/
chown puppet:puppet /usr/share/puppet/rack/puppetmasterd/config.ru
