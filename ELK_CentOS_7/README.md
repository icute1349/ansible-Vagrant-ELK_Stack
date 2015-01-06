<h1>ELK Stack on CentOS 7 x86 64</h1>
<h3>Description:</h3>
<p>An ELK stack provisioned with Vagrant and Ansible.  Below are all my notes from this project.  These are all the steps I went through to build this stack manually from the command line before coverting to Ansible.</p>

<h2>Prereqs</h2>
<p>TODO: include link to CentOS_7 box I am using.</p>
<ol>
<li>Create a Vagrantfile in the root of your project directory:
$ vagrant init <box_name></li>
<li>Configure the Vagrantfile.</li>
<li>Create Ansible playbook.yml and ansible.cfg files in the root directory</li>
</ol>

<h3>Install Java 7</h3>
<p>1. Install OpenJDK 7:
	$ sudo yum install -y java-1.7.0-openjdk
</p>

<h3>Install Elasticsearch 1.1.1</h3>
<p>1. Import the Elasticsearch public GPG key into rpm:
$ sudo rpm --import http://packages.elasticsearch.org/GPG-KEY-elasticsearch</p>
<p>2. Create a new yum repository file for Elasticsearch:
$ sudo vi /etc/yum.repos.d/elasticsearch.repo</p>

<p>Add the following:
[elasticsearch-1.1]
name=Elasticsearch repository for 1.1.x packages
baseurl=http://packages.elasticsearch.org/elasticsearch/1.1/centos
gpgcheck=1
gpgkey=http://packages.elasticsearch.org/GPG-KEY-elasticsearch
enabled=1</p>

<p>3. Install Elasticsearch(Elastic):
$ sudo yum install -y elasticsearch-1.1.1</p>

<p>4. Edit the Elastic Configs:
$ sudo vi /etc/elasticsearch/elasticsearch.yml</p>

<p>4a. Disable dynamic scripts by adding this line:
script.disable_dynamic: true</p>

<p>4b. Restrict outside access:
Change "# network.host: 192.168.0.1" to "network.host: localhost"</p>

<p>4c. Disable multicast by uncommenting the following line:
discovery.zen.ping.multicast.enabled: false</p>

<p>5. Start Elastic and enable on boot:
$ sudo systemctl start elasticsearch.service
$ sudo systemctl enable elasticsearch.service</p>

<h3>Install Kibana 3.01</h3>
<p>1. Download Kibana:
$ curl -O https://download.elasticsearch.org/kibana/kibana/kibana-3.0.1.tar.gz</p>

<p>2. Extract the archive:
$ tar xvf kibana-3.0.1.tar.gz</p>

<p>3. Edit the Kibana config file and change the elasticsearch server URL port from 9200 to 80:
$ vi ~/kibana-3.0.1/config.js
Change 'elasticsearch: "http://"+window.location.hostname+":9200",' to
elasticsearch: "http://"+window.location.hostname+":80",
</p>

<p>4. Create a directory and copy the Kibana installation into it:
$ sudo mkdir -p /var/www/kibana3
$ sudo cp -R ~/kibana-3.0.1/* /var/www/kibana3/</p>

<h3>Install Apache to serve our Kibana Installation</h3>
<p>1. Install Apache HTTP:
$ sudo yum install -y httpd</p>

<p>2. Configure Apache to proxy the port 80 requests to port 9200.  We do this by configuring an Apache VirtualHost.</p>
$ wget https://assets.digitalocean.com/articles/logstash/kibana3.conf # this is a sample VirtualHost configuration from Digital Ocean

<p>2a. Change VirtualHost and ServerName values to your domain name:
$ vi kibana3.conf
Changed <VirtualHost FQDN:80> to <VirtualHost localhost:80>
Changed "ServerName FQDN" to "ServerName localhost"
# make sure you change root to the directory where we installed Kibana</p>

<p>3. Copy this config file to your Apache configuration:
$ sudo cp ~/kibana3.conf /etc/httpd/conf.d/</p>

<p>4. Create a login that will be used to access Kibana:
$ sudo htpasswd -c /etc/httpd/conf.d/kibana-htpasswd vagrant
# I used the password "vagrant"</p>
# This added "vagrant:<password>" to the file /etc/httpd/conf.d/kibana-htpasswd, where password is hashed.

<p>5. Start Apache and enable on boot</p>
<p>$ sudo systemctl start httpd.service
$ sudo systemctl enable httpd.service</p>

<p>Kibana should now be accessible via "localhost" or the private_ip address we gave Vagrant.</p>

<h4>Errors</h4>
1.  No Kibana welcome screen.  Only the Apache welcome screen.


<h4>Resources</h4>
<ul>
<li>https://www.digitalocean.com/community/tutorials/how-to-use-logstash-and-kibana-to-centralize-logs-on-centos-7</li>
</ul>