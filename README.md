<h1>ELK Stack on CentOS 7 x86 64</h1>
<h3>Description:</h3>
<p>This is an ELK stack that uses an nginx web server and forwards logs with the logstash-forwarder.  Below are all my notes from this project.  These are all the steps I went through to build this stack manually from the command line before writing the Ansible roles.</p>
<p>NOTE: This is not for production</p>
<h4>General To Do:</h4>
<ul>
<li>Refactor:</li>
<li>Remove tarballs after install</li>
</ul>

<h2>Prereqs:</h2>
<p>You can find the CentOS 7 box I used via Atlas: mjp182/CentOS_7</p>
<ol>
<li>Create a Vagrantfile in the root of your project directory:
<pre>$ vagrant init mjp182/CentOS_7</pre></li>
<li>Configure the Vagrantfile.</li>
<li>Create Ansible playbook.yml and ansible.cfg files in the root directory</li>
</ol>

<h3>Install Java 7</h3>
<p>1. Install OpenJDK 7:</p>
<pre>$ sudo yum install -y java-1.7.0-openjdk</pre>

<h3>Install Elasticsearch 1.1.1</h3>
<p>1. Import the Elasticsearch public GPG key into rpm:</p>
<pre>$ sudo rpm --import http://packages.elasticsearch.org/GPG-KEY-elasticsearch</pre>
<p>2. Create a new yum repository file for Elasticsearch:</p>
<pre>$ sudo vi /etc/yum.repos.d/elasticsearch.repo</pre>
<p>Add the following:
[elasticsearch-1.1]
name=Elasticsearch repository for 1.1.x packages
baseurl=http://packages.elasticsearch.org/elasticsearch/1.1/centos
gpgcheck=1
gpgkey=http://packages.elasticsearch.org/GPG-KEY-elasticsearch
enabled=1</p>

<p>3. Install Elasticsearch(Elastic):</p>
<pre>$ sudo yum install -y elasticsearch-1.1.1</pre>

<p>4. Edit the Elastic Configs:</p>

<p>4a. Disable dynamic scripts by adding this line:</p>
<p>script.disable_dynamic: true</p>

<p>4b. Restrict outside access:</p>
<p>Change "# network.host: 192.168.0.1" to "network.host: localhost"</p>

<p>4c. Disable multicast by uncommenting the following line:</p>
<p>discovery.zen.ping.multicast.enabled: false</p>

<p>5. Start Elastic and enable on boot:</p>
<pre>$ sudo systemctl start elasticsearch.service</pre>
<pre>$ sudo systemctl enable elasticsearch.service</pre>

<h3>Install Kibana 3.01</h3>
<p>1. Download Kibana:</p>
<pre>$ curl -O https://download.elasticsearch.org/kibana/kibana/kibana-3.0.1.tar.gz</pre>

<p>2. Extract the archive:</p>
<pre>$ tar xvf kibana-3.0.1.tar.gz</pre>

<p>3. Edit the Kibana config file and change the elasticsearch server URL port from 9200 to 80:</p>
<pre>$ vi ~/kibana-3.0.1/config.js</pre>
<p>Change 'elasticsearch: "http://"+window.location.hostname+":9200",' to
elasticsearch: "http://"+window.location.hostname+":80",
</p>

<p>4. Create a directory and copy the Kibana installation into it:</p>
<pre>$ sudo mkdir -p /var/www/kibana3</pre>
<pre>$ sudo cp -R ~/kibana-3.0.1/* /var/www/kibana3/</pre>

<h3>Install Apache to serve our Kibana Installation</h3>

<p>1. Install Apache HTTP: (I actually decided to Use nginx)</p>
<pre>$ sudo yum install -y httpd</pre>

<p>2. Configure Apache to proxy the port 80 requests to port 9200.  We do this by configuring an Apache VirtualHost.</p>
<pre>$ wget https://assets.digitalocean.com/articles/logstash/kibana3.conf # this is a sample VirtualHost configuration from Digital Ocean</pre>

<p>2a. Change VirtualHost and ServerName values to your domain name:</p>
<pre>$ vi kibana3.conf</pre>
<p>Changed <VirtualHost FQDN:80> to <VirtualHost localhost:80></p>
<p>Changed "ServerName FQDN" to "ServerName localhost"</p>
<p>Make sure you change root to the directory where we installed Kibana</p>

<p>3. Copy this config file to your Apache configuration:</p>
<pre>$ sudo cp ~/kibana3.conf /etc/httpd/conf.d/</pre>

<p>4. Create a login that will be used to access Kibana:</p>
<pre>$ sudo htpasswd -c /etc/httpd/conf.d/kibana-htpasswd vagrant</pre>
<p># I used the password "vagrant"</p>
<p># This added "vagrant:<password>" to the file /etc/httpd/conf.d/kibana-htpasswd, where password is hashed.</p>

<p>5. Start Apache and enable on boot</p>
<pre>$ sudo systemctl start httpd.service</pre>
<pre>$ sudo systemctl enable httpd.service</pre>

<p>Kibana should now be accessible via "localhost" or the private_ip address we gave Vagrant.</p>

<h3>Install Logstash</h3>
<p>1. Create a new Logstash Yum Repository:</p>
<pre>$ sudo vi /etc/yum.repos.d/logstash.repo</pre>

<p>2. Add the below:
[logstash-1.4]
name=logstash repository for 1.4.x packages
baseurl=http://packages.elasticsearch.org/logstash/1.4/centos
gpgcheck=1
gpgkey=http://packages.elasticsearch.org/GPG-KEY-elasticsearch
enabled=1
</p>

<p>3. Install Logstash</p>
<pre>$ sudo yum install -y logstash-1.4.2</pre>

<p>4. Generate the SSL Certificates to Use with Logstash Forwarder</p>
<pre>$ cd /etc/pki/tls; sudo openssl req -x509 -batch -nodes -days 3650 -newkey rsa:2048 -keyout private/logstash-forwarder.key -out certs/logstash-forwarder.crt</pre>

<p>5. Create the Logstash Input config file with the below content:</p>
<pre>$ sudo vi /etc/logstash/conf.d/01-lumberjack-input.conf</pre>
<p>input {
  lumberjack {
    port => 5000
    type => "logs"
    ssl_certificate => "/etc/pki/tls/certs/logstash-forwarder.crt"
    ssl_key => "/etc/pki/tls/private/logstash-forwarder.key"
  }
}</p>

<p>6. Create a config file to filter syslog messages with the below</p>
<pre>$ /etc/logstash/conf.d/10-syslog.conf</pre>
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
<pre>$ sudo vi /etc/logstash/conf.d/30-lumberjack-output.conf</pre>
<p>output {
  elasticsearch { host => "localhost" }
  stdout { codec => rubydebug }
}
</p>

<h3>Install Logstash Forwarder</h3>
<p>1. Copy SSL Certificate and Logstash Forwarder Package</p>
<pre>$ scp /etc/pki/tls/certs/logstash-forwarder.crt user@server_private_IP:/tmp</pre>

<p>2. Install Logstash Forwarder Package into the Home Directory</p>
<pre>$ curl -O http://packages.elasticsearch.org/logstashforwarder/centos/logstash-forwarder-0.3.1-1.x86_64.rpm</pre>

<p>3. Install the Forwarder init script</p>
<p>See included file</p>
<p>And change the permissions</p>
<pre>$ sudo chmod +x logstash-forwarder</pre>

<p>4. Install init script dependent file</p>
<pre>$ sudo curl -o /etc/sysconfig/logstash-forwarder http://logstashbook.com/code/4/logstash_forwarder_redhat_sysconfig</pre>

<p>5. Edit the script to include the below</p>
<pre>$ sudo vi /etc/sysconfig/logstash-forwarder</pre>
<pre>$ sed -i 's/^.LOGSTASH_FORWARDER_OPTIONS=/(LOGSTASH_FORWARDER_OPTIONS="-config /etc/logstash-forwarder -spool-size 100/")/' /etc/sysconfig/logstash-forwarder # this script is not working....</pre>
<p>LOGSTASH_FORWARDER_OPTIONS="-config /etc/logstash-forwarder -spool-size 100"</p>

<p>6. Copy the Downloaded SSL Certificate to certs directory</p>
<pre>$ sudo cp /tmp/logstash-forwarder.crt /etc/pki/tls/certs/</pre>

<p>7. Configure the Forwarder on the Server and Input the Private IP Address of your Logstash Server</p>
<pre>$ sudo vi /etc/logstash-forwarder</pre>
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
<pre>$ sudo chkconfig --add logstash-forwarder</pre>

<p>9. Start the Logstash Forwarder</p>
<pre>$ sudo service logstash-forwarder start</pre>

<p>You should now be able to connect to Kibana</p>

<h4>Resources</h4>
<ul>
<li>https://www.digitalocean.com/community/tutorials/how-to-use-logstash-and-kibana-to-centralize-logs-on-centos-7</li>
</ul>