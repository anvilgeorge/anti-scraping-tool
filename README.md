# anti-scraping-tool

**Please read**

Tools to prevent web scraping

The tools first identify the ip address of the party you would like to ban. ie. an ip address that is making a significant number of requests.  It then provides all of the ip addressess associated with the that party.  Those addresses are then added to the packet filters and blocked.

This is not meant to be a plug and play tool.  You will need to adapt it for your environment.

I generally run them once a day.

my environment: freebsd 13.2, apache 2.4, pf packet filter, perl and sh shell

The first 3 files are bsd command line shell scripts that are run in order:

1. accessday - This will report the number of requests for each ip address, sorted by number of requests.  The first number reported is the number of requests followed by the first 3 octets of the ip addresses associated with the requests.
2. ip2registrant - This will provide the name of the registrant for the ip address that you would like to ban
3. registrant2ips - This will give you the other ips associated with the registrant
4. I then update my packet filtering blocking system with all of the ip addresses associated with the registrant I want to ban. 

For the script files you will need to give them execute permission e.g. chmod 700 accessday

**file: accessday**
State number of days prior eg 

./accessday 1     - for yesterday 
./accessday 0     - for today 

This allows you to go back several days.  

The script will search your http access log and pull all of the access requests for that day and provide the number of accesses for each ip address.  It currently will search the last 500,000 entries of your access log.  You can change this as needed by changing the number in "tail -n 500000"

the xxx.xxx.xxx and yyy.yyy.yyy in the script are the first 3 octets of ip addresses you do not want reported, such as your own or a search crawler you do not want to block.  You can delete and add more as needed, for example by adding "| grep -v zzz.zzz.zzz" to the line, before the back slash

I have a version of this script without the '"." $3' as some scrapers will use multiple ip addresses under the first 2 octets. 

The script assumes your http access file is located at /var/log/httpd-access.log.  You may need to change that based on your set up.

**file: ip2registrant**
usage: try the following on the command line:

./ip2registrant xxx.xxx.xxx

then try

./ip2registrant xxx.xxx

You will need to get ip2asn-v4.tsv from iptoasn.com and place that file in the same directory as the scripts.  The xxxs are the first 2 or 3 octets of the ip address that you would like to ban that were identified using accessday above. Use the first 3 octets of the ip address and if nothing is reported then use the first 2 octets.

This will give you the registrant associated with the ip address you would like to ban. 

**file: registrant2ips**
usage: ./registrant2ips registrant   eg. ./registrant2ips badscraper

This will give you the ips associated with the registrant identfied by ip2registrant above. 
It will also create a file ASbadscraper with their ip addresses

You will need perl and p5-Net-CIDR or its equivalent to run this script

**file: pf.conf**
You will need to then add those ip addresses to your system's packet filtering.

I created /etc/blocked_ips for this purpose.  

I also added the following to pf.conf in the appropriate spots

table <blocked_ips> persist file "/etc/blocked_ips" 

block drop in quick on $ext_if inet from { <blocked_ips> } to any

This assumes you are running the pf packet filter on your system.
