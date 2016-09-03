# Examining the X.509 certificate installed by the NSO Pegasus spyware via the Trident vulnerabilities

---
___
## Contents
1. **Introduction**
  * **1.1 Background**
  * **1.2 Disclaimers**
  * **1.3 Feedback**
2. **Pegasus spyware & Trident vulnerabilities**
3. **Pegasus uses a self-signed root CA certificate**
4. **What can be learned from the certificate?**
5. **CN=Asterisk Private CA, O=My Super Company**
  * **5.1 What is "CN=Asterisk Private CA"?**
  * **5.2 What is "O=My Super Company"?**
  * **5.3 The Asterisk Secure Calling Tutorial and the ast_tls_cert script**
  * **5.4 The tutorial and script defaults vs. Pegasus**
  * **5.5 The script's OpenSSL options vs. Pegasus**
  * **5.6 Command line possibilities**
6. **Unanswered (unanswerable?) questions**
7. **Conclusion**


---
___

## 1. Introduction

### 1.1 Background

This is an examination of a self-signed root CA certificate, **ca.crt**, that was used by the Pegasus 'lawful intercept' tool / spyware in a real-world attack.  Details about the attack itself are further below.

This work was undertaken for several reasons:
* I hadn't read about anyone else specifically examining this certificate to see what information could be gleaned from it, which was a little surprising considering its source and intended use.
* I wanted to see if my amateur research skills could turn up anything interesting or if I would just hit dead ends.
* I'm a student of encryption, TLS, PKI, and related tech, so this seemed like a good opportunity to dive into and learn more about the X.509 format.
* I had some free time to spend on a task that I fully expected to fizzle out after what I thought would be a few hours of fruitless poking around.

Although this examination turned out to be way more interesting, time-consuming, and personally educating than I expected, I don't presume to think that I've found out anything particularly revealing or groundbreaking about NSO or Pegasus.

---

### 1.2 DISCLAIMERS
* I'm very new to OpenSSL, parsing X.509 certs, and making sense of Linux Bash scripts.
* Despite my best efforts to be accurate and thorough, I expect that my newness to these topics has probably led to me making errors or incorrect conclusions.
* I'm not a professional malware researcher or threat analyst (_...yet!_).  Please conduct your own research before relying on any of this information to protect your network or devices.

---

### 1.3 Feedback
* I welcome all criticism and corrections. Please open an issue and provide a good description and links to any resources that will help illustrate the solution.
* Comments and interesting tangents are appreciated.  @ or DM me on Twitter: https://twitter.com/CuanticoSec


---
___


## 2. Pegasus spyware & Trident vulnerabilities

On Aug 24, 2016, the news broke in the world of InfoSec about the capture and analysis of some sophisticated and rare iOS zero-day exploits that were intended to be used against a political dissident, **Ahmed Mansoor**.   The exploits were designed to deliver a so-called lawful intercept tool, the **Pegasus spyware**, that give the attacker a persistent presence on the target's iPhone and allow for complete access to the iPhone's apps, data, and microphones.

The three exploits comprise a linked chain, dubbed the **Trident vulnerabilities**.   Capturing these kind of exploits in the wild is unprecedented, although their existence and use has been assumed true for several years.

All four of these links are worth reading if you work in InfoSec -

**Motherboard** - summary of the event:  https://motherboard.vice.com/read/government-hackers-iphone-hacking-jailbreak-nso-group

**Citizen Lab** - investigation and analysis of NSO's infrastructure and Pegasus-related assets:  https://citizenlab.org/2016/08/million-dollar-dissident-iphone-zero-day-nso-group-uae/

**Lookout** - blog post about the event and their work: https://blog.lookout.com/blog/2016/08/25/trident-pegasus/

**Lookout** - technical analysis of the Pegasus spyware:  https://info.lookout.com/rs/051-ESQ-475/images/lookout-pegasus-technical-analysis.pdf

The Citizen Lab report is a great example of good, extensive OSINT research and how covering your tracks online is difficult without dedicated preparation and attention to detail - aka OPSEC is not retroactive (https://www.youtube.com/watch?v=9XaYdCdwiWU).

---
___

## 3. Pegasus uses a self-signed root CA certificate

From Lookout's technical analysis:

>"The attack is comprised of three separate stages that contain both the exploit code and the espionage software. The
stages are sequential; each stage is required to successfully decode, exploit, install, and run the subsequent stage.
Each stage leverages one of the Trident vulnerabilities in order to run successfully.

Stage 1 is delivered via a specially crafted HTML file that uses a vulnerability in WebKit, used by the Safari browser.  Stage 2 is the exploit package that performs the iOS jailbreak.  Stage 3 is installation of the Pegasus espionage software.  Once installed, Pegasus gathers data from the phone and then exfiltrates it to the command and control (C&C) server(s).

There are many  files installed as part of the Pegasus package but **ca.crt** is the focus of my attention.

The **ca.crt** file is a [self-signed root CA certificate](https://blogs.msdn.microsoft.com/kaushal/2013/01/09/self-signed-root-ca-and-intermediate-ca-certificates/).  Pegasus installs it in the iPhone's keystore so the spyware can create an encrypted connection to its C&C servers.  The encrypted connection protects data in transit and prevents any third-parties in the middle from seeing or tampering with the exact contents of the communication.

The certificate's contents are included at the end of [Lookout's analysis](https://info.lookout.com/rs/051-ESQ-475/images/lookout-pegasus-technical-analysis.pdf) on page 32.

A slightly cleaned-up text version of the certificate's data is provided [here](Pegasus_ca-crt.txt).

---
___

## 4. What can be learned from the certificate?

Notes about the certificate's fields, appearing somewhat out of order:

---

#### **Version: 1 (0x0)**
* The certificate uses Version 1 of the X.509 standard (https://tools.ietf.org/html/rfc1422).  Using Version 1 is usually an indicator of a self-signed certificate.
* Version 3 was formalized in 2008 (https://tools.ietf.org/html/rfc5280) and is the standard for the regular root CA certificates that modern websites rely on to establish HTTPS connections.

---

#### **Serial Number: a9:c2:dc:41:57:dc:50:14**
* No matches in Censys.io or Google/Bing/DDG to the hex (A9C2DC4157DC5014) or decimal (12232581711096729620) version.
* Possibly a randomly assigned value.  From the OpenSSL manual, https://www.openssl.org/docs/manmaster/apps/req.html:
>"Unless specified using the set_serial option, a large random number will be used for the serial number."

---

#### **Signature Algorithm: sha256WithRSAEncryption**
* To get the output **sha256WithRSAEncryption** using the OpenSSL software, the message digest (aka signature algorithm) must be SHA-256.
* The message digest is defined by OpenSSL's built-in default, by a user-editable configuration file, or manually on the command line at the time the certificate is created.
* OpenSSL's built-in default digest is defined here: https://github.com/openssl/openssl/blob/a7be5759cf9d8e2bf7c1ecd0efa2d53aae9ab706/crypto/rsa/rsa_ameth.c#L347-L349
* SHA-256 was added as the default digest to the development codebase on Sep 8, 2014: https://github.com/openssl/openssl/blob/44e0c2bae4bfd87d770480902618dbccde84fd81/crypto/rsa/rsa_ameth.c
* OpenSSL version 1.1.0, which used SHA-256 as the default digest for the first time, was publicly released on **Aug 26, 2016**, over two weeks after the target was sent the malicious link.
* Prior to version 1.1.0, the default digest was MD5.
* Prior to version 1.1.0, using SHA-256 for the digest required either editing OpenSSL's default configuration file or defining it manually on the command line.
* 	The default digest can be set in the configuration file **openssl.cnf** by adding the following under the section `[ req ]`:
  * `default_md = sha256      # use SHA-256 for Signatures`
  *  https://github.com/openssl/openssl/blob/a7be5759cf9d8e2bf7c1ecd0efa2d53aae9ab706/apps/openssl.cnf
* The digest can be defined via the command line with the option `-[digest]` at the time the certificate is generated: https://www.openssl.org/docs/manmaster/apps/req.html

---

#### Issuer: CN=Asterisk Private CA, O=My Super Company
#### Subject: CN=Asterisk Private CA, O=My Super Company
* The Issuer matches the Subject.
* The Issuer is not a public certificate authority (DigiCert, Comodo, Verisign, etc).
* This is self-signed and self-generated root CA certificate.
* **See the next section for further information about the specific values.**

---

#### Validity - Not Before: Jul 18 11:21:53 2016 GMT
* The certificate was generated **23 days** before the exploits were sent to the target.


#### Validity - Not After: Jul 17 11:21:53 2021 GMT
* The certificate expires after five years.
* The OpenSSL default expiration period for CA certificates is one year: https://github.com/openssl/openssl/blob/a7be5759cf9d8e2bf7c1ecd0efa2d53aae9ab706/apps/openssl.cnf#L73

---

#### Public Key Algorithm: rsaEncryption
* Uses the RSA algorithm: https://simple.wikipedia.org/wiki/RSA_(algorithm)

---

#### RSA Public Key: (4096 bit)
* OpenSSL default is 512: https://www.openssl.org/docs/manmaster/apps/genrsa.html


#### Modulus (4096 bit)
  * Hex version of the public key

#### Exponent: 65537 (0x10001)
  * OpenSSL default is 65537: https://www.openssl.org/docs/manmaster/apps/genrsa.html

#### Signature Algorithm: sha256WithRSAEncryption
  * See notes above


---
___


## 5. CN=Asterisk Private CA, O=My Super Company

Why do those values appear in this certificate?

Possibly because the creator of the Pegasus **ca.crt** file used the Asterisk communications toolkit (http://www.asterisk.org/get-started) to create it.

**Asterisk Private CA** is the default Common Name that is applied by Asterisk when a built-in Bash script is used to create a self-signed root CA certificate.  

**My Super Company** is the example Organization Name that is included in an Asterisk tutorial for creating the certificates for encrypted VoIP calls.

---

### 5.1 What is "CN=Asterisk Private CA"?

CN stands for Common Name.  For legitimate certificates used by websites this field should take the form of `subdomain.example.com`.  For self-generated certs, it can be anything the creator defines.

The earliest Google search result for **CN=Asterisk Private CA** is from Oct 21, 2010:  https://reviewboard.asterisk.org/r/979/diff/

That page contains the original version of a Linux Bash script that's been incorporated info Asterisk.  The script invokes OpenSSL to create the different X.509 certificates and files that Asterisk uses to encrypt VoIP calls.  **Asterisk Private CA** is a default value included in the script that is applied to root CA certificates if the user does not override it.  See **Section 5.3** for more details about the script.


Searching Censys.io (https://censys.io/certificates?q=%22Asterisk+Private+CA%22) for certificates containing **Asterisk Private CA** yields 143 results.

I did not dig in to those Censys search results any further.  Perhaps there are links between some of those results and the information gathered by Citizen Lab, but searching for them is beyond my agenda.

---
### 5.2 What is "O=My Super Company"?

O stands for Organization Name.  For legitimate certificates used by websites this field often appears as some variation of  `Name_of_Company_that_owns_the_website`.  For self-generated certs, it can be anything the creator defines.

The earliest Google search results for **O=My Super Company** are from Jan 24, 2011 (http://www.siplab.cn/ast/Asterisk-Admin-Guide/Secure-Calling-Tutorial_8127019.html) and Jan 28, 2011 (https://wiki.asterisk.org/wiki/display/AST/Secure+Calling+Tutorial).

The pages are essentially the same.  They are a tutorial for using Asterisk  to generate the root CA, server, and client TLS certificates that are used for encrypting VoIP calls.  The tutorial provides some example values - including **My Super Company** - to illustrate how to use a built-in Bash script to create the certificates.  See **section 5.3** for more details about the tutorial and the script.

Searching Censys.io (https://censys.io/certificates?q=%22My+Super+Company%22 and https://censys.io/certificates?q=parsed.issuer.organization%3A+%22My+Super+Company%22) for **My Super Company** yields only two results.  Neither of those two certs appear to have anything in common with the Pegasus certificate beyond the Issuer field.

---
### 5.3  The Asterisk Secure Calling Tutorial and the `ast_tls_cert` script

To allow encrypted VoIP calls, the admin of an Asterisk server can generate their own root CA, server, and client TLS (X.509) certificates.

Asterisk provides a tutorial for creating the certificates here:
	https://wiki.asterisk.org/wiki/display/AST/Secure+Calling+Tutorial

Instructions and example code from the tutorial for making a self-signed root CA certificate:
>Next, use the "ast_tls_cert" script in the "contrib/scripts" Asterisk source directory to make a self-signed certificate authority and an Asterisk certificate.
>
> `./ast_tls_cert -C pbx.mycompany.com -O "My Super Company" -d /etc/asterisk/keys`




* -C defines the Common Name
* -O defines the Organization name
* -d defines the output directory

The certificate generation script `ast_tls_cert` from Asterisk version 13, the latest release, is available here:  
	http://svn.asterisk.org/svn/asterisk/branches/13/contrib/scripts/ast_tls_cert

The script in version 13 is almost entirely unchanged from when it was introduced in version 1.8.  Barring a one-version variation in the script (and I have not reviewed the script in every single version to check for this), any version of Asterisk could have been used to create the certificate.

Asssuming the `ast_tls_cert` script was used to generate the Pegasus certificate, what other options or values were used to produce the output in the certificate's fields?

---
### 5.4 The tutorial and script defaults vs. Pegasus


**The script's defaults for CA certificates:**
```
DEFAULT_ORG="Asterisk"
DEFAULT_CA_CN="Asterisk Private CA"
```

* The tutorial provides **pbx.mycompany.com** as the Common Name, but the Pegasus certificate has **Asterisk Private CA**.  If not defined on the command line, the script defaults the CA certificate Common Name to **Asterisk Private CA**.
  * ∴	If the `ast_tls_cert` script was used to make the certificate, the script was not copied exactly from the tutorial.  Either the option -C was not defined in the command line and the script's default was used, or else -C was defined in the command line as **Asterisk Private CA**.

* The tutorial provides **My Super Company** for the Org Name, and the Pegasus certificate uses **My Super Company**.  However, the script defaults the Org Name to **Asterisk** if no Org Name is defined in the command line.  The script does not include the string **My Super Company** at all.
  * ∴	If the `ast_tls_cert` script was used to make the certificate, the  creator included the option `-O "My Super Company"`.
---
### 5.5 The script's OpenSSL options vs. Pegasus

**The script contains an entry to generate an RSA private key:**
```
echo "Creating CA key ${CAKEY}"
openssl genrsa -des3 -out ${CAKEY} 4096 > /dev/null
```

* The script's default OpenSSL options, as defined by https://www.openssl.org/docs/manmaster/apps/genrsa.html:
  * genrsa
    * generates an RSA private key
  * des3
    * encrypt the private key with the Triple DES cipher
  * out {}
    * output the key to the specified file
  * 4096
    * size of the private key to generate in bits
  * \> /dev/null
    *	prevents output from being displayed to the user

---

**The script contains an entry to generate a self-signed CA certificate:**
```
echo "Creating CA certificate ${CACERT}"
openssl req -new -config ${CACFG} -x509 -days 365 -key ${CAKEY} -out ${CACERT} > /dev/null
```

* The script'sdefault OpenSSL options, as defined by https://www.openssl.org/docs/manmaster/apps/req.html:

  * req
    * used to create self-signed root CA certificates
  * new
    * generates a new certificate request and prompts user for certificate field values unless defined in the **openssl.cnf** configuration file
  * config {}
    *	allows for alternative config file to be used that may override some OpenSSL defaults
    * Asterisk's default certificate config file, **ca.cfg**, only defines the config file name, the CA Common Name, and the Org Name
  * x509
    * outputs a self-signed certificate instead of a certificate request
  * days {}
    *	number of days the certificate is valid
    * OpenSSL default is 365
  * key {}
    * file to read the private key from
    * Asterisk script creates this file in the previous step
  * out {}
    * output file name
    * Asterisk default is **ca.crt**
  * \> /dev/null
    *	prevents output from being displayed to the user

---

#### **OpenSSL options needed in order to generate the Pegasus certificate:**
  * **genrsa**
    * 4096
      * make the public key 4096 bits
  * **req**
    * x509
      * make a self-signed certificate
    * -days 1826
      * make it valid for 5 years
    * -sha256
      * override the default MD5 message digest if the creator used an OpenSSL version older than 1.1.0


---

### 5.6 Command line possibilities

**1)** The script included in the Asterisk software was used as the starting point but the certificate fields and some of the OpenSSL options were changed from the script defaults using either the command line at creation time or pre-configured in **openssl.cnf**.

Possible script command:
```
./ast_tls_cert -C "Asterisk Private CA" -O "My Super Company" -d /etc/asterisk/keys
```

Possible OpenSSL command:
```
openssl req -new -config ${CACFG} -x509 -days 1826 -sha256 -key ${CAKEY} -out ${CACERT}
```


**2)**	The Asterisk script was not used as the starting point.  Instead, the certificate creator borrowed only the "CN=Asterisk Private CA, O=My Super Company" values, and all other options were configured on the command line or in openssl.cnf.

Exploring this option is beyond my agenda.  

---
___

## 6. Unanswered (unanswerable?) questions:

#### **Asterisk Private CA & My Super Company**
  * Why does the Pegasus spyware use a certificate with a non-unique or non-traceable Common Name and Org Name?
  * Why is the certificate's Common Name the Asterisk script's default value but the Org Name is a user-supplied value?

#### **Asterisk software**
  * Is it possible to install Asterisk, customize the script and OpenSSL, and create a **ca.crt** file that essentially matches the Pegasus certificate (except for the public key)?
  * Assuming the Asterisk script was used to create the certificate, what benefits does it provide over using OpenSSL by itself?
  * Did the Pegasus spyware communicate out to a C&C server that was running Asterix?
  * If so, is the C&C server still operational?
  * Shodan searches for Asterisk (default port for VoIP: 5061):
    * Asterisk (https://www.shodan.io/search?query=Asterisk): 34,801 results
    * Asterisk PBX (https://www.shodan.io/search?query=Asterisk+PBX): 28,288 results
    * Asterisk port:5061 (https://www.shodan.io/search?query=Asterisk+port%3A5061): 3 results


#### **Serial Number**
  * Is this number within OpenSSL's range of possible values?


#### **Validity**
  * The certificate was created 23 days before the exploits were sent to the target.  Is 23 days from  certificate generation to attempted target exploitation normal, hurried, or delayed?
  * Was the certificate generated by NSO or by their customer?  The answer to this may shed light on the attack timetable.
  * The OpenSSL default expiration time period is one year. Why was five years chosen?  Was the Pegasus malware expected to persist longer than a year?  Was it not expected to go past five?

#### **Signature Algorithm**
  * Was the certificate creator savvy enough to manually enable SHA-256 to prevent network eavesdroppers from decrypting captured transmissions, but lazy enough to not create disposable a Common Name or Org Name?

#### **Meta**
  * What else can be inferred from the Pegasus certificate?
  * Why didn't NSO craft the certificate so that it wouldn't reveal any identifying information about the threat actor or their methods?
  * Does NSO (or their client) care that the certificate isn't entirely unrelated to anything else on the Internet?


---
___

## 7. Conclusion

The root CA certificate installed by the Pegasus spyware was intended to be used to encrypt communications to its C&C server.  I was curious to see what information might be discovered on the Internet using the certificate as the starting point.

Using a self-signed root CA certificate with your custom, expensive, nation-state-only espionage software makes sense.  Buying a certificate from a regular public CA establishes a payment and customer trail that is just one more complicated operational aspect that needs to be sanitized or backstopped.  A self-signed certificate can be created with disposable identification data that doesn't have to link back to anything.

However, the Pegasus certificate's Common Name and Organization Name fields are not random or disposable values. They originate from a tutorial and script used by the Asterisk communications software.  According to Google and Censys.io, the values have appeared in many other (probably legitimate) certificates, but the combination of the two in the same certificate appears to be unique.

Using a self-signed certificate that contains data that reveals how the certificate was generated - **Asterisk Private CA** and **My Super Company** - or perhaps reveals details about your C&C server - **Asterisk PBX over port 5601** - doesn't seem very wise.  Small tidbits of seemingly unimportant data like that might turn out to be very useful to a well-placed, patient, or sophisticated adversary.

Perhaps the use of Asterisk is inconsequential.  I'm not knowledgable enough to say whether the use of **Asterisk** and **My Super Company** is buried treasure or an absolute non-issue.  I can't help but notice that those terms are not mentioned at all in the reports from Citizen Lab or Lookout.  Have I been on a wild herring chase (likely), or did they happen to overlook this minor aspect of the spyware (unlikely)?

---

Regardless of the results of my digging around, this was an educational use of my time. I learned a lot about the X.509 format, got a nice intro to OpenSSL's capabilities, and put my recently-acquired-yet-still-very-beginner knowledge of Linux Bash scripting to the test.


---
___
