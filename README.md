# LinuxLokaverkefni

1. Configure static ip address
    
    In order to configure a static address in the horrible piece of crap that is ubuntu 18.04 you need to use the netplan utility, by default the directory /etc/netplan is empty and you need to create a new .yaml configuration file
    
    run: 
    ```
    sudo nano /etc/netplan/01-netcfg.yaml
    ```
    add the following to file:
    ```
     network:
        version: 2
        renderer: networkd
        ethernets:
            eth0:   -> the name of your 
                dhcp4: no
                addresses: [x.x.x.x/24] -> /24 is your netmask (dont type content after ->)
                gateway4: x.x.x.x
                nameservers:
                    addresses: [x.x.x.x, x.x.x.x]
    ```
    
    run: 
    
    ```
    sudo netplan apply
    ```
    This will apply the new configuration
    
2. change your hostname and domain name
    run: 
    ```
    sudo hostnamectl set-hostname "your-new-hostname.domain"
    ```
    -> enter this without the quotes
    verify the changes with the command:
    ```
    hostname -f
    ```
    
3. Setup a DHCP server

    install the dhcp server
    
    ```
    sudo apt install isc-dhcp-server
    ```
    
    in order to configure the dhcp server you must open the configuration file with a text editor like nano
    
    ```
    sudo nano /etc/dhcp/dhcpd.conf
    ```
    
    change the following options
    
    ```
    option domain-name "yourdomain.com";
    option domain-name-servers x.x.x.x, x.x.x.x;
    default-lease-time 3600;
    max-lease-time 7200;
    ```
    
    Uncomment the following line
    
    ```
    #authoritative;
    ```
    
    Add the following to the end of the file
    
    ```
    subnet x.x.x.x netmask x.x.x.x{
        option routers x.x.x.x;
        option subnet-mask x.x.x.x;
        range x.x.x.x x.x.x.x;
    }
    ```
    
    Next you must restart the dhcp service
    
    ```
    sudo systemctl restart isc-dhcp-server.service
    ```
    
    Pro tip you can view the dhcp lease list with the following command
    
    ```
    dhcp-lease-list
    ```
    
4. Install and configure DNS
    
    start by installing the dns server packages
    
    ```
    sudo apt install -y bind9 bind9utils bind9-doc dnsutils
    ```
    
    In order to create zones you need to edit the DNS server configuration file
    
    ```
    sudo nano /etc/bind/named.conf.local
    ```
    
    add an entry for a sample domain forward lookup zone
    
    ```
    zone "margeir.local" IN {// <- replace with your domain name
        type master; //This means that this is the Primary DNS server/master DNS server
        file "/etc/bind/fwd.margeir.local.db"; //This is your Forward lookup file
        allow-update { none; }; //This is a primary dns server so updates are not needed
    };
    ```
    
    you can also create a reverse lookup zone
    
    ```
    zone "100.168.129.in-addr.arpa" IN { // this should match your network in reverse order
        type master;
        file "/etc/bind/rev.margeir.local.db" //reverse lookup zone
        allow-update { none; } //this is a primary dns so none is fine
    };
    ```
    
    Next we need to create a zone lookup file, we will start by copying the sample file to /etc/bind
    
    ```
    cp /etc/bind/db.local /etc/bind/fwd.margeir.local.db
    ```
    
    edit the zone lookup file
    
    ```
    sudo nano /etc/bind/fwd.margeir.local.db
    ```
    
    Lets update the contents of said file, when you update these remember to change the serial number to a random intiger
    
    ```
    ;
    ; BIND data file for local loopback interface
    ;
    $TTL    604800
    @       IN      SOA     192.168.100.1. root.margeir.local. (
                                 20         ; Serial
                             604800         ; Refresh
                              86400         ; Retry
                            2419200         ; Expire
                             604800 )       ; Negative Cache TTL
    ;
    ;@      IN      NS      localhost.
    ;@      IN      A       127.0.0.1
    ;@      IN      AAAA    ::1

    ;Name Server Information
           IN      NS      192.168.100.1.
    ;IP address of Name Server
    ns1     IN      A       192.168.100.1.
    
    ;A - Record HostName To Ip Address
    www     IN       A      192.168.100.1
    @       IN       A      192.168.100.1
    ;CNAME record
    ftp     IN      CNAME   www.margeir.local.
    ```
    
    cpoy the sample file for the reverse lookup zones
    
    ```
    cp /etc/bind/db.127 /etc/bind/rev.margeir.local.db
    ```
        
    change the configuration for the reverse lookup zones
    
    ```
    sudo nano /etc/bind/rev.margeir.local.db
    ```
    
    update your file with the content shown below and remember to update your serial number with a random intiger
    
    ```
    ;
    ; BIND reverse data file for local loopback interface
    ;
    $TTL    604800
    @       IN      SOA     margeir.local. root.margeir.local. (
                                 20         ; Serial
                             604800         ; Refresh
                              86400         ; Retry
                            2419200         ; Expire
                             604800 )       ; Negative Cache TTL
    ;
    ;@      IN      NS      localhost.
    ;1.0.0  IN      PTR     localhost.

    ;Name Server Information
           IN      NS     192.168.100.1.
    ;Reverse lookup for Name Server
    10      IN      PTR    192.168.100.1.
    ;PTR Record IP address to HostName
    100     IN      PTR    www.margeir.local.
    200     IN      PTR    margeir.local.
    ```
    
    Now its time to check our configuration, use the following command to check for syntax errors in zone files
    
    ```
    named-checkzone margeir.local /etc/bind/ <- name of zone file goes here // replace margeir.local with your domain
    ```
    check the named.conf file for syntac errors with the following command
    ```
    named-checkconf
    ```
    These commands should have an output similar to this
    
    > zone margeir.local/IN: loaded serial 20
    OK
    
    now you need to restart the dns server
    ```
    systemctl restart bind9
    ```
    enable the dns server on startup
    ```
    systemctl enable bind9
    ```
    check the status of the dns server
    ```
    systemctl status bind9
    ```
    
    in order to verify that the dns server works you can now go to a client machine and add the new dns server ip address to **/etc/resolv.conf**
    
    ```
    sudo nano /etc/resolv.conf
    ```
    
    add and entry like this for your server
    
    ```
    nameserver 192.168.100.1
    ```
    
    now you can use the dig command to check the forward lookup zone
    
    ```
    dig www.margeir.local
    ```
    
    test the reverse lookup
    ```
    dig -x 192.168.100.1
    ```
    
5. Create user accounts
    
    
    
