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

#### If windows and planning to connect through tunnel

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

#### If mac and planning to connect through tunnel


http://apple.stackexchange.com/questions/16976/whats-a-good-ssh-tunneling-client-for-os-x

```bash
ssh -D 8080 -C -N ec2-user@ec2-54-214-192-xxx.us-west-2.compute.amazonaws.com -i "eyc3_alex1.pem"
```

Now, let’s start browsing the web using with your new SSH Tunnel (Chrome):  

Open Google Chrome  
Select ‘Chrome’ up the top left  
Select ‘Preferences’  
Select ‘Show advanced settings…’  
Select ‘Change proxy settings…’  
Select ‘SOCKS Proxy’  
Enter ’127.0.0.1′  
Enter port ’8080′  
Save changes by selecting ‘OK’  

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

Stop (FYI only don't execute at the moment)
```bash
~/nifi/bin/nifi.sh stop
```

On local machine open web browser and navigate to:  
127.0.0.1:8080/nifi

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


## Save data to Postres DB

### GetHTTP

PURPOSE

Imports file/data via a HTTP request into the flowfile's content. In this example we're getting an XML file.

PROPERTIES (LISTED)

url: http://www.abc.net.au/news/feed/51892/rss.xml  

filename: raw_rss_abc  

else: default

### EvaluateXPath

PURPOSE

Parse XML document for specific tags/attributes.

PROPERTIES (LISTED)

Destination: flowfile-attribute  
(parse into attributes - as opposed to parsing into the content of the flowfile)

PROPERTIES (MANUAL)

description: /rss/channel/item/description

pubDate: /rss/channel/item/pubDate

(etc...)

NOTES

Xpath resource  
http://www.w3schools.com/xsl/xpath_intro.asp

Needs fixing: Current setup only grabs attributes of first item, need to work out how to iterate through all items...

### AttributeToJSON

PURPOSE

Get data into JSON format - easy to then convert to SQL for inserting into a database.

PROPERTIES (LISTED)

Attributes List: pubDate, description

Destination: flowfile-content

(We are now ready to move the parsed data from attributes to the content of the flowfile)

else default

### ConvertJSONToSQL

PURPOSE

Generate a SQL statement (based on our JSON data) to be passed to a PutSQL call. 

PROPERTIES (LISTED)

Statement Type: INSERT
(Needs fixing: This should be an UPDATE call instead - need to get keys right during create table step...)

Table Name: abc

Schema Name: public

JDBC Connection Pool:

Click arrow > NiFi Flow Settings > Controller Services

New Controller service > DBCPConnectionPool

Database Connection URL: jdbc:postgresql://instanceid01.cxyhnytsfsmi.us-west-2.rds.amazonaws.com:5432/dbname01

Database Driver Class Name: org.postgresql.Driver

Database Driver Jar Url: file:///home/ec2-user/pgres/postgresql-9.4.1211.jre6.jar

(Need to download this file: 
'''bash
cd /home/ec2-user/pgres  
wget https://jdbc.postgresql.org/download/postgresql-9.4.1211.jre6.jar
'''
)

Database User: masteru

Password: masterpass

(Refer to https://github.com/metcalfalex/quickpostgresdb)

### PutSQL

PURPOSE

Push generated SQL to server

PROPERTIES (LISTED)

JDBC Connection Pool: (DBCPConnectionPool set up above)

else default

NOTES

Need to manually create table for data to be inserted into:  
'''sql
CREATE TABLE dbname01.public.abc (

pubDate VARCHAR(50)
,title VARCHAR(200)

);
'''








