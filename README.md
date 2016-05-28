# quickstream
Instructions for capturing a data stream

1. launch server  
2. install nifi  
3. launch nifi  
4. listen for data
5. save data to s3 as csv  
6. save data to postre db  

## Launch server

### Amazon AWS - EC2

Amazon Linux AMI 2016.03.1 (HVM), SSD Volume Type - ami-d0f506b0  
t2.micro  
8 GiB storage  
No tag  
Security:  
SSH (TCP 22 0.0.0.0/0)  
Custom UDP Rule (UDP 29100 0.0.0.0/0)  
Custom TCP Rule (TCP 29101 0.0.0.0/0)  
Custom TCP Rule (TCP 8080 0.0.0.0/0)  
Save key  

The UDP (29100) and TCP (29101) ports we opened will be used for sending/listening for packets.

They were chosen as not to clash with an major ports listed here:  
https://en.wikipedia.org/wiki/List_of_TCP_and_UDP_port_numbers  
But otherwise the choices were arbitrary.

The UDP (8080) port allows us to access the nifi web application from a local web browser.

#### If windows and planning to connect through putty

Save key file e.g. aws_01.pem to c:\users\username\.ssh\  
Open PuTTY Key Generator  
Load aws_01.pem  
Save private key  
As aws_01.ppk  
No passphrase required  

Open PuTTY  
Session
Host Name: ec2-user@ec2-54-191-211-XXX.us-west-2.compute.amazonaws.com  
Port: 22  

Connection > SSH > Auth  
Private key file for authentication - attach aws_01.ppk  

Connection > SSH > Tunnels
Source Port: L8080   
Destination: 127.0.0.1:8080  
(This allows us to connect to nifi, once running, through a local web browser)

Save profile (for convenience) and connect

## Install NIFI

Install security updates  
```bash
sudo yum update
```

On your local machine visit apache nifi website:  
https://nifi.apache.org/download.html

Pick the most recent version and make note of the URL to download the tar.gz file of the binaries  
e.g.  
http://mirror.ventraip.net.au/apache/nifi/0.6.1/nifi-0.6.1-bin.tar.gz

On EC2 use wget to download this file  
```bash
wget http://mirror.ventraip.net.au/apache/nifi/0.6.1/nifi-0.6.1-bin.tar.gz
```

Unzip, rename directory to 'nifi', remove download zip file
```bash
tar -zxvf nifi-0.6.1-bin.tar.gz
mv nifi-0.6.1 nifi
rm nifi-0.6.1-bin.tar.gz
```

## Launch NIFI

Run
```bash
~/nifi/bin/nifi.sh start
```

Check status
```bash
~/nifi/bin/nifi.sh status
```

Stop
```bash
~/nifi/bin/nifi.sh stop
```

On local machine open web browser and navigate to:  
127.0.0.1:8080/nifi
http://localhost:8080/nifi

## Listen for data

Create ListenUDP process

Configure > Properties  
Port: 29100

Configure > Settings  
Autoterminate on success: True

Start process

### Packet Sender

Download and install on local machine:  
https://packetsender.com/

Create new packet  
Name: nifi_udp_01  
ASCII: [nifi_udp_01:data1,data2,data3]  
Address: ec2-54-191-211-XXX.us-west-2.compute.amazonaws.com  
Port: 29100  
UDP

Save.  
Send.

Right click on ListenUDP process > Data Provenance  

You should see a RECEIVE event then immediately a DROP event.

Select RECEIVE event > show linearge  
(tree diagram in far right column)

Right click RECEIVE > View details > Content > Output Claim > View

Here we can see the packet we sent!  
[nifi_udp_01:data1,data2,data3]

### Getting Started

There are a number of good pre-built processes, so you can start to get a feel for how to get things done:  
https://cwiki.apache.org/confluence/display/NIFI/Example+Dataflow+Templates

Here are details on how to load a template:  
http://nifi.apache.org/docs/nifi-docs/html/user-guide.html#Manage_Templates


## Save data to S3 as CSV

## Save data to Postres DB

