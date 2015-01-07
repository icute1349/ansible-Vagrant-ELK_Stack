<h1>ELK Stack on CentOS 7 x86 64</h1>
<h3>Description:</h3>
<p>First iteration of an ELK stack provisioned with Vagrant and Ansible.  Below are all my notes from this project.  These are all the steps I went through to build this stack manually from the command line before coverting to Ansible.</p>
<p>NOTE: This stack is not for production....yet</p>
<h4>General TODO:</h4>
<ul>
<li>Edit for firewall settings</li>
</ul>

<h2>Prereqs</h2>
<p>TODO: include link to CentOS_7 box I am using.</p>
<ol>
<li>Create a Vagrantfile in the root of your project directory:
$ vagrant init <box_name></li>
<li>Configure the Vagrantfile.</li>
<li>Create Ansible playbook.yml and ansible.cfg files in the root directory</li>
</ol>

<h3>Install Java 7</h3>
<p>1. Install OpenJDK 7:</p>
<p>$ sudo yum install -y java-1.7.0-openjdk
</p>

<h3>Install Elasticsearch 1.1.1</h3>
<p>1. Import the Elasticsearch public GPG key into rpm:</p>
<p>$ sudo rpm --import http://packages.elasticsearch.org/GPG-KEY-elasticsearch</p>
<p>2. Create a new yum repository file for Elasticsearch:</p>
<p>$ sudo vi /etc/yum.repos.d/elasticsearch.repo</p>

<p>Add the following:
[elasticsearch-1.1]
name=Elasticsearch repository for 1.1.x packages
baseurl=http://packages.elasticsearch.org/elasticsearch/1.1/centos
gpgcheck=1
gpgkey=http://packages.elasticsearch.org/GPG-KEY-elasticsearch
enabled=1</p>

<p>3. Install Elasticsearch(Elastic):</p>
<p>$ sudo yum install -y elasticsearch-1.1.1</p>

<p>4. Edit the Elastic Configs:</p>
<p>$ n</p>

<p>4a. Disable dynamic scripts by adding this line:</p>
<p>script.disable_dynamic: true</p>

<p>4b. Restrict outside access:</p>
<p>Change "# network.host: 192.168.0.1" to "network.host: localhost"</p>

<p>4c. Disable multicast by uncommenting the following line:</p>
<p>discovery.zen.ping.multicast.enabled: false</p>

<p>5. Start Elastic and enable on boot:</p>
<p>$ sudo systemctl start elasticsearch.service</p>
<p>$ sudo systemctl enable elasticsearch.service</p>

<h3>Install Kibana 3.01</h3>
<p>1. Download Kibana:</p>
<p>$ curl -O https://download.elasticsearch.org/kibana/kibana/kibana-3.0.1.tar.gz</p>

<p>2. Extract the archive:</p>
<p>$ tar xvf kibana-3.0.1.tar.gz</p>

<p>3. Edit the Kibana config file and change the elasticsearch server URL port from 9200 to 80:</p>
<p>$ vi ~/kibana-3.0.1/config.js</p>
<p>Change 'elasticsearch: "http://"+window.location.hostname+":9200",' to
elasticsearch: "http://"+window.location.hostname+":80",
</p>

<p>4. Create a directory and copy the Kibana installation into it:</p>
<p>$ sudo mkdir -p /var/www/kibana3</p>
<p>$ sudo cp -R ~/kibana-3.0.1/* /var/www/kibana3/</p>

<h3>Install Apache to serve our Kibana Installation</h3>

<p>1. Install Apache HTTP:</p>
<p>$ sudo yum install -y httpd</p>

<p>2. Configure Apache to proxy the port 80 requests to port 9200.  We do this by configuring an Apache VirtualHost.</p>
<p>$ wget https://assets.digitalocean.com/articles/logstash/kibana3.conf # this is a sample VirtualHost configuration from Digital Ocean</p>

<p>2a. Change VirtualHost and ServerName values to your domain name:</p>
<p>$ vi kibana3.conf</p>
<p>Changed <VirtualHost FQDN:80> to <VirtualHost localhost:80></p>
<p>Changed "ServerName FQDN" to "ServerName localhost"</p>
<p># make sure you change root to the directory where we installed Kibana</p>

<p>3. Copy this config file to your Apache configuration:</p>
<p>$ sudo cp ~/kibana3.conf /etc/httpd/conf.d/</p>

<p>4. Create a login that will be used to access Kibana:</p>
<p>$ sudo htpasswd -c /etc/httpd/conf.d/kibana-htpasswd vagrant</p>
<p># I used the password "vagrant"</p>
<p># This added "vagrant:<password>" to the file /etc/httpd/conf.d/kibana-htpasswd, where password is hashed.</p>

<p>5. Start Apache and enable on boot</p>
<p>$ sudo systemctl start httpd.service</p>
<p>$ sudo systemctl enable httpd.service</p>

<p>Kibana should now be accessible via "localhost" or the private_ip address we gave Vagrant.</p>

<h4>Issues</h4>
1) "Upgrade Required Your version of Elasticsearch is too old. Kibana requires Elasticsearch 0.90.9 or above." - occurs when I try to access Kibana server.
2) "Error Could not reach http://localhost:80/_nodes. If you are using a proxy, ensure it is configured correctly" - additional error message from Kibana.

These could be more permissions issues.

<h4>To DO:</h4>
<ul>
<li>Remove kibana tarball</li>
</ul>

<h3>Install Logstash</h3>
<p>1. Create a new Logstash Yum Repository:</p>
<p>$ sudo vi /etc/yum.repos.d/logstash.repo</p>

<p>2. Add the below:
[logstash-1.4]
name=logstash repository for 1.4.x packages
baseurl=http://packages.elasticsearch.org/logstash/1.4/centos
gpgcheck=1
gpgkey=http://packages.elasticsearch.org/GPG-KEY-elasticsearch
enabled=1
</p>

<p>3. Install Logstash</p>
<p>$ sudo yum install -y logstash-1.4.2</p>

<p>4. Generate the SSL Certificates to Use with Logstash Forwarder</p>
<p>$cd /etc/pki/tls; sudo openssl req -x509 -batch -nodes -days 3650 -newkey rsa:2048 -keyout private/logstash-forwarder.key -out certs/logstash-forwarder.crt</p>

<p>5. Create the Logstash Input config file with the below content:</p>
<p>sudo vi /etc/logstash/conf.d/01-lumberjack-input.conf</p>
<p>input {
  lumberjack {
    port => 5000
    type => "logs"
    ssl_certificate => "/etc/pki/tls/certs/logstash-forwarder.crt"
    ssl_key => "/etc/pki/tls/private/logstash-forwarder.key"
  }
}</p>

<p>6. Create a config file to filter syslog messages with the below</p>
<p>$ /etc/logstash/conf.d/10-syslog.conf</p>
<p>filter {
  if [type] == "syslog" {
    grok {
      match => { "message" => "%{SYSLOGTIMESTAMP:syslog_timestamp} %{SYSLOGHOST:syslog_hostname} %{DATA:syslog_program}(?:\[%{POSINT:syslog_pid}\])?: %{GREEDYDATA:syslog_message}" }
      add_field => [ "received_at", "%{@timestamp}" ]
      add_field => [ "received_from", "%{host}" ]
    }
    syslog_pri { }
    date {
      match => [ "syslog_timestamp", "MMM  d HH:mm:ss", "MMM dd HH:mm:ss" ]
    }
  }
}</p>

<p>7. Create a Configuration file for output</p>
<p>$ sudo vi /etc/logstash/conf.d/30-lumberjack-output.conf</p>
<p>output {
  elasticsearch { host => localhost }
  stdout { codec => rubydebug }
}
</p>

<h3>Install Logstash Forwarder</h3>
<p>1. Copy SSL Certificate and Logstash Forwarder Package</p>
<p>$ scp /etc/pki/tls/certs/logstash-forwarder.crt user@server_private_IP:/tmp
</p>

<p>2. Install Logstash Forwarder Package into the Home Directory</p>
<p>$ curl -O http://packages.elasticsearch.org/logstashforwarder/centos/logstash-forwarder-0.3.1-1.x86_64.rpm</p>

<p>3. Install the Forwarder init script</p>
<p>$ cd /etc/init.d/; sudo curl -o logstash-forwarder http://logstashbook.com/code/4/logstash_forwarder_redhat_init</p>
<p>And change the permissions</p>
<p>$ sudo chmod +x logstash-forwarder
</p>

<p>4. Install init script dependent file</p>
<p>$ sudo curl -o /etc/sysconfig/logstash-forwarder http://logstashbook.com/code/4/logstash_forwarder_redhat_sysconfig</p>

<p>5. Edit the script to include the below</p>
<p>$ sudo vi /etc/sysconfig/logstash-forwarder</p>
<p>$ sed -i 's/^.LOGSTASH_FORWARDER_OPTIONS=/(LOGSTASH_FORWARDER_OPTIONS="-config /etc/logstash-forwarder -spool-size 100/")/' /etc/sysconfig/logstash-forwarder # this script is not working....</p>
<p>LOGSTASH_FORWARDER_OPTIONS="-config /etc/logstash-forwarder -spool-size 100"</p>

<p>6. Copy the Downloaded SSL Certificate to certs directory</p>
<p>$ sudo cp /tmp/logstash-forwarder.crt /etc/pki/tls/certs/</p>

<p>7. Configure the Forwarder on the Server and Input the Private IP Address of your Logstash Server</p>
<p>$ sudo vi /etc/logstash-forwarder</p>
<p>{
  "network": {
    "servers": [ "logstash_server_private_IP:5000" ],
    "timeout": 15,
    "ssl ca": "/etc/pki/tls/certs/logstash-forwarder.crt"
  },
  "files": [
    {
      "paths": [
        "/var/log/messages",
        "/var/log/secure"
       ],
      "fields": { "type": "syslog" }
    }
   ]
}</p>

<p>8. Add the Forwarder service with chkconfig</p>
<p>$ sudo chkconfig --add logstash-forwarder</p>

<p>9. Start the Logstash Forwarder</p>
<p>$ sudo service logstash-forwarder start</p>

<p>You should now be able to connect to Kibana</p>

<h4>Issues</h4>
<p>1. Cannot start logstash-forwarder.  When I run "$ sudo systemctl start logstash-forwarder", get this error message: "msg: /etc/init.d/logstash-forwarder: line 25: /lib/init/vars.sh: No such file or directory"</p>
<p>2. Additional ansible provisioning fails at "Install the Forwarder package" with this error message "failed: [default] => {"changed": true, "cmd": ["rpm", "-ivh", "/home/vagrant/logstash-forwarder-0.3.1-1.x86_64.rpm"], "delta": "0:00:00.014631", "end": "2015-01-06 13:10:31.592836", "rc": 1, "start": "2015-01-06 13:10:31.578205", "warnings": ["Consider using yum module rather than running rpm"]}
stderr: 	package logstash-forwarder-0.3.1-1.x86_64 is already installed
stdout: Preparing...                          ########################################"</p>

<h4>Resources</h4>
<ul>
<li>https://www.digitalocean.com/community/tutorials/how-to-use-logstash-and-kibana-to-centralize-logs-on-centos-7</li>
</ul>