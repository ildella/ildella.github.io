---
title: Deconstructing Bitwarden Part 1 - password
tags:
  - security
  - cryptography
  - programmming
  - nodejs
date: 2019-11-18 10:47:36
---


In December 2018 I removed all my passwords from Firefox built-in manager and started using [Bitwarden](https://bitwarden.com/).

* Note: I am publishing this in November 2019 despite having written this almost a year ago. For some reason.

{% blockquote @ildella https://twitter.com/ildella/status/1076889952250707969 %}
Today is the day I remove all my passwords from Firefox and go full @bitwarden_app.
{% endblockquote %}

Bitwarden is a password manager like LastPass and others. It offers the usual browsers extensions and an enterprise plan, plus:

  * is Open Source 
  * has a command line interface
  * you can host your own server

## How Bitwarden works

As cryptography and security is my most recent interest in software engineering, I was curious on how Bitwarden works. 

{% asset_img "cover.svg" "" %}

In a nutshell:

  1. creates a JSON storage (called data.json) with all the user data
  2. This data.json has encrypted values. The encryption is what we'll understand in this series of articles
  3. Bitwarden sync data.json with a remote server. This allows different clients to sync with the same data the browser extensions as well as the command line application. 
  4. This is different from how a normal Web Application works. The data are not "served" from the server each time, they are synced between the clients and read from a local file.
  5. As far as I understand, there is only one master password that will grant access to the Bitwarden account and the same password is used to encrypt/decrypt data. 

This last part is something that I am not completely at peace at. I would have used two different passwords. 
A partial explanation I have for now is that the key used to encrypt/decript data is not the actual password, but a token that changes each session, that is based on the password but is not the password. 

I need to dig more into the details to understand this better. 

## Bitwarden security

As the code is [Open Source on Github](https://github.com/bitwarden) I looked at the CLI as building interactive CLI apps is another recent passion of mine to understand some details.

Security in Bitwarden is a complex and a non trivial topic of course. But it's possible and educational to understand it one piece at a time.

The main task is being able to unlock my local database and then decipher the encrypted data. This actually consist of 2 parts: hash the password to match Bitwarden hash, then generate and store a valid session token that is used to decrypt data.

In this post I'll explain the first part.

## Part 1 - Hash the password

The first thing we need is being able to hash the password to match the one stored in the local data. 

I went through the implementation of [Bitwarden CLI](https://github.com/bitwarden/cli) which, being written in TypeScript and with an object oriented style is a little... verbose for my taste. It's very well sorted, still I get lost between lots of Interfaces, Objects, Generics and many buffer to array buffer to string transformations... but in the end the code is simple and concise, despite being scattered around many files.

Turns out the relevany code, after some preparation / requirements, is just **3 lines**.

First, we require a few things from Node.js builtin modules:

```javascript
const {promisify} = require('util')
const fs = require('fs').promises
const homedir = require('os').homedir()
const crypto = require('crypto')
const pbkdf2 = promisify(crypto.pbkdf2)
```

Then we need the user email that will be used as "salt" in the code that will follow. 
The email is stored in clear in the local Bitwarden data.json file:

```javascript
const bwFile = await fs.readFile(`${homedir}/.config/Bitwarden CLI/data.json`)
const bwConfig = JSON.parse(bwFile.toString())
const salt = bwConfig.userEmail
```

Finally, let's get to the ciphers!

Now, **pbkdf2** [(Password-Based Key Derivation Function 2)](https://nodejs.org/api/crypto.html#crypto_crypto_pbkdf2_password_salt_iterations_keylen_digest_callback), as far as I undertand, strengthen the hash we create from the password, making lots of iterations of an HMAC function. 
The details on how and why are for cipherpunks and another type of article (from someone who actually understand about cryptograpyhy...), but is an accepted thing. In fact is part of builtin `crypto`  module in Node.js. 

What Bitwarden does is applying the pbkdf2 function twice, the second time using the first hash as password and the original password as salt. It's easier to read the code:

```javascript
const algorithm = 'sha256'
const length = 32
const hashed = await pbkdf2(password, salt, 5000, length, algorithm)
const rehashed = await pbkdf2(hashed, password, 1, length, algorithm)
return Buffer.from(rehashed).toString('base64')
```

The length is 32 because the algorithm is 256 bit. Had Bitwarden used a 512 bit algorithm  then lenght would have been 64.

5000 and then 1 are the number of iterations. 

Now we have the base64 version of the password, that maches the field `keyHash` in Bitwarden data.json

## What is next

The next part is about generating a session token (easy) and storing it encrypted in a proper way in the data.json (not easy).

When that is done, I can use the bitwarden-cli itself with the session key I generated from my own code to read the actual data.json. 

At least that's what I think now, I only am halfaway throug it :)

To the next article!
