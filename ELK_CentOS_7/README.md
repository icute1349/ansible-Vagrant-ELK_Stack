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


<h4>Resources</h4>
<ul>
<li>https://www.digitalocean.com/community/tutorials/how-to-use-logstash-and-kibana-to-centralize-logs-on-centos-7</li>
</ul>