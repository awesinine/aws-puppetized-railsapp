# aws-puppetized-railsapp
Interested in getting a small scale puppetized rails app with passenger setup within AWS?  Here's a guide!
_I'm editing this on the fly as I go with the aim to provide clear documentation._

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
Sorry, you're on your own.  You'll need a credit card, an e-mail address and some ambition.

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

Create a key or use an existing.  Save that pem somewhere safe 
launch

check permissions on your key
chmod 0400

takes ~ 5 mins for everything to get launched and all that Jazz
go download your [enterprise puppet master](https://puppetlabs.com/download-puppet-enterprise-welcome)

Make note of your public DNS
[no public dns?](http://stackoverflow.com/questions/20941704/ec2-instance-has-no-public-dns)

**secure copy the puppet enterprise over to aws:** `scp -i example.pem ./puppet-enterprise-2015.2.3-el-7-x86_64.tar.gz root@publicDNS:/home/root`

...15 mins later...

**ssh in**
`ssh -i example.pem root@publicDNS`

**update / upgrade**
`yum update -y`
`yum upgrade -y`

* untar that badboy
  * `tar -xzvf puppet-enterprise-2015.2.3-el-7-x86_64.tar.gz`
  * `cd puppet-enterprise-2015.2.3-el-7-x86_64`

* run the puppet installer
./puppet-enterprise-installer -s master.answers (saves an answer file that will allow you to automate the installation later!)
Install puppet master?  Yes
install puppetdb console to this node?  Yes
puppet master cert dns : yes (but we change this later)
dns alias: all the names by which this node can be known
postgres? yes
443
admin login, password
smtp host: not needed choose localhost
vendor packages yes
accept symbolic links yes
Cloud provisioner? Yes

backticks execute commands from the shell!  Replace master hostname certname with `curl http://169.254.169.254/latest/meta-data/public-hostname`

add to dnshost name (don't replace)

install
./puppet-enterprise-installer -a master.answers

setup firewall to let port 443 get and send traffic!
service iptables stop (temporary for testing, you'll want to go back and setup iptables correctly)

check for nodes puppet agent -t
setup path or alias for that command
/opt/puppet/bin/puppet agent -t (performs an agent checkin with the node, targets master!)

NEXT STEPS

spin up node automagically

phone home (contact master) <3
