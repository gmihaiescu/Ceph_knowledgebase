Ceph_knowledgebase based on emails read on the Ceph mailing list and other sources
==================

A repository of the things I learned about Ceph

1.	Journal loss during operation means OSD loss. Total OSD loss, no recovery.

2.	One SSD 200GB DC Intel 3700s can write close to 400MB/s, so having a 1:4 or even 1:5 SSD journal/SATA disk ratio is sensible, considering a SATA disk can write ~ 100 MB/s.

3.	 However these will be the ones limiting your max sequential write speed if that is of importance to you. In nearly all use cases you run out of IOPS (on your HDDs) long before that becomes an issue, though.

4.	One 10 Gb NIC can theoretically bring 1.2 GB/s to the OSDs with the SSD journal being in front of the data flow and they can write 400 MB/s max, it means that three SSD journaling disks in a server would max the network capacity. Three SSD journaling disks would have enough write capacity for 12 SATA disks (at 100 MB/s).  

5.	Raiding the journal SSDs seems wasteful given the cost and quality of the DC 3700s, and RAID 1 would halve the IOPS provided by the SSDs.

a.	Set the "mon osd downout subtree limit" so that a host going down doesn't result in a full re-balancing and the resulting IO storm. In nearly all cases servers are recoverable and they come back after a reboot, and even if that fails, you get to pick the time for the recovery.

b. Set the out interval (mon osd down out interval) so you can react to drive failures, instead of automatic replication to happen.

c.	Configure the various backfill options to have only a small impact when a drive or entire servers fails. The missing replicas will have to be synchronized from other servers/OSD which will compete for IO and network bandwidth with production/client requests.

6.	Journals on a filesystem go against KISS. Not only do you add one more layer of complexity that can fail, you're also wasting CPU cycles that might needed over in the less than optimal OSD code. And you gain nothing from putting journals on a filesystem.

One possible configuration to maximize SSD utilization while minimizing risk if loosing one SSD journal:
“We use 14 disks in each osd-server; 8 platter and 4 ssd for ceph, and 2 ssd for OS/journals. We partitioned the two OS ssd as raid1 using about half the space for the OS and leaving the rest on each for 2x journals and unprovisioned. We've partitioned the OS disks to each hold 2x platter journals. On top of that our ssd pooled disks also hold 2x journals; their own + an additional from a platter disk. We have 8 osd-nodes. So whenever an ssd fail we lose 2 osd (but never more)”.

If you loose an SSD disk that also has a journal, then the SAS/SATA disk using it as journal is considered gone/corrupted and had to be re-instated in the pool with full backfills, once the SSD journal is replaced.
Basically, if you put 5 journals on a SSD disk and you loose it, you’ll have to backfill 5 large OSD SATA disks (many TB of data).

Set the “noout” interval  for the OSDs long enough so the operations team can replace the drive and do only one replication. Otherwise, when the OSD is marked as “down, out” the data that was on the failed OSD will start being replicated to other OSDs in the cluster to maintain the replica count. When you replace the failed disk, the backfilling process will start causing a new replication process.

Configure backfilling options as below: 1 backfill, 1 recovery, and the lowest queue priority (1/63) for recovery IOs.

One possible good option is to have one OSD (400 MB/s write) with four partitions (each 20 GB) and to set the journal for four SATA 1/2/3 TB drives on the four SSD partitions.


The normal way of growing a cluster is to add more OSDs. Preferably of the same size and same performance disks.
This will not only simplify things immensely but also make them a lot more predictable.
This of course depends on your use case and usage patterns, but often when running out of space you're also running out of other resources like CPU, memory or IOPS of the disks involved. So adding more instead of growing them is most likely the way forward.

If you were to replace actual disks with larger ones, take them (the OSDs) out one at a time and re-add it. If you're using ceph-deploy, it will use the disk size as basic weight, if you're doing things manually make sure to specify that size/weight accordingly. Again, you do want to do this for all disks to keep things uniform.

If your cluster (pools really) are set to a replica size of at least 2 (risky!) or 3 (as per Firefly default), taking a single OSD out would of course never bring the cluster down.
However taking an OSD out and/or adding a new one will cause data movement that might impact your cluster's performance.


Useful for monitoring Ceph for performance issues:
https://www.mail-archive.com/ceph-users@lists.ceph.com/msg12709.html

Troubleshooting OSDs:
http://ceph.com/docs/master/rados/troubleshooting/troubleshooting-osd/

Sample ceph.conf (with notes):
https://github.com/ceph/ceph/blob/master/src/sample.ceph.conf


PG calculation:

“When using multiple data pools for storing objects, you need to ensure that you balance the number of placement groups per pool with the number of placement groups per OSD so that you arrive at a reasonable total number of placement groups that provides reasonably low variance per OSD without taxing system resources or making the peering process too slow.”
It’s best to round up that 1500 to 2048 for starters.

With smaller clusters, it is beneficial to overprovision PGs for various reasons (smoother data distribution, etc).

The larger the cluster gets, the closer you will want to adhere to that 100 PGs per OSD, as the resource usage (memory, CPU, network peering traffic) creeps up.

So, for example with a cluster using 50 servers with 20 OSDs each 2 TB (total 1000 OSDs), three replicas and three large pools (Cinder , Glance, S3):

Total PGs for the entire cluster: (1000 OSD * 100 PGs) / 3 (repl) = 100000/3 = 33333 => but because the next ^2 is 65536, this is the recommended number.


Dashboard to monitor Ceph:
https://github.com/Crapworks/ceph-dash

=============================================
Instructions to install Ceph Dashboard:
=============================================

From a Ceph node that has Cephx and ceph.conf correctly set:

git https://github.com/Crapworks/ceph-dash 
cd ceph-dash && python ceph-dash.py

Then go to: http://ceph_node

To run the Ceph Dashboard behind Apache:

apt-get install apache2 libssl libapache2-mod-wsgi

a2enmod libssl mod_wsgi
Create the SSL cert and key:
openssl req -x509 -nodes -days 1095 -newkey rsa:2048 -out /etc/apache2/ssl/ssl.crt -keyout /etc/apache2/ssl/ssl.key

cp ceph-dash/contrib/apache/cephdash /etc/apache2/sites-enabled/
cp -ar ~/ceph-dash /opt/
vi /etc/apache2/sites-enabled/cephdash  => Add "Require all granted" and comment out the "Allow" directives

root@ceph1:~/ceph-cluster# cat /etc/apache2/sites-enabled/cephdash.conf 
<VirtualHost *:80>
    ServerName foo.bar.de

    RewriteEngine On
    RewriteCond %{REQUEST_URI} !^/server-status
    RewriteRule ^/?(.*) https://%{HTTP_HOST}/$1 [R,L]
</VirtualHost>

<VirtualHost *:443>
    ServerName ceph1.local

    WSGIDaemonProcess cephdash user=www-data group=www-data processes=1 threads=5
    WSGIScriptAlias /ceph /opt/ceph-dash/contrib/wsgi/cephdash.wsgi
    WSGIPassAuthorization On

    SSLEngine on
    SSLCertificateFile /etc/apache2/ssl/ssl.crt
    SSLCertificateKeyFile /etc/apache2/ssl/ssl.key

    <Directory /opt/ceph-dash/contrib/wsgi>
        WSGIProcessGroup cephdash
        WSGIApplicationGroup %{GLOBAL}
#        Order deny,allow
Require all granted
#        Order allow,deny
#        Allow from all
    </Directory>
</VirtualHost>


Restart the apache server.

Visit: https://ceph

--------------------------------------------------------------------------------------------------------

Good deployment instructions:
http://switzernet.com/3/public/130925-ceph-cluster/


Multi-chassis link aggregation:
http://www.networkcomputing.com/networking/when-mlag-is-good-enough/a/d-id/1232541?

http://www.fragmentationneeded.net/2011/11/nexus-5k-layout-for-10gbs-servers-part.html

Dell whitepaper on building a HPC cluster:
http://i.dell.com/sites/doccontent/corporate/case-studies/en/Documents/2012-tgen-10011143.pdf







