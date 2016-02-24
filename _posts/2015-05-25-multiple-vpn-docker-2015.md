---
layout: post
title:  "Multiple VPN outgoing connections with Docker"
date:   2015-05-25 00:00:00 +0000
categories: coding docker node
---

#### The Problem

At early stage of [KNXMonitor.com](http://www.knxmonitor.com) this has been top 1 hard-to-solve-next-to-impossible on our task board. There is no connection security mechanism comes with KNX standard. Smart-building installers would setup VPN server on client's router to fence the house from hackers turning it into their private reality TV channel. As a cloud debugging product we would not want any user had their automated building wide open on the Internet. However for us, it requires a system designed to be able to maintain many *outgoing* VPN connections from single server to hundreds or thousands of building's IP addresses, within feasible cost.

- One server should be able to connect as VPN client to many VPN servers
- It needs to be fast to start initiating a connection
- Each connection needs to have next to native footprint
- It can be automated to maintain connected to client IP, and reconnect by itself when needed
- Each connection should live strictly within its own territory

Google has not been helpful on this topic, majority of on-line texts are talking about one server serving multiple *incoming* connections, that is, client computers *dial-in* to a VPN server to acquire a secured tunnel and an internal IP address. This is absolutely opposite of what we are trying to achieve, and the only answer we have been given from infrastructure experts was simply "expensive or impossible", with seemingly achievable solutions tagged with brands like Cisco or Jupiter's custom hardware.

To illustrate this problem , let's say we have two clients' buildings both behind a VPN server set up on its router.
{% highlight ruby %}
        [Subnet Conflict]

Client 1 (UK):
public address: client1.dyndns.org
internal KNXIP: 192.168.1.3 : 3671

Client 2 (Germany):
public address: 92.138.77.186
internal KNXIP: 192.168.1.3 : 3671

{% endhighlight %}

For our server to connect to both buildings and listens to devices' datagrams, as soon as we made connection with first building, via some ppp call and routes setup, we are given, say, 192.168.1.15 to Client 1's 192.168.1.x subnet, how would we connet to second Client 2 and route to a same subnet and communicate to a device that sits another side of the world but with same internal IP address?

We could setup VMs within host VM, yes, then how about connecting to 10 buildings? 1000? To spin up a wholly VM just to get hold of one tiny UDP stream is seriously endangering to our bank account as well as mother Earth.


#### God Bless Container

{% highlight text %}

	                                              +------------------+
	                                              | Docker Container |
	                                              +------------------+
	+------------+               +------------+   |                   
	| Building   |               |  Connector |   +------------------+
	| Connection +--+ +-+ +-+ +--+  Host      +---+ Docker Container |
	| Manager    |               |  Machine   |   +------------------+
	+------------+               +------------+   |                   
	                                              +------------------+
 	                                              | Docker Container |
 	                                              +------------------+
{% endhighlight %}

Over the other end of our messaging bus I have a service program monitoring buildings from databases and emitting control signals. Then this Docker version of connector would be similiar to traditional node clustering *child-process*.exec(), but spinning up/down Docker containers. After several trials and errors, mostly errors. Here is my final version of Dockerfile:
{% highlight shell linenos %}
FROM centos:7
MAINTAINER Chao Yan <chaoyan@gmail.com>

RUN yum -y install epel-release
RUN yum -y install tar

RUN curl -sL https://rpm.nodesource.com/setup | bash -
RUN yum install -y nodejs
RUN curl http://nodejs.org/dist/v0.12.2/node-v0.12.2-linux-x64.tar.gz | tar xvzf - -C /usr --strip-components=1

RUN yum -y install pptp
RUN yum -y install net-tools

RUN mkdir /etc/ppp/peers

CMD ["/bin/bash"]

{% endhighlight %}

Nothing particular other than setting up Node.js running environment and PPTP.
For test purpose I ran below command:

{% highlight shell %}
docker run -it --rm -v ~/deploy/knx:/knx --cap-add=NET_ADMIN --device=/dev/ppp pptp node /knx/nodeConnectPPTP.js -h client1.dyndns.org -u testuser -p test123
{% endhighlight %}

Notes:

1. **CentOS**: I tested Debian/Ubuntu/CentOS across Azure/AWS and our private data centre, with various VPN client/server combinations. CentOS has the most successful rate to make it through. When I got time I will return to update this part of article with further findings.
1. Detailed explanations of docker test command above:
	- **-it --rm** with console and remove after ctrl-c
	- **-v ~/deploy/knx:/knx** to map node program to host deploy folder
	- **--cap-add=NET_ADMIN --device=/dev/ppp** to give privilege to this container for ppp devices
	- **pptp** image name built from above Dockerfile
	- **node /knx/nodeConnectPPTP.js -h client1.dyndns.org -u testuser -p test123** I call a test program within container to call any target PPTP server.
1. **nodeConnectPPTP.js**: a combination of *pptp-setup* tool, *pppd call* or *pon*, and setup routing to your private network via ppp0 interface, in Node.js, see below.

> nodeConnectPPTP.js : *a test program within container to call a given PPTP server.*

{% highlight javascript linenos %}
var program = require('commander');

program
  .option('-h, --host <value>', 'VPN Server Address')
  .option('-u, --user <value>', 'Username')
  .option('-p, --pass <value>', 'Password')
  .parse(process.argv);

console.log('Dialing PPTP', program.user,'@', program.host);

//=== Prepare option files
var fs = require('fs');
fs.appendFileSync('/etc/ppp/chap-secrets', program.user + ' target ' + program.pass + ' *');
var pptp_opts = 'pty \"pptp ' + program.host + ' --nolaunchpppd\"\nlock\nnoauth\nnobsdcomp\nnodeflate\nname ' + program.user + '\nremotename target\nipparam target\nrequire-mppe-128';
fs.writeFileSync('/etc/ppp/peers/target', pptp_opts);

//=== dial PPPD
var exec = require('child_process').exec;
var pppCallProcess = exec('pppd call target'
  , function (error, stdout, stderr) {
      if (error !== null) {
        console.log('exec error: ' + error);
        process.exit(1);
      }
  });

var num_tries = 10;
var tmrConnectVPN = setInterval(function(){
  exec('ip addr | grep ppp0'
  , function (error, stdout, stderr) {
      var result = '' + stdout;
      if (result.indexOf('inet ') > 0) {
        clearInterval(tmrConnectVPN);

//=== set up ip route
        exec('route add default dev ppp0', function (error, stdout, stderr) {

          console.log('VPN Connected.\r\n', result);

        });
      } else {
        num_tries--;
        if (num_tries <= 0) {
          process.exit(1);
        }
      }
  });
}, 1000);
{% endhighlight %}

And finally, an orchestra daemon made possible by **dockerode** to programmatically spawn VPN container as required.

> dockerController.js

{% highlight javascript linenos %}
var Docker = require('dockerode');
var docker = new Docker();
var _ = require('lodash');

docker.listImages(function(err, images) {
    if (_.find(images, function(img){
      return _.find(img.RepoTags, function(tag) {
        return (tag.indexOf('pptp') == 0);
      })
    })) {
      docker.run('pptp',
                ['/bin/bash', '-c', 'node' + ' ' + '/knx/nodeConnectPPTP.js' + ' ' +
                      '-h ' + "client1.dyndns.org" + ' ' +
                      '-u ' + "testuser" + ' ' +
                      '-p ' + "test1234"], [process.stdout, process.stderr],
              { Tty: false,
                "WorkingDir": "/knx",
                "HostConfig": {
                     "Binds": [ workingDir + ":/knx"],
                     "CapAdd": ["NET_ADMIN"],
                     "Devices": [{ "PathOnHost": "/dev/ppp",
                             "PathInContainer": "/dev/ppp",
                             "CgroupPermissions": "mrw"}],
                   }
              }, function (err, data, container) {
                if (err)
                  console.log(err);
            container.remove(function (err, data) {
                console.log('Container removed: ', container);
            });
    }).on('container', function (container) {
        console.log('Container online: ', container);

        // ...
    });

    } else {
      throw new Error('Could not find pptp docker image')
    }
});


{% endhighlight %}

---

## The Result

This is screenshot from KNXMonitor on three test buildings with different VPN options and conflicting subnet. There is no need to setup redial for dropped connection, our controller(a state machine) would simply kill and restart its container.

![Multi VPN outgoing connection test](/images/knxmonitor/vpn_tests.png)

Thankfully the rise of Docker has made possible a major feature I'd never expect to solve by Docker while learning it for the whole time! The roller-coasting journey led me to the above solution has not been easy. Hopefully this article could help any other fellow coder in any way it could.

Of course in production there is more to details, things like:

- different docker file and configuration programs for PPTP/L2TP/IPSec
- how program running within container communicates with orchestra daemon for starting/stopping/heartbeating. I used socket by exposing host IP to container ports.
- encrypt client VPN credentials transmitted between building hub and connector host.
