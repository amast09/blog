---
date: "2017-11-30T21:00:35-05:00"
title: "Command Line Apple Pay CSR's"
description: "Creating Apple Pay, Payment Processing Certificate Requests from the Command Line"
tags: [ "openssl", "ios" ]
categories: [ "Development", "iOS", "How To" ]
type: "post"
---

Setting up Apple Pay is a simple process when using a payment processor that accepts Apple Pay (Square, Stripe, ect).

* Create a merchant ID
* Upload a certificate signing request given by the payment processor into the Apple Developer Portal
* Upload the Apple generated payment processing certificate into the payment gateway's portal
* Enable Apple Pay in the app's configuration

Done, nothing particularly difficult about it.

It becomes tricky when your payment processor does not accept Apple Pay and you need to decrypt your own Apple Payment
Tokens.

Not only is there a LOT of cryptography involved but you also have to create and manage your own certificate signing
requests and generated certificates and keys securely. Did I mention that they expire? :P

Something that can tend to be insecure is a random developer's or client's Mac. Apple's documentation for creating
a Certificate Signing Request for Apple Pay involves using MacOS's keychain program. Because of this, a lazier or more
naive developer may quickly create a CSR (Certificate Signing Request) straight away from their own machine without
thinking about the security implications. Maybe said developer even forget's to delete the file's off of his computer
or out of their local keychain. Any compromise to the machine could possibly compromise all the Apple Pay transactions
sent for that merchant identifier. 

An ideal scenario would be to generate the CSR on a known hardened machine, preferably ephemeral. Unfortunately ephemeral
machines tend not to be MacOS's.

What would be great is if Apple had detailed documentation on creating said CSR
as well as other required decryption artifacts using openssl on the command line. Then we would not be bound to MacOS or
OSX.

I am no crypto guru or openssl expert so I came to the solution through Google-foo and stubbornness. I will do my best
to explain what is going on in each command.

First we will generate the private key for our CSR,
```bash
openssl ecparam -out private.key -name prime256v1 -genkey
```


Apple specifies the CSR should be use ECC (why we use the `ecparam` command) as well as having a key length of 256
(why we have `prime256v1`).

This command creates the key for our CSR, next we will create the CSR itself.

```bash
openssl req -new -sha256 -key private.key -nodes -out request.csr
```

After creating our `request.csr` simply upload the file into the Apple Developer Portal associating it to the correct
Merchant Identifier for your application.

Apple will then hand us back a `.cer` file in return. That certificate combined with a PKCS #12 file (`.p12` file extension)
allow us to actually verify the Apple Payment Token's signature and decrypt it's payload so we can process the transaction
ourselves or pass it along to our payment processor that does not support Apple Pay.

Back to the command line foo.

```bash
openssl x509 -inform DER -outform PEM -in apple_pay.cer -out temp.pem
```

Here we are create a `.pem` file from the `apple_pay.cer` Apple gave us. We will use it to generate our `.p12` file.

```bash
openssl pkcs12 -export -out key.p12 -inkey private.key -in temp.pem
```

This is the final required command, it is generating a `.p12` file based off of the original CSR `.key` file as well as
the `temp.pem` file we just created.

After running the command it will prompt you for a password to protect the `.p12` file with. This should be a strong
safely guarded password since it is a key to decrypting payment data.

After this has been completed we have the 3 ingredients required to decrypt any Apple Payment Token generated for any
app using the associated Merchant Identifier.

1) `apple_pay.cer`

the payment processing certificate Apple gave us

2) `key.p12`

the generated PKCS #12 file based off of our original CSR key and the Apple Payment Processing Certificate

3) `password`

the password to our `key.p12` file

If you are stuck decrypting your own Apple Payment Token's I hope this short guide will help you secure the Apple Pay
Certificate Signing Request process as well as the sensitive decryption artifacts it creates.

Cheers!
