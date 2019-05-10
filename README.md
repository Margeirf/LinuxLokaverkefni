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
            ens5:
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
    #authorative;
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

    
