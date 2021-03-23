# Quick setup Vagrant with ELK
Quick and dirty Vagrant script to create an Ubuntu 20.04 with elasticstack and Kibana.

## Prerequisites

- Virtualbox https://www.virtualbox.org/
- vagrant https://www.vagrantup.com/downloads

```
vagrant plugin install vagrant-reload
```

## Lab

Create a file called *Vagrantfile* in a directory, and paste the following content into it:
```
$elk = <<SCRIPT
echo "Getting ELK"
apt-get update
apt-get install docker-compose -y
git clone https://github.com/deviantony/docker-elk.git
cd docker-elk
docker-compose up -d
docker update --restart=always $(sudo docker ps -q) #Make sure to run after reboot
SCRIPT

Vagrant.configure("2") do |config|
  config.vm.define "elk" do |cfg|
      config.vm.box = "ubuntu/focal64"
      cfg.vm.hostname = "elk"
      config.vm.provision "shell", inline: $elk
      config.vm.network "forwarded_port", guest: 5601, host: 5601
      config.vm.network "forwarded_port", guest: 9200, host: 9200
    end
    config.vm.provider "virtualbox" do |vb, override|
      vb.name = "elk"
      vb.customize ["modifyvm", :id, "--memory", 8192]
      vb.customize ["modifyvm", :id, "--cpus", 4]
      vb.customize ["modifyvm", :id, "--clipboard", "bidirectional"]
    end
end
```

The elasticstack and Kibana instance takes a while to become ready after the VM is build.

## Injecting data
Download Winlogbeat (the zip file): https://www.elastic.co/downloads/beats/winlogbeat and extract the content

Create the file winlogbeat-evtx.yml and paste the following:
```
winlogbeat.event_logs:
  - name: ${EVTX_FILE} 
    no_more_events: stop 
winlogbeat.shutdown_timeout: 60s 
winlogbeat.registry_file: evtx-registry.yml 
processors:
  - drop_fields:
        fields: ["event.kind", "event.code", "agent.ephemeral_id", "ecs.version"]

output.elasticsearch:
  hosts: ["localhost:9200"]
  username: elastic
  password: changeme
```
Parts stolen from: https://burnhamforensics.com/2019/11/19/manually-upload-evtx-log-files-to-elk-with-winlogbeat-and-powershell/

Inject the evtx logs with following command:
```
.\winlogbeat.exe -e -c .\winlogbeat-evtx.yml -E EVTX_FILE=c:\backup\Security-2019.01.evtx
```

Remember to setup the kibana index patterns for *winlogbeat* for indexing.

## Interaction

Kibana can be opened through http://127.0.0.1:5601/

- Username: *elastic*
- Password: *changeme*

The ubuntu machine can be accessed using the build in ssh functionality in vagrant:
```
vagrant ssh
# See CPU/memory stats
sudo docker stats --all
```

# Interesting logs
| (2000/XP/2003) | (Vista/7/8/2008/2012) | Description |
| -------- | -------- | -------- |
| 528     | 4624     | Successful Logon     |
| 529     | 4625     | Failed Login     |
| 538     | 4634     | An account was logged off     |
| 552     | 4648     | A logon was attempted using explicit credentials     |
| 680     | 4776     | Successful /Failed Account Authentication	     |
| 624     | 4720     | 	A user account was created     |
| 626   | 4722     | A user account was enabled     |
| 628     | 4724     | An attempt was made to reset an accounts password     |
| 632     | 4728	     | A member was added to a security-enabled |
| 636     | 4732     | 	A member was added to a security-enabled local group     |
| 2934     | 7030     | Service Creation Errors     |
| 2944     | 7040     | 	The start type of the IPSEC Services service was changed from disabled to auto start.     |
| 2949     | 7045     | Service Creation     |




## Logon Type Codes
One of the useful information that Successful/Failed Logon event provide is how the user/process tried to logon (Logon Type) but Windows display this information as a number and here is a list of the logon type and their explanation:

| Logon type | Logon | Description |
| -------- | -------- | -------- |
| 2     | Interactive     | A user logged on to this computer.     |
| 3     | Network     | A user or computer logged on to this computer from the network.     |
| 4     | Batch     | Batch logon type is used by batch servers, where processes may be executing on behalf of a user without their direct intervention.     |
| 5     | Service     | A service was started by the Service Control Manager.     |
| 7     | Unlock     | This workstation was unlocked.     |
| 8     | NetworkCleartext     | A user logged on to this computer from the network. The user’s password was passed to the authentication package in its unhashed form. The built-in authentication packages all hash credentials before sending them across the network. The credentials do not traverse the network in plaintext (also called cleartext).     |

Succesful log on from specific machine:
```
winlog.event_id: 4624 AND winlog.event_data.IpAddress : "{source_IP}"
```

