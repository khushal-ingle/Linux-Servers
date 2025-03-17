# DHCP Configuration Guide

## What is DHCP?

Dynamic Host Configuration Protocol (DHCP) is a network management protocol used to assign IP addresses and other network configurations to devices on a network. A DHCP server allocates IP addresses to clients.

In CentOS 9, the DHCP server can be easily configured to distribute IP addresses within a specified range, along with other important network configuration details like DNS servers, gateways, and lease times.

## Working of Dynamic Host Configuration Protocol

DHCP automatically assigns IP addresses and network configurations to devices on a network. Here’s a simplified breakdown of how it works:

1. **DHCP Discover (Client Request):**  
   When a device (e.g., a computer or phone) connects to the network, it sends a DHCP Discover message to find a DHCP server. This message is broadcast because the client doesn’t have an IP address yet.

2. **DHCP Offer (Server Response):**  
   The DHCP server responds with a DHCP Offer that includes an available IP address, subnet mask, gateway, and DNS server settings.

3. **DHCP Request (Client Confirmation):**  
   The client receives the offer and sends a DHCP Request message back to the server, accepting the offered IP address.

4. **DHCP Acknowledgment (Server Confirmation):**  
   The DHCP server sends a DHCP Acknowledgment (ACK) to confirm that the IP address has been assigned to the client.

## Prerequisites

Before configuring the DHCP server, ensure you meet the following prerequisites:

- A static IP address to maintain a stable connection.
- Firewall configured to allow DHCP traffic.
- SELinux policies adjusted, if necessary.

## Step 1: Install DHCP Server Package

The first step is to install the DHCP server package on CentOS 9. Open a terminal and execute the following commands:

```sh
sudo dnf install dhcp-server -y
sudo yum install dhcp-server -y
```

Once the installation is complete, verify it by checking the version of DHCP:

```sh
dhcpd -v
rpm -qacl dhcp-server
```

## Step 2: Configure the DHCP Server

The main configuration file for the DHCP server is located at `/etc/dhcp/dhcpd.conf`. This file holds the settings for the DHCP service, such as IP address ranges, subnet masks, and lease times.

Open the configuration file using:

```sh
vim /etc/dhcp/dhcpd.conf
```

Set the default lease time and max lease time:

```sh
default-lease-time 600;
max-lease-time 7200;
```

Define the subnet and the range of IP addresses that the DHCP server will assign to the clients:

```sh
authoritative;
subnet 192.168.143.0 netmask 255.255.255.0 {
    range 192.168.143.100 192.168.143.200;
    option routers 192.168.143.1;
    option domain-name-servers 8.8.8.8, 8.8.4.4;
    option broadcast-address 192.168.143.255;
    default-lease-time 600;
    max-lease-time 7200;
}
```

Restart the DHCP server:

```sh
systemctl restart dhcpd.service
```

Enable the service to start on boot:

```sh
systemctl enable dhcpd.service
```

Check if the service is enabled by verifying the port:

```sh
netstat -nltup | grep "dhcp"
```

## Step 3: Configure Firewall for DHCP

CentOS 9 has a firewall enabled by default. Allow the DHCP service through the firewall:

```sh
systemctl status firewalld.service
firewall-cmd --zone=public --add-service=dhcp --permanent
firewall-cmd --zone=public --add-port=67/udp --permanent
```

## IP Reservation

To assign a specific IP to a host based on its MAC address:

```sh
vim /etc/dhcp/dhcpd.conf
```

Add the following configuration:

```sh
host pc-one {
    hardware ethernet 08:00:27:13:06:ec;
    fixed-address 192.168.143.11;
}
```

Restart the DHCP service:

```sh
systemctl restart dhcpd.service
```

## Deny List (Blacklisting Clients)

To deny specific clients from obtaining an IP address:

```sh
vim /etc/dhcp/dhcpd.conf
```

Add the following configuration:

```sh
host pc-one {
    hardware ethernet 08:00:27:13:06:ec;
    deny booting;
}
```

Restart the DHCP service:

```sh
systemctl restart dhcpd.service
```

## Allow List (Whitelisting Clients)

To allow only specific clients to obtain an IP while blocking others:

```sh
vim /etc/dhcp/dhcpd.conf
```

Add the following configuration:

```sh
subnet 192.168.143.0 netmask 255.255.255.0 {
    range 192.168.143.100 192.168.143.200;
    option routers 192.168.143.1;
    option domain-name-servers 8.8.8.8, 8.8.4.4;
    option broadcast-address 192.168.143.255;
    default-lease-time 600;
    max-lease-time 7200;
    deny unknown-clients;
}

host pc-one {
    hardware ethernet 08:00:27:6c:22:80;
}
```

Restart the DHCP service:

```sh
systemctl restart dhcpd.service
```

## DHCP Pools (Client Segmentation)

To create pools that allow or deny specific clients:

```sh
vim /etc/dhcp/dhcpd.conf
```

Add the following configuration:

```sh
subnet 192.168.143.0 netmask 255.255.255.0 {
    range 192.168.143.100 192.168.143.200;
    option routers 192.168.143.1;
    option domain-name-servers 8.8.8.8, 8.8.4.4;
    option broadcast-address 192.168.143.255;
    default-lease-time 600;
    max-lease-time 7200;

    pool {
        range 192.168.143.100 192.168.143.150;
        deny unknown-clients;
    }
    
    pool {
        range 192.168.143.200 192.168.143.250;
        allow unknown-clients;
    }
}

host pc-one {
    hardware ethernet 08:00:27:6c:22:80;
}
```

Restart the DHCP service:

```sh
systemctl restart dhcpd.service
```

This completes the DHCP server configuration on CentOS 9.

