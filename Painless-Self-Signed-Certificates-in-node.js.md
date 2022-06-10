# Painless Self Signed Certificates in node.js

# Try the code

I made a complete, cloneable example:

https://git.coolaj86.com/coolaj86/nodejs-self-signed-certificate-example

# TL;DR

If you don't like to read you can just **copy and paste** this into your terminal.

## Create your Root CA and your Signed Certificate

**STOP**: There is one thing you need to change: Replace `CN=local.ldsconnect.org` with your domain. 

**HOWEVER**, `local.ldsconnect.org` points to `127.0.0.1`, so this example will work if you simply copy and paste with 0 modifications.

```bash
# make directories to work from
mkdir -p server/ client/ all/

# Create your very own Root Certificate Authority
openssl genrsa \
  -out all/my-private-root-ca.key.pem \
  2048

# Self-sign your Root Certificate Authority
# Since this is private, the details can be as bogus as you like
openssl req \
  -x509 \
  -new \
  -nodes \
  -key all/my-private-root-ca.key.pem \
  -days 1024 \
  -out all/my-private-root-ca.crt.pem \
  -subj "/C=US/ST=Utah/L=Provo/O=ACME Signing Authority Inc/CN=example.com"

# Create a Device Certificate for each domain,
# such as example.com, *.example.com, awesome.example.com
# NOTE: You MUST match CN to the domain name or ip address you want to use
openssl genrsa \
  -out all/my-server.key.pem \
  2048

# Create a request from your Device, which your Root CA will sign
openssl req -new \
  -key all/my-server.key.pem \
  -out all/my-server.csr.pem \
  -subj "/C=US/ST=Utah/L=Provo/O=ACME Tech Inc/CN=local.ldsconnect.org"

# Sign the request from Device with your Root CA
openssl x509 \
  -req -in all/my-server.csr.pem \
  -CA all/my-private-root-ca.crt.pem \
  -CAkey all/my-private-root-ca.key.pem \
  -CAcreateserial \
  -out all/my-server.crt.pem \
  -days 500

# Put things in their proper place
rsync -a all/my-server.{key,crt}.pem server/
rsync -a all/my-private-root-ca.crt.pem server/
rsync -a all/my-private-root-ca.crt.pem client/
```
The only **3 files** you need **on your server** are these:

```bash
server
├── my-private-root-ca.crt.pem
├── my-server.crt.pem
└── my-server.key.pem
```

The **1 file** you need **on your clients** is this:

```bash
client
└── my-private-root-ca.crt.pem
```

## Your server

```javascript
#!/usr/bin/env node
'use strict';

var https = require('https')
  , port = process.argv[2] || 4443
  , fs = require('fs')
  , path = require('path')
  , server
  , options
  ;

require('ssl-root-cas')
  .inject()
  .addFile(path.join(__dirname, 'server', 'my-private-root-ca.crt.pem'))
  ;

options = {
  key: fs.readFileSync(path.join(__dirname, 'server', 'my-server.key.pem'))
// You don't need to specify `ca`, it's done by `ssl-root-cas`
//, ca: [ fs.readFileSync(path.join(__dirname, 'server', 'my-private-root-ca.crt.pem'))]
, cert: fs.readFileSync(path.join(__dirname, 'server', 'my-server.crt.pem'))
};


function app(req, res) {
  res.setHeader('Content-Type', 'text/plain');
  res.end('Hello, encrypted world!');
}

server = https.createServer(options, app).listen(port, function () {
  port = server.address().port;
  console.log('Listening on https://127.0.0.1:' + port);
  console.log('Listening on https://' + server.address().address + ':' + port);
  console.log('Listening on https://local.ldsconnect.org:' + port);
});
```

## Your client

```javascript
#!/usr/bin/env node
var https = require('https')
  , fs = require('fs')
  , path = require('path')
  , ca = fs.readFileSync(path.join(__dirname, 'client', 'my-private-root-ca.crt.pem'))
  ;

var options = {
  host: 'local.ldsconnect.org',
  path: '/',
  ca: ca
};
options.agent = new https.Agent(options);

https.request(options, function(res) {
  res.pipe(process.stdout);
}).end();
```

# What you need to know

I struggled for a bit with self-signed certificates until I found out that **YOU'RE NOT SUPPOSED TO SERVE SELF-SIGNED CERTIFICATES**.

Seriously. They're not supposed to be public-facing. And that's why you get some many various types of errors that are difficult to resolve such as `SSL certificate problem: Invalid certificate chain` and `DEPTH_ZERO_SELF_SIGNED_CERT`.

The purpose of self-signed certificates is for Root CAs to hide them away in a safe place and occasionally sign 2nd-tier certificates which sign 3rd-tier certificates which sign... and so forth until you pay just $24.97 for an Nth-tier certificate that only works in the most recent browsers and devices.

So here's how to win:

* Create a CA - **KEEP IT PRIVATE** (your server never sees it except the `.crt.pem`)
* Self-sign your CA
* Create a **server certificate**
* Sign your public-facing cert **THIS IS PUBLIC** (but none of it ever leaves your server and the clients never see any of it)

## Create A Certificate Authority

Since this is for **private** use and testing it doesn't much matter if the information is correct or not.

If you want, you can just copy and paste this directly:

```bash
openssl genrsa \
  -out my-private-root-ca.key.pem \
  2048
```

That creates a key *without a passphrase*. If you want to protect this key with a passphrase, add the option `-des3`.

## Sign your Certificate Authority with itself

Normally you have to create a signing request (csr.pem) and then have it signed. This does both in one step.

```bash
openssl req \
  -x509 \
  -new \
  -nodes \
  -key my-private-root-ca.key.pem \
  -days 1024 \
  -out my-private-root-ca.crt.pem \
  -subj "/C=US/ST=Utah/L=Provo/O=ACME Signing Authority Inc/CN=example.com"
```

* `-new` means your generating a new signature, this is why you don't have to provide an infile
* `-key` is the key you're using to sign it
* `-nodes` means "no des" or "don't encrypt with a des cipher" or, most simply, "no password"

If you want to keep this cert in a safe place and sign lots of stuff with it, then you should put a passphrase on it.

## Create your SERVER cert

Looks pretty familiar, eh?

```bash
openssl genrsa \
  -out my-server.key.pem \
  2048
```

## Create a signing request (csr.pem)

It's important to note that **the CN MUST match YOUR domain**.

If I want to use this certificate with `api.example.com` then it must say exactly that, `example.com` will not work. And although `*.example.com` is a valid wildcard for `api.example.com`, it will not work for `example.com` (you would need two certificates, and SNL vhosting or the [v3_req subjectAltName extension](http://techbrahmana.blogspot.com/2013/10/creating-wildcard-self-signed.html))

**CN** may also be an IP address.

```bash
openssl req -new \
  -key my-server.key.pem \
  -out my-server.csr.pem \
  -subj "/C=US/ST=Utah/L=Provo/O=ACME Tech Inc/CN=awesome.example.com"
```

I recommend using a different `O`rganization name, just so that it's easier to spot and debug at-a-glance in browsers and whatnot.

## Sign your server cert with your CA

```bash
openssl x509 \
  -req -in my-server.csr.pem \
  -CA my-private-root-ca.crt.pem \
  -CAkey my-private-root-ca.key.pem \
  -CAcreateserial \
  -out my-server.crt.pem \
  -days 500
```

* `-days` should be fewer than you specified in your certificate authority

# Appendix

**pem**: Just so you know, *pem* means **plain-text format** (but the acronym is something about email and mime types)

Other SSL Resources
=========

Zero-Config clone 'n' run (tm) Repos:


* [io.js / node.js HTTPS SSL Example](https://github.com/coolaj86/nodejs-ssl-example)
* [io.js / node.js HTTPS SSL Self-Signed Certificate Example](https://git.coolaj86.com/coolaj86/nodejs-self-signed-certificate-example)
* [io.js / node.js HTTPS SSL Trusted Peer Client Certificate Example](https://github.com/coolaj86/nodejs-ssl-trusted-peer-example)
* [SSL Root CAs](https://github.com/coolaj86/node-ssl-root-cas)

Articles

* [http://greengeckodesign.com/blog/2013/06/15/creating-an-ssl-certificate-for-node-dot-js/](Creating an SSL Certificate for node.js)
* [http://www.hacksparrow.com/express-js-https-server-client-example.html/comment-page-1](HTTPS Trusted Peer Example)
* [How to Create a CSR for HTTPS SSL (demo with name.com, node.js)](https://coolaj86.com/articles/how-to-create-a-csr-for-https-tls-ssl-rsa-pems/)
* [coolaj86/Painless-Self-Signed-Certificates-in-node](https://github.com/coolaj86/ssl-root-cas.js/master/Painless-Self-Signed-Certificates-in-node.js.md)