# Introduction

## Finding Files
### locate
`updatedb`

`locate sbd.exe`

### which
`which sbd`

### find
#### All the files in entire FS with sbd in name

`find / -name sbd*`

#### All the files with sbd and identify file format

`find / -name sbd* -exec file {} \;`

## Services in Kali

### SSH Service
  `service ssh start`
  `service ssh stop`

#### Verfication
  `netstat -antp | grep sshd`

### HTTP Service
`service apache2 start`
`service apache2 stop`

#### Default location of apache files
/var/www/

### Service Management

#### Usining init scripts
`/etc/init.d/apache2 start`

## Boot Persistence
`update-rc.d ssh enable`

### To get more control
`rcconf`

## Bash Shell
### Scenario 1 - Locate all subdomains

`wget www.cisco.com`

* gets index.html
`less index.html`

* Look for links
  * <a href=....

`cat index.html | grep "href="`
* messy
* look at the fields

`cat index.html | grep "href="| cut -d"/" -f3 | more`
* includes several domain names but also useless entries

`cat index.html | grep "href="| cut -d"/" -f3 | grep "cisco\.com"`

* filters out entries that do not contain cisco.com
* need to get rid of quoted and duplicates


`cat index.html | grep "href="| cut -d"/" -f3 | grep cisco\.com"|cut -d'"' -f1`
* still has duplicates

`cat index.html | grep "href="| cut -d"/" -f3 | grep cisco\.com"|cut -d'"' -f1 | sort -u`
* removes duplicates


`cat index.html | grep "href="| cut -d"/" -f3 | grep cisco\.com"|cut -d'"' -f1 | sort -u > cisco.txt`
* sends output to text for use in script

`host www.cisco.com`
* look for line that says _has address_

`host www.cisco.com | grep "has address" | cut -d" " -f4`
* grabs ip address

`vi cisco.sh`

```bash
#!/bin/bash

for url in $(cat cisco.txt);do
host $url | grep "has address" | cut -d" " -f4
done
```

`for url in $(grep -o '[A-Za-z0-9_\.-]*\.*cisco.com' index.html | sort -u); do host $url|grep "has address"|cut -d" " -f4;done`

## Scenario 2
* Ping sweep

`nano ping-loop.sh`

```bash
#!/bin/bash

for ip in $(seq 200 254); do
  ping -c 1 192.168.31.$ip | grep "bytes from" | cut -d" " -f4 | cut -d ":" -f1 &
done
```
