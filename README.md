# swan-dns
 
```
   (o_
\\\_\
<____)    
 ```
Swan DNS is a python based DNS server taking the concept of plug-ins to the next level.
The entire idea is to create a DNS server where writing a costume DNS resolver is not more than writing a simple small plug-in.

More than this plug-ins can be reused and concatenate together to provide a simple way for a very complex to achive DNS resolution
with commonly used dns servers/tools.

Swan-dns stands for Swiss Army Knife DNS Server (yes I know I am missing the K) and got nothing to do with the magnificent bird called swan. 


## Requests Flow

The swan-dns handle requests with a multiplexing approach.
Once a request recived it tries to identify if a zone defined for this server can handle the request.
If no zone was found than an empty request is returned.
The following diagram describe that flow.
The numbers are indicating the resolution order.

![Processing Flow](/images/Diag1.png)

## Resolving modules 
Resolving modules are the actual resolution code parts of the DNS server.
Each resolution module is designed to be an zone indpendent code that will do it's work to whatever zone it will need to resolve requests for.

At server startup according to the configuration a resolving module with it's configurations is attached to a chain of processing modules for a specific zone.

The structure of zone relation to the resolution module is described at the following diagram.
![Zone resolvers structure](/images/Diag2.png)

!!! please note that a zone without any resolution module is a possiable scenario but pretty useless since nothing will be resolved.

## Rolling response 
As mentioned before the resolving modules for each zone are chained together but the code of each does some work related to the resoltion.
The only connection between the modules chaned together is that the response (and request) data is rolled from one module to the other.
This means several things.

* Each resoltion module recives the response object which might have been changed by all the modules in front of him in the chain.
* Each change made by the resolution module is reflected to all the modules following him in the chain.
* The First module always get an empty response object.
* After the last module in the chain finished it's work the response will be sent back to the client.

The following diagram describes the process.
![Rolling Response](/images/Diag3.png)

Modules can also stop the chain of processing in 2 ways: 
* Stop the chain and return the dns reponse object which was built so far.
* Stop the chain and drop the response returning nothing to the client.

For example if we would like to have some whitelist module which filters ip addreses , we should put it as our first module which will stop the chain if a certain ip is not allowed to query the dns server.

# Installation.

## Prerequsites 

Swan-dns requieres the following python packages to work 

* dnslib 
* ipcalc

Install then anyway you like as long as they are avaliable to load.
A good option is to use pip form installation.

`pip install dnslib ipcalc`

The swan-dns was written as a complete python package so in theory you can just check out the code add `<path to swan-dns dir>` to your PYTHONPATH.
Next you will be able to import the swandns package or an other sub package of it.

To make things a little bit more easy a setup.py script was written.

## Using the setup.py 

The setup.py script was written with python [distutils.core.setup](https://docs.python.org/2/distutils/setupscript.html).
So all the setup.py options are avaliable.

1. ```git clone https://github.com/tpotlog/swan-dns.git``` 
2. ```cd swan-dns``` 
3. ```python setup.py install [intallation options]``` 

# Running The server

Currently the server supports loading it's configurations from an ini file.

## Ini configuration file format.
The ini format is as follows.

```ini
[server]
###This section holds all the server configurations### 
address=0.0.0.0 # the ip address to bind to "0.0.0.0" means bind to all.
port=53 # The dns port to listen to (please make sure you got the right user permissions to user port 53).
logfile=/var/log/swan-dns.log #path to the log file
loglevel=info # logging level avaliable are info,warn,debug (case in-sesitive).
modules_paths=<list of os paths to search for mudules "," is used as delimiter> # those directories are added to the PYTHONPATH
##Define dns zones this server will resolve at the following format.
#zone.<dns zone fqdn>=<list of zones section to load "," is used as delimiter>.
zone.example.com=zonefile.example.com

[zonefile.example.com]
###This section represent a module###
type=module
#the format is as follows 
#module option:
#opt1=val1
#opt2=val2
##options are documented at module code.
module_name=zonefile
zone_file=<path to the zone file>
```

**Example**

This example demostrates a server with the following charesteristics.

1. Listen on port 5053
2. Binds to all interfaces.
3. serving zone example.com using the zone file module.

***zone file (/var/tmp/example.com.zonefile)***
```zonefile
$ORIGIN example.com.
$TTL 3h
;Define records.
example.com. IN SOA ns.example.com username.example.com ( 2007120710 1d 2h 4w 1h )
example.com. IN NS ns
@	     IN NS ns2.example2.com.
@	     IN MX 10  mail01.example.com.
example.com. IN MX 20 mail02
; A Records
example.com. IN A 192.0.2.11 ;IPv4 for example.com.
example.com. IN A 192.0.2.12 ;IPv4 another for example.com.
	     IN AAAA 2001:db8:10::1 ; IPv6 for example.com.
ns 14000     IN A 192.0.2.12 ; IPv4 Address for the ns record.
	     IN AAAA 2001:db8:10::2 IPv4 Address for ns resord.
www	     IN CNAME example.com. ; cname for example.com record.
www2 3600 IN CNAME www ; cname for www cname which leads to example.com.
```

***ini file (/var/tmp/swan-dns.ini)***
```ini
[server]
port=5053
address=0.0.0.0
logfile=/var/log/swan-dns.log
zone.example.com=zonefile.example.com

[zonefile.example.com]
type=module
module_name=zonefile
zone_file=/var/tmp/example.com.zonefile
```
