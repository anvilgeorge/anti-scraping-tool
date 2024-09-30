# anti-scraping-tool
Tools to prevent scraping
I created the following tools to prevent web scraping.

environment: freebsd 13.2, apache24, pf, perl and sh shell

The first 3 are files run in the sh shell
file: accessday
must state number of days prior eg ./accessday 1 - for yesterday ./accessday 0 for today

datetag=date -v-$1d "+%d/%b/%Y" tail -n 500000 /var/log/httpd-access.log | grep $datetag
| grep -v xxx.xxx.xxx | grep -v yyy.yyy.yyy
| awk '{print $1}' | awk -F'.' '{print $1 "." $2 "." $3}' | sed s/' '/\n/g | sort | uniq -c | sort -n

the xxx.xxx.xxx and yyy.yyy.yyy are the first 3 octets of ip addresses you do not want reported, such as your own or a search crawler.

This will report the number of requests for each ip address, sorted by number of requests I have a version of this without the '"." $3' as some scrapers will use multiple ip addresses under the first 2 octets. I also have a version with '"." $4' added. You will have to see what works for you.

if your access log is located somewhere else then you will need to change that.
file: ip2name
usage: try ./ip2name xxx.xxx.xxx then try ./ip2name xxx.xxx

grep ^$1 ip2asn-v4.tsv

get ip2asn-v4.tsv from iptoasn.com xxxs are the first 2 or 3 octets of the ip address from the abusers from accessday above. Use the first 3 octets and if nothing is reported then use the first 2 octets.

This will give you the registrant associated with the ip address
file: name2ip
usage: ./name2ip registrant eg. ./name2ip Yandex

grep -i $1 ip2asn-v4.tsv | awk '{print $5}' | grep -f - ip2asn-v4.tsv | awk '{print $1 " - " $2}' | sort -n | perl -MNet::CIDR=range2cidr -lne 'print for range2cidr $_' > ASN$1 cat ASN$1

This will give you the ips associated with the registrant. You will need perl and p5-Net-CIDR
file: pf.conf

you will need to then add those ips to a file you create /etc/blocked_ips and add the following to pf.conf in the appropriate spots

table <blocked_ips> persist file "/etc/blocked_ips" 

block drop in quick on $ext_if inet from { <blocked_ips> } to any

I know there is a lot here but it works well. I am sure others can suggest improvements.
