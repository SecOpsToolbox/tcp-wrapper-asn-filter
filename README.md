<h1 align="center">
	<a href="https://github.com/WolfSoftware">
		<img src="https://raw.githubusercontent.com/WolfSoftware/branding/master/images/general/banners/64/black-and-white.png" alt="Wolf Software Logo" />
	</a>
	<br>
	TCP wrapper to filter by AS Number
</h1>


<p align="center">
	<a href="https://travis-ci.com/SecOpsToolbox/tcp-wrapper-asn-filter">
		<img src="https://img.shields.io/travis/com/SecOpsToolbox/tcp-wrapper-asn-filter/master?style=for-the-badge&logo=travis" alt="Build Status">
	</a>
	<a href="https://github.com/SecOpsToolbox/tcp-wrapper-asn-filter/releases/latest">
		<img src="https://img.shields.io/github/v/release/SecOpsToolbox/tcp-wrapper-asn-filter?color=blue&style=for-the-badge&logo=github&logoColor=white&label=Latest%20Release" alt="Release">
	</a>
	<a href="https://github.com/SecOpsToolbox/tcp-wrapper-asn-filter/commits/">
		<img src="https://img.shields.io/github/commits-since/SecOpsToolbox/tcp-wrapper-asn-filter/latest.svg?style=for-the-badge&logo=github&logoColor=white" alt="Commits since release">
	</a>
	<a href="LICENSE.md">
		<img src="https://img.shields.io/badge/license-MIT-blue?style=for-the-badge&logo=read-the-docs&logoColor=white" alt="Software License">
	</a>
	<br>
	<a href=".github/CODE_OF_CONDUCT.md">
		<img src="https://img.shields.io/badge/Code%20of%20Conduct-blue?style=for-the-badge&logo=read-the-docs&logoColor=white" />
	</a>
	<a href=".github/CONTRIBUTING.md">
		<img src="https://img.shields.io/badge/Contributing-blue?style=for-the-badge&logo=read-the-docs&logoColor=white" />
	</a>
	<a href=".github/SECURITY.md">
		<img src="https://img.shields.io/badge/Report%20Security%20Concern-blue?style=for-the-badge&logo=read-the-docs&logoColor=white" />
	</a>
	<a href=".github/SUPPORT.md">
		<img src="https://img.shields.io/badge/Get%20Support-blue?style=for-the-badge&logo=read-the-docs&logoColor=white" />
	</a>
</p>

## Overview

This is a [TCP wrapper](https://en.wikipedia.org/wiki/TCP_Wrappers) which will filter server connection attempts based on the AS Number that the IP is assigned to. It can be configured in one of two different ways:

1. Allow connections only from a specified list of AS Numbers.
1. Deny connections EXCEPT those from a specified list of AS Numbers.

This allows for the dynamic blocking (or allowing) of all IPs belonging to an AS Number without having to manage or maintain IP lists, which can often be very large.

### What is an AS Number (ASN)?

An Autonomous System (AS) is a group of one or more IP prefixes (lists of IP addresses accessible on a network) run by one or more network operators that maintain a single, clearly-defined routing policy. ASNs are important because the ASN uniquely identifies each network on the Internet.

There is approximately 100,000 unique AS Numbers in use today.

### Security

The use of TCP wrappers does not eliminate the need for a properly configured firewall. This script should be seen as **part** of your security solution, **not** the whole of it.

### Prerequisites

This tool relies on **geoiplookup**, if it is not installed then the script will log an error and **allow** the connection, even if the default action is **DENY**. The reason for this is that without this **ALL** connections would be blocked including your own *(which would be bad)*.

#### Installing the Prerequisites

> This may require additional apt or yum sources depending on your distribution.

<b>Debian / Ubuntu</b>

```shell
# apt-get install geoip-bin geoip-database geoip-database-extra
```

<b>CentOS / RHEL</b>

```shell
# yum install GeoIP GeoIP-data
```

By default this will install the *free* version of the GeoLite Country binary database (GeoIP.dat etc.), usually in the /usr/local/share or /usr/share directory. The specific location doesn't matter as the geoiplookup command will know where to look for the data files.

> We currently do not support GeoIP2 format (mmdb) or automated updates from MaxMind, although is this on the roadmap for this tool.

#### Testing the Prerequisites

Look up one of Google’s IPs.

```shell
# geoiplookup 74.125.225.33
GeoIP Country Edition: US, United States
GeoIP ASNum Edition: AS15169 Google LLC
```

> If you see the above or similar then geoiplookup is installed and working.

### Configuration

Although this was developed for use with sshd, the principle should work for any service that is supported by TCP wrappers, however in this documentation we will use sshd.

#### Install the filter

Copy the [script](src/asn-filter.sh) to /usr/local/sbin/asn-filter (and ensure that it is executable [*chmod +x]*).

Out of the box the ASNS list is empty and the script has the default [`ACTION`](src/asn-filter.sh#L29) of `DENY` (only block AS Numbers in the list), so the net effect at this point is to block nothing.

#### Adding AS Numbers

To add AS Numbers to the list, add them to the [`ASNS`](src/asn-filter.sh#L26) variable. This is a space separated list of AS Numbers.

There are times where an AS Number cannot be identified, if you want to block all entries where an AS Number cannot be identified, add `XX` to the [`ASNS`](src/asn-filter.sh#L26) variable.

#### Allow or Deny

By default the script will deny connections from any AS Number listed in the [`ASNS`](src/asn-filter.sh#L26) variable, however you can invert this logic and only allow connections from these AS Numbers, by setting the [`ACTION`](src/asn-filter.sh#L29) variable to `ALLOW`.

If you change the default [`ACTION`](src/asn-filter.sh#L29) to `ALLOW`, ensure your own AS Number is in the list of AS Numbers before you do this, otherwise you will no longer be able to connect to your server. This won't effect existing open connections, so test with a new connection attempt to ensure the configuration is correct.

#### Process Ordering

In Linux/Unix based systems the processing order for TCP wrappers is as follows:

1. hosts.allow
2. hosts.deny

This means that anything that is not handled (allowed / denied) by hosts.allow will be handled by hosts.deny.

#### /etc/hosts.allow

The following configuration will tell the system to pass all IPs, for ssh connections, to the asn-filter. The return code of the filter specifies the action to be taken.

1. 0 = Success - allow the connection.
2. 1 = Failure - deny the connection.

```shell
sshd: ALL: aclexec /usr/local/sbin/asn-filter %a 
```

> aclexec tells the system to execute the following script and %a is replace by the current IP address.

#### /etc/hosts.deny

The following configuration will tell the system to deny all ssh connections. 

```shell
sshd: ALL
```

> This should never be reached because all cases should be handled by the asn filter, but as with all security configurations **protection in depth** is key and having a safe / secure fallback position is preferable.

## Alternatives

We provide a number of different [TCP Wrapper filters](https://github.com/SecOpsToolbox?q=in%3Aname+tcp+wrapper+filter&type=&language=).

## Multiple Rules

If you wish to use more than one of our TCP Wrappers then please refer to our [TCP Wrapper Multiplexer](https://github.com/SecOpsToolbox/tcp-wrapper-multiplexer).

## Contributors

<p>
	<a href="https://github.com/TGWolf">
		<img src="https://img.shields.io/badge/Wolf-black?style=for-the-badge" />
	</a>
</p>

## Show Support

<p>
	<a href="https://ko-fi.com/wolfsoftware">
		<img src="https://img.shields.io/badge/Ko%20Fi-blue?style=for-the-badge&logo=ko-fi&logoColor=white" />
	</a>
</p>
