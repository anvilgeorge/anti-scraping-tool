**Please read**

Substantive revisions since most recent release:
 - Jan 2, 2025 - see packet filtering near the bottom of the file

Tools to prevent web scraping

The tools first identify the ip address of the party you would like to ban, ie. an ip address that is making a significant number of web requests. It then provides all of the ip addresses associated with that party. Those addresses are then added to the packet filter and blocked.

This is not meant to be a plug and play tool. You will need to adapt it for your environment.

requirements: perl and p5-Net-CIDR as well as a copy of ip2asn-v4.tsv 

my environment: freebsd 13.2, apache 2.4, pf packet filter, perl and bourne shell

The first 3 files are bsd command line shell scripts that are run in order:

    ad - This will report the number of requests for each ip address, sorted by number of requests.
    ip2r - This will provide the name of the registrant for the ip address that you would like to ban
    r2ip - This will give you the other ips associated with the registrant
    I then update my packet filtering blocking system with all of the ip addresses associated with the registrant I want to ban.

For the script files you will need to give them execute permission e.g. chmod 700 ad

You may also have to change the first line of each file, #!/bin/sh, depending on your set up.

any references to xxx, yyy or zzz need to be replaced by an actual ip address octet  eg. replace xxx.yyy.zzz by 154.232.12 as appropriate.

**file: ad** (for accessday) 

Usage: ./ad [-d days] [-o octets]

days is the number of days prior, 0 - today, 1 - yesterday, 2 - the day before

day default: 0, i.e. today, if -d is not specfied

Octets is the number of octets that are used to count requests. eg. all requests under xxx.yyy will be reported if 2 is specified for octets and xxx.yyy.zzz if 3 is specified.  Some scrappers will use multiple ip addresses with the same first 2 octets to avoid detection and that is where reporting based on the first 2 octets can be helpful.  

You will generally need the 3 octets of an ip address in order to properly identify the registrant using the scripts that follow.  2 octets are only helpful to identify scrapers that are using multiple ip addresses with different 3 octets but share to same leading 2 octets. If you identify a 2 octet ip address of interest, then rerun the script with 3 octets and identify a 3 octet ip address with the same leading 2 octets.  Use that 3 octet ip address to then identify the registrant with the following scripts.

Octets default: 3 octets, if -o is not specfied

The script will search your httpd access log and pull all of the access requests for that day and provide the number of accesses for each ip address. 

The output reading across is:

	- the number of requests 
	- the first set of octets of the ip addresses associated with the requests.

The script assumes your httpd access log is located at /var/log/httpd-access.log. You may need to change that based on your setup. This can be changed by changing log_location='/var/log/httpd-access.log' at the top of the file.  Ensure you have read access to this file.

It currently will search the last 500,000 entries of your access log. You can change this as needed by changing the number in "lines=500000" at the top of the file

the ^xxx.xxx.xxx and ^yyy.yyy.yyy in the exclude variable are the first 3 (can be 2 or 4 as well) octets of ip addresses you do not want reported, such as your own or a search crawler you do not want to block. You can delete and add more as needed, for example by adding "|^zzz.zzz.zzz" to the line, before the end quote.   

**file: ip2r** 

Usage: try the following on the command line:

./ip2r xxx.yyy.zzz

then try

./ip2r xxx.yyy

You will need to get ip2asn-v4.tsv from iptoasn.com - I used gunzip to unzip the file - tar did not seem to work.  This file should be updated regularly.  If you are getting a "Not routed" as the registrant or if you are now getting a registrant that you previously banned, it means this file is likely out of date.

Place that file in the same directory as the scripts. The xxxs are the first 2 or 3 octets of the ip address that you would like to ban that were identified using ad above. Use the first 3 octets of the ip address and if nothing is reported then use the first 2 octets.  As well, if nothing is reported for 2 octets, then walk back the last octet.  eg. try ./ip2r xxx.104 then ./ip2r xxx.103 then ./ip2r xxx.102 etc  

for example xxx.104 might be in the range of xxx.102.0.0 to xxx.105.255.255

If you search on the first 2 octets and get multiple responses you will need to review the list of responses to see where your 3 octets fit in to find the registrant.

for example, if you search xxx.104 and your 3 octets is xxx.104.234 that could correspond to the range of xxx.104.230.0 to xxx.104.235.255 in the list.  Then identify the registrant that corresponds to the range.

The output reading across is:

    start of the ip adress block
    end of the ip address block
    the asn number for the ip address block
    the country
    the registrant

**file: r2ip**

Usage: ./r2ip [-v] [-c country_code] registrant|asn

Country: a 2 letter country code can be specified to narrow the search. Sometimes registrants have similar names.

Country default: all countries if -c is not specified

View: -v is to enable viewing of the output.  Note that the output is not in CIDR format and cannot be used in the pf packet filter.  This takes less processing when the CIDR output is not produced.

View default: creates file with list of ip addresses in CIDR format if -v not specified

Registrant|asn: the registrant's name or an asn number, without a leading 'ASN', can be specified.  For the registrant, it is not case sensitive.  It will capture both XYZ and xyz.  As well, searches are from the start of the registrant's name.  A search for xyz will not catch ab-xyz.

When the view flag is not selected, you will get the ips in CIDR format associated with the registrant or asn.  As well, when the view flag is not selected, the script  will create a file ASN-registrant or ASN-number with the ip addresses in CIDR format.  If the country code is selected then the file name will include the country code in the form of ASN-CC-registrant, where CC is the country code.

You will need perl and p5-Net-CIDR to run this script.

**file: pf.conf** 

You will need to then add those ip addresses to your system's packet filtering.

I created /etc/blocked_ips for this purpose.  I add the ip addresses in CIDR format to this file.  I first add a line to identify the registrant and then I add the ip addresses after that line so I can find the registrant's ip addresses if I wish to remove or replace them later on.

	# regitratant-name
 	www.xxx.yyy.zzz/15
 
I also added the following to pf.conf in the appropriate spots

	table <blocked_ips> persist file "/etc/blocked_ips"

	block drop in quick on $ext_if inet from { <blocked_ips> } to any

	
EDIT Jan 2/25: If you are providing other services, such as named or mail, then you may want to restrict your packet filtering instead, e.g. 
 	
  	block drop in quick on $ext_if inet proto tcp from { <blocked_ips> } to any port {8080, 80, 443}
 
 You will need to restart the packet filter after adding ips to be blocked.

 	eg. service pf restart

This assumes you are running the pf packet filter on your system.
