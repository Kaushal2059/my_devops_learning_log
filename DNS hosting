🖥️ Local DNS Server Setup using BIND9 on Ubuntu
This repository documents my hands-on practice of configuring a local DNS server using BIND9 on an Ubuntu Server environment.
The goal of this project is to understand how DNS works internally by manually configuring forward and reverse lookup zones and testing name resolution locally.

📌 Project Overview
    OS: Ubuntu Server
    DNS Software: BIND9
    Environment: Local Virtual Machine
    Testing Tools: dig, nslookup, ping
    Configuration Type:
    Forward Lookup Zone
    Reverse Lookup Zone
    Custom Domain Mapping

  1: Install the DNS server DNS: 
      as: sudo apt install bind9 bind9-utils bind9-doc -y

  2: check status of the service and start and enable the service if not enables:
      as: sudo systemctl status bind9.service (To check the status of the server)
        sudo systemctl enable bind9.service (To enable the service)
        sudo systemctl start bind9.service (To start the service)

  3: Creating own Domain zone in the location /etc/bind/named.conf:  
      as: zone "kaushal.local" {
        type master;
        file "/etc/bind/db.kaushal.local";
        };

        DNS works using something called zones.
        A zone is a domain you control.
        Here we are telling BIND:
        "I am the master DNS server for kaushal.local, and its records are stored in this file."
        So now the server becomes the authority for kaushal.local.

  4: The next step is to create a zone file :
    Inside /etc/bind, vim db.kaushal.local
              $TTL    604800
              @       IN      SOA     kaushal.local. root.kaushal.local. (
                                            2         ; Serial
                                       604800         ; Refresh
                                        86400         ; Retry
                                      2419200         ; Expire
                                       604800 )       ; Negative Cache TTL
              ;
              @       IN      NS      kaushal.local.
              srv1    IN      A       192.168.221.128
              www     IN      A       192.168.221.128
              @       IN      A       192.168.221.128
              where,
              TTL means Time to live. Generally this means how long the DNS server will keep the cache.Here 604800(seconds) mean 7 days
              i.e if another DNS server resolves your domain, it can cache it for 7 days before asking again.
              SOA mean State of Authority i,e the server is the official authority of the domain. Everyzone must have exactly one SOA record where,
              @ ---> current zone name i.e kaushal.local
              IN ---> Internet class which means this record belong to the internet.
              kaushal.local. is the primary name server for the domai. The dot(.) at the end mean it is the fully qualified domain name.
              root.kaushal.local is the email adderess of the DNS administrator. This is read as root@kaushal.com the first dot is replaced by @ by thr DNS


















         
