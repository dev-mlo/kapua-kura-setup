This document describes how to build, install and setup
_Kapua_ and _Kura_. _Kapua_ and _Kura_ needs Unix system to build
and run.

Also have a look at the ESF Dokumentation:
* Kura: https://esf.eurotech.com/docs
* Kapua: https://ec.eurotech.com/docs


# Run _Kapua_ on Raspberry PI (64bit, 8GB RAM)

Because there are no existing images for arm64 systems, we have to build
the _Kapua_ services ourselves from the sources.

## Preparation

### Install curl
```
sudo apt update
sudo apt install -y curl
```


### Install git
```
sudo apt update
sudo apt install -y git
```

### Install jdk 8 & 11
In this case we use the Zulu JDKs, but you can install any other JDK.

```
sudo apt install gnupg ca-certificates curl
curl -s https://repos.azul.com/azul-repo.key | sudo gpg --dearmor -o /usr/share/keyrings/azul.gpg
echo "deb [signed-by=/usr/share/keyrings/azul.gpg] https://repos.azul.com/zulu/deb stable main" | sudo tee /etc/apt/sources.list.d/zulu.list

sudo apt update
sudo apt install -y zulu8-jdk zulu11-jdk
```


### Install maven
```
sudo apt update
sudo apt install -y maven
```


### Install Docker and -compose
```
curl -fsSL https://get.Docker.com -o get-Docker.sh
sudo sh get-Docker.sh
sudo usermod -aG docker $USER
newgrp docker

sudo apt install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin
sudo apt install -y docker-compose
```


## Build and deploy Kapua

### Clone github repo
```git clone https://github.com/eclipse/kapua.git```


### Build Kapua
```mvn install -f $HOME/kapua -P docker -DskipTests```

This step takes forever because of Raspberry PIs hardware limitations.<br>
Seriously: Do something else now: take a break, drink three coffees, etc.


### Deploy
```$HOME/kapua/deployment/docker/unix/docker-deploy.sh```

Now every service should be running on the Raspberry PI and can be used.


### Swagger-UI available under
```http://<rasp-ip>:8081/doc/```


---------------------------------------------------

## Kapua-Console
The _Kapua_ console is __NOT__ running on the Raspberry PI.  
We are running the console on another x64 system because there
is an existing image on Docker Hub for this Architecture we
can use. So we don't have to build the console from the sources.

docker-compose.yml
```
version: '3.1'

services:
  kapua-console-raspberry:
    container_name: kapua-console-raspberry
    image: kapua/kapua-console:latest
    ports:
      - "8080:8080"
      - "8443:8443"
    environment:
      - SQL_DB_ADDR=<rasp-ip> z. B. 192.168.178.37
      - BROKER_ADDR=<rasp-ip>
      - SERVICE_BROKER_ADDR=amqp:/<rasp-ip>:5672
      - DATASTORE_ADDR=<rasp-ip>
      - JOB_ENGINE_BASE_ADDR=<rasp-ip>
```

After the service started successfully the _Kapua_ UI is available under ```http://localhost:8080```<br>
User: ```kapua-sys```<br>
PW: ```kapu-password```


## Kura
### Install Kura (recommended)
There are currently no packages available to install, so you have to build
the binaries by yourself. This may require a Maven and or Java installation.

```bash
cd $HOME
git clone https://github.com/eclipse/kura.git
cd kura
sudo chmod +x ./build-all.sh
sudo ./buil-all.sh
```

This will take a while. After the build process has finished you can
find all installers in ```/kura/kura/distrib/target```. The directory
contains a lot of intermediate files. Use the following commands to
copy all relevant files to your ```home``` directory:

```bash
cd $HOME
mkdir kura-installer
cd $HOME kura/kura/distrib/target/*installer.sh
cd $HOME kura/kura/distrib/target/*installer.deb
```

Find the correct installer for your system and execute it.

### Run container
__Important__: A Kura instance in a container may not have the complete
functionality. For example, it is not possible to use the "Container
Orchestration Service". To use this feature you have to install Kura
on your machine.

There is an existing _Kura_ image on Docker Hub so we don't have to build the image
on our own.

```bash
docker run -d --name kura -p 443:443 eclipse/kura:latest 
```

The volume bind for ```docker.sock``` makes docker for the container available.

|          |                    |
|----------|--------------------|
| Kura UI  | https://localhost  |
| Username | admin              |
| Password | amin               |


### Cloud Connections
Connect Kura to _Kapua_ Server.

#### CloudService
Define the following parameters

| Parameter name     | Note                                                                       | Example      |
|--------------------|----------------------------------------------------------------------------|:-------------|
| Devise Custom-Name | An ID for this __Kura__ instance. In _Kapua_ it will be listed as a device | ```rpi-1```  | 


![Cloudservice](/doc/kura_1_cloudservice.png)

#### MqttDataTransport
Define the following parameters

| Paramter name             | Note                                                                                                                                               | Example                          |
|---------------------------|----------------------------------------------------------------------------------------------------------------------------------------------------|:---------------------------------|
| Broker URL                | URL to the running _Kapua_ Instance ```mqtt://<ip-to-kapua>:1883```                                                                                | ```mqtt://192.168.178.37:1883``` |
| Topc Context Account-Name | Name of the Child Account of _Kapua_. Must contain a user with defined credentials                                                                 | ```ACME123```                    |
| Username                  | A name of a user of the defined child account                                                                                                      | ```user123```                    |
| Password                  | The corresponding password of the child account                                                                                                    | ```kura@Kapua123```              |
| Client-id                 | An ID for this __Kura__ instance. In _Kapua_ it will be listed as a device. Should be the same as the "Devise Custom-Name" from "CloudService" tab | ```rpi-1```                      |

![MqttDataTransport](/doc/kura_2_mqtt_data_transport.png)


#### Publisher (for later use)
* "New Pub/Sub"
* Select "org.eclipse.kura.cloud.publisher.CloudPublisher" and give it a name
* Set the "Application Id" which will be shown in the Data section in K

The publisher can be used to transfer data from a field device (any sensors
like temperature / humidity) to the _Kapua_ server. The defined publisher
can be selected in the "Wire Graph" section when adding a "WireAsset".


### Packages
|                                             |                                                                                |     |
|---------------------------------------------|--------------------------------------------------------------------------------|:----|
| ```org.eclipse.kura.protocol.modbus```      |                                                                                |     |
| ```com.eurotech.modbus.driver```            | Read data from Modbus devices and/or from the _ModusPal_                       |     |
| ```org.eclipse.kura.example.publisher```    | If you just want to test your _Kapua_ connection and want to publish some data |     |



### Modbus Simulation for Kura
https://esf.eurotech.com/docs/modbus-application-in-kura-wires

This documentation is nearly complete and describes, how to simulate a
device using ModbusPal. One point is missing in this doc: How to publish
the date to _Kapua_.


#### Setup _Kapua_ Publisher
* Switch back to "Wire Graph"
* Select the added Publisher
* Select the added "CloudPublisher Target Filter"

Open _Kapua_, switch to "Data" and now you should see the Date published
by the ModbusPal