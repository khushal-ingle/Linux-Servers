## What is DNS (Domain Name Server)?
A DNS server is a specialized server that performs domain name resolution, translating human-readable domain names into IP addresses that computers and devices use to communicate with each other. Without DNS, users would need to remember and input IP addresses to access websites, which would be much more difficult to do.

DNS servers can be configured in different roles:

- **Cache Only DNS Servers**: They store previously resolved domain names for faster lookups.
- **Authoritative DNS Servers**: These provide definitive answers for domains they manage.
- **Recursive DNS Servers**: These servers resolve domain names by querying other DNS servers.

DNS also helps route emails, ensure efficient load balancing across servers, and improve the overall security of your network through mechanisms like DNSSEC (DNS Security Extensions).

---

## Step 1: Install BIND and Related Packages
```sh
yum install bind bind-utils -y
```
  
or
  
```sh
yum install bind*
```

Check the files and information of installed Package
```sh
rpm -qa bind
rpm -qc bind   #check configuration files
```

Check the details of all root in servers
```sh
vim /var/named/named.ca
```

Start and enable the DNS service
```sh
systemctl status named.service
systemctl restart named.service
systemctl enable named.service
```

Change the DNS setting for the network to localhost
```sh
vim /etc/NetworkManager/system-connections/enp0s3.nmconnection
```

```sh
dns=127.0.0.1
```

Try resolving the names through the configured server
```sh
dig google.com 127.0.0.1
```

---

## Step 2: Configure BIND
Now we are creating our Cache Only DNS Server to Master DNS Server.

Create the Backup of your main configuration file
```sh
cp -v /etc/named.conf /etc/named.conf.bak
vim /etc/named.conf
```

Update the named.conf file with the following configuration:
```sh
options {
 listen-on port 53 { 127.0.0.1; 192.168.43.61; };
 listen-on-v6 port 53 { ::1; };
 directory  "/var/named";
 dump-file  "/var/named/data/cache_dump.db";
 statistics-file "/var/named/data/named_stats.txt";
 memstatistics-file "/var/named/data/named_mem_stats.txt";
 secroots-file "/var/named/data/named.secroots";
 recursing-file "/var/named/data/named.recursing";
 allow-query     { localhost; 192.168.43.0/24; };

 recursion yes;

 // if the forwarder are available then will be able to resolve external
 forwarders {8.8.8.8; 8.8.4.4; };

 dnssec-validation yes;

 managed-keys-directory "/var/named/dynamic";
 geoip-directory "/usr/share/GeoIP";

 pid-file "/run/named/named.pid";
 session-keyfile "/run/named/session.key";

 /* https://fedoraproject.org/wiki/Changes/CryptoPolicy */
 include "/etc/crypto-policies/back-ends/bind.config";
};

logging {
        channel default_debug {
                file "data/named.run";
                severity dynamic;
        };
};

zone "." IN {
 type hint;
 file "named.ca";
};

include "/etc/named.rfc1912.zones";
include "/etc/named.root.key";
```

### Key Points
- `listen-on`: Specifies the IP addresses on which the server will listen.
- `allow-query`: Determines which hosts can query the DNS server.
- `forwarders`: Defines DNS servers to which unresolved queries will be forwarded.
- `recursion`: Enables recursive DNS queries.
- `zone "."`: Defines the root zone, which contains the default root DNS server references.

---

## Step 3: Create the Forward Zone File

Set the host-name of the server as per the domain (phoenix.local)
```sh
vim /etc/hostname
```

```sh
ns1.phoenix.local
```

Make the entry of the IP address and domain name
```sh
127.0.0.1   localhost ns1 ns1.phoenix.local
::1         localhost localhost6 ns1 ns1.phoenix.local
192.168.43.51 ns1 ns1.phoenix.local
```

Making an entry of the (phoenix.local) zone file in the `/etc/named.rfc1912.zones`
```sh
zone "phoenix.local" IN {
 type master;
 file "forward.phoenix.local";
 allow-update { none; };
};
```

Create the zone file with the same name provided while making the entry of the zone
```sh
vim /var/named/forward.phoenix.local
```

```sh
$TTL 1D
@       IN SOA  ns1.phoenix.local. root.phoenix.local. (
                                        0001    ; serial
                                        1D      ; refresh
                                        1H      ; retry
                                        1W      ; expire
                                        3H )    ; minimum
@                       IN      NS      ns1.phoenix.local.
ns1                     IN      A       192.168.43.51
phoenix.local.          IN      A       192.168.43.51
www                     IN      CNAME   phoenix.local.
router                  IN      A       192.168.43.1
mainpc                  IN      A       192.168.43.87
debian                  IN      A       192.168.43.41
```

Change the ownership of the file
```sh
chown -Rv named:named /var/named/forward.phoenix.local
systemctl restart named.service
```

---

## Step 4: Create the Reverse Zone File

Make an entry of the (phoenix.local) zone file in the `/etc/named.rfc1912.zones`
```sh
zone "43.168.192.in-addr.arpa" IN {
 type master;
 file "reverse.phoenix.local";
 allow-update { none; };
};
```

Create the zone file
```sh
vim /var/named/reverse.phoenix.local
```

```sh
$TTL 1D
@ IN SOA ns1.phoenix.local. root.phoenix.local. (
     001 ; serial
     1D ; refresh
     1H ; retry
     1W ; expire
     3H ) ; minimum
@   IN  NS ns1.phoenix.local.
@   IN  PTR ns1.phoenix.local.
31   IN  PTR ns1.phoenix.local.
87   IN  PTR mainpc.phoenix.local.
41   IN   PTR debian.phoenix.local.
100.43.168.192.in-addr.arpa. IN  PTR mail.phoenix.local.
```

---

**NOTE**: Whenever the change happens in DNS records in the Master DNS server, the serial number needs to be updated to reflect the change in the Slave Server.

