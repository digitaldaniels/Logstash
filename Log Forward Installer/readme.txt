Build and Package Logstash Forwarder

Note: We will build and Package the Logstash Forwarder on our Logstash Server.

We need to install a few tools that are used to build and package Logstash Forwarder. Let's start by installing Go.

Add the go PPA to apt and update:
sudo apt-add-repository ppa:duh/golang
sudo apt-get update


Then run this command to install Go:
sudo apt-get install golang


Next, install developer tools, git, ruby, and ruby-dev:
sudo apt-get install build-essential git ruby ruby-dev


Clone the logstash-forwarder repository from github:
cd ~; git clone git://github.com/elasticsearch/logstash-forwarder.git
cd logstash-forwarder


Install fpm gem which will be used to create the package:
sudo gem install fpm


Make the deb (or rpm, by replacing deb with rpm) package:
umask 022
make deb


This creates a file called logstash-forwarder_0.3.1_amd64.deb in ~/logstash-forwarder. This package needs to be installed on all of the servers that you want to send logs to your Logstash Server.


Set Up Logstash Forwarder

Note: Do these steps for each server that you want to send logs to your Logstash Server.

Copy SSL Certificate and Logstash Forwarder Package

On Logstash Server, copy the SSL certificate to Server (substitute with your own login):
scp /etc/pki/tls/certs/logstash-forwarder.crt user@server_private_IP:/tmp


Also, copy over the Logstash Forwarder package that you created:
scp ~/logstash-forwarder/logstash-forwarder_0.3.1_amd64.deb user@server_private_IP:/tmp


Install Logstash Forwarder Package

On Server, install the Logstash Forwarder Package:
sudo dpkg -i /tmp/logstash-forwarder_0.3.1_amd64.deb


Next, you will want to install the Logstash Forwarder init script, so it starts on bootup:
cd /etc/init.d/; sudo wget https://raw.github.com/elasticsearch/logstash-forwarder/master/logstash-forwarder.init -O logstash-forwarder
sudo chmod +x logstash-forwarder
sudo update-rc.d logstash-forwarder defaults


Now copy the SSL certificate into the appropriate location (/etc/pki/tls/certs):
sudo mkdir -p /etc/pki/tls/certs
sudo cp /tmp/logstash-forwarder.crt /etc/pki/tls/certs/


Configure Logstash Forwarder

On Server, open the Logstash Forwarder configuration file, which is formatted as JSON, for editing:
sudo vi /etc/logstash-forwarder


Now add the following lines into the file, substituting in your Logstash Server's private IP address for logstash_server_private_IP:
{
  "network": {
    "servers": [ "logstash_server_private_IP:5000" ],
    "timeout": 15,
    "ssl ca": "/etc/pki/tls/certs/logstash-forwarder.crt"
  },
  "files": [
    {
      "paths": [
        "/var/log/syslog",
        "/var/log/auth.log"
       ],
      "fields": { "type": "syslog" }
    }
   ]
}


Save and quit. This configures Logstash Forwarder to connect to your Logstash Server on port 5000 (the port that we specified an input for earlier), and uses the SSL certificate that we created earlier. The paths section specifies which log files to send (here we specify syslog and auth.log), and the type section specifies that these logs are of type "syslog* (which is the type that our filter is looking for).

Note that this is where you would add more files/types to configure Logstash Forwarder to other log files to Logstash on port 5000.

Now restart Logstash Forwarder to put our changes into place:
sudo service logstash-forwarder restart


Now Logstash Forwarder is sending syslog and auth.log to your Logstash Server! Repeat this process for all of the other servers that you wish to gather logs for.
