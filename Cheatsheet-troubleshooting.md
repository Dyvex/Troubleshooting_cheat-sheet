# Cheat sheet Troubleshoot-test
### Bottom-up werking essentieel! 
## 1. Physical layer and network access layer
## Netwerkinterface
- Kabel correct aangesloten? -> kijk naar LEDs
- Kabels testen
- VirtualBox: "Cable connected"?
- terminal: `ip link` -> zien of interfaces 'up' zijn
## 2. Internetlayer - BINNEN
 **EERST op machine zelf kijken**
- IP adres en network mask
- default gateway
- DNS server
**Dan pas "naar buiten" kijken**
- Gateway bereikbaar?
- DNS-server beschikbaar?
- Andere hosts op het LAN
IP-adres: `ip a` commando
- VBox NAT: **10.0.2.15** -> Vbox NAT is altijd dit
- Host-only: **192.168.65.101-254**
- `/etc/sysconfig/network-scripts/ifcfg-IFACE` 
    (*IFACE = interface (bv.eth0*)). 
    -> *Hier controleer je die instellingen vd interface*
    - **eth0/enp0s3** = heeft DHCP protocol(dynamisch)
    - **eth1/enp0s8** = statische IP(manueel geconfigureerd) OOK SUBMASK   INVULLEN
    -> **Hoe config file aanpassen?**
        > bv. `vi /etc/sysconfig/network-scripts/ifcfg-eth0`
Default gateway: `ip route`  commando 
- VBox NAT: 10.0.2.2 altijd
- thuisnetwerken vaak: 192.168.0.1 of 192.168.1.1
### DNS Server: 
- VBox Nat altijd: 10.0.2.3
- thuisnetwerken vaak: 192.168.0.1 of 192.168.1.1

## Internetlayer - BUITEN
- **Default gateway:** `ping xx.xx.xx.xx`
- **DNS Configuratie:** `cat /etc/resolv.conf`
- **DNS request:**
    - `dig / nslookup www.domainname.com `  == Kijken of de DNS service         draait
        > `dig www.domainname.com +short `
- Traceroute naar buiten
    - traceroute, tracert, tcpraceroute, tracepath
**OPGELET:** ping/tracert werkt niet altijd. OOK NIET OP HOGENT NETWERK
        
            
## 3. Transportlaag
Zijn de juiste poorten open/draait de service
**Belangrijke commando's**
- **Systemctl** 
    - **`sudo systemctl status SERVICE.service`** (bv. httpd.service)
            -> toont aan of service draait,poort 80 en 434(tls) moeten openstaan!
    - `sudo systemctl start/enable/kill/disable/kill SERVICE.service`**
            -> service operaties uitvoeren(disable/enable -> al dan niet                                 opstarten bij de boot)
    - `sudo systemctl list-units --type service`
            -> Lijst van alle services
    - `sudo systemctl --failed`\
            -> geeft lijst van alles gefaalde services bij de boot
- **firewall-cmd**
    - `sudo firewall-cmd --list-all` -> geeft lijstje van de actieve                                          firewall instellingen
    - `sudo firewall-cmd --add-service=SERVICE --permanent`
            -> service permanent toevoegen
    - `sudo firewall-cmd --add-port=PORTNUMMER/tcp --permanent`
            -> port permanent toevoegen
    - zelfde voor een port/service te verwijderen maar dan                  *--remove-service* of *--remove-port*
    - `sudo firewall-cmd --get-services`
- `sudo ss -tulpn` -> kijkt of services draaien
    * t = tcp
    * u = udp
    * l = listening
    * p = processen achter poorten
    * n = poortnummers worden niet omgezet naar naam
- `sudo ps -ef` -> alle processen, kijk hier of service draait

- `sudo iptables -L -n -v` -> lijst van firewall instellingen

### Transportlaag-Buiten: portscanner
vb. nmap -A -T4, nmap, -sS -sU
> Voorbeeld scannen port 80 en 443: `sudo nmap -sS -p 80,443 HOST` 

## 4.Applicatielaag
- Webbrowser:
    > `wget http://HOST/, wget https://HOST/` -> HOST = IPv4
    > `curl http://HOST/, curl https://HOST/` -> HOST = IPV4
- fileserver: smbclient, nmblookup, net use
**Bekijk logfiles!!!!!!!**
- `sudo journalctl -f -u SERVICE.service`
    * l = lijnen tot de breedte van u venster
    * f = logfile open laten, nieuwe events worden dus ook gelogt, je kan       hier wel geen commands meer uitvoeren.(2terminals nodig!)
    * u = de naam van de service meegeven, hier httpd.service
    > *tip: open apart terminal venster met `journalctl -f` !!!*

- **Valideren van Syntax in een config file:**
    * webbrowser: 
        - -> **`sudo apachectl configtest`** -> httpd
            > **ALTIJD HERSTARTEN SERVICE na test of aanpassingen!**
        - -> **`sudo systemctl restart httpd.service`**
            > Na wijziging van een service altijd **herstarten van de service**!!
        - -> **`sudo systemctl restart SERVICE`**
    * fileserver: testparm
    * DNS (BIND): named-checkconf, named-checkzone

    ### NGINX
    ->*Commando's hetzelfde als 'httpd' maar dan met 'nginx' als naam,enkele uitzonderingen:*
    - **Operaties op nginx**
        * `systemctl quit nginx`
        * `systemctl reload nginx`
    - **Test nginx configuration**
        * `systemctl config nginx`
        * `nginx -t`
    - **check nginx versie**
        * `systemctl -v nginx`
    - **Commando help**
        * `systemctl -? nginx`
    **READ THE ERROR MESSAGES!!!**
        - Op welk nieveau van TCP/IP zit het probleem?
        > Internet? transport? Applicatie?
        Foutboodschap = shortcut voor startpunt troubleshoot
        **LET OP:** Foutboodschap komt in logs, niet op console
        
    ### Bind
    - **Installeren van nslookup,dig en host**
        * **`sudo yum install bind-utils`**
    - **bekijken van log files**
        * **`sudo journalctl -f -u named.service`**
            > Indien de query logging nog niet aanstaat kan je **`rndc querylog on`** gebruiken
    - **Configuration files checken**
        * **Main configuration file**: `do named-checkconf /etc/named.conf`
            * Deze zou er dus zo moeten uitzien:
                   
                    
                    options {
                    listen-on port 53 { any; };
                    listen-on-v6 port 53 { any; };
                    directory   "/var/named";

                    allow-query     { any; };   
                    allow-transfer  { any; };

                    recursion no;       ( recursion should be no on authoritative name server )

                    rrset-order { order random; };
                    };
        
        * **zone files**: `sudo named-checkzone ZONE ZONE_FILE`
            - Een concreet voorbeeld: 
                - `named-checkconf`
                - `named-checkzone example.com /var/named/example.conf`
                - `named-checkzone 16.172.in-addr.arpa /var/named/16.172.in-addr.arpa`
    - * **Hoe de forward lookup zones er moeten uitzien (bijvoorbeeld bij example.com)**
            
            
              zone "example.com" IN {
                type master;
                file "example.com";
                notify yes;
                allow-update { none; };
              };
               
     - * **Hoe de forward lookup zones er moeten uitzien (bijvoorbeeld bij example.com)**
            //...
            
                zone "56.168.192.in-addr.arpa" IN {
                    type master;
                    file "56.168.192.in-addr.arpa";
                    notify yes;
                    allow-update { none; };
                };
                          
    - * **Testen van de availability**
        * **gebruik van 'dig'**: `dig @DNS_SERVER_IP HOSTNAME`
            * *dig www.hogent.be @193.190.172.1 +short*  --> **Forward lookup**
            * *dig -x 178.62.144.92* --> **Reverse lookup**
            * *dig NS hogent.be* --> **authoritive name server of hogent.be**
            * *dig SOA hogent.be* --> **Start-of-Authority section of hogent.be**
            * *dig www.hogent.be*
            * *dig +short www.hogent.be*
            * *dig +short @8.8.8.8 www.hogent.be*
            * *dig +short -x 195.130.131.1* -> reverse lookup
        * **gebruik van 'nslookup'**: `nslookup HOSTNAME DNS_SERVER_IP`
            * *nslookup www.hogent.be*
            * *nslookup www.hogent.be 8.8.8.8*
    - **Soorten ERRORS en hoe oplossen**
        * **Running queries locally works, but not remotely**
            > Kijken naar de main-config file in **/etc/named.conf**,deze kunnen vaak nog op default staan (dus localhost)
            Door **listen-on** en **allow-query** te wijzigen naar **any** ipv **localhost** zal elke host kunnen query'en naar deze                 host.
        * **End fully qualified domain names with a dot**
            - Uitvoer  zou dus zo moeten zijn:
               
               
                    www.hogent.be.	    2796	    IN	    A	    178.62.144.90 
            -> Je kan dit testen met `$ORIGIN www.hogent.be.`
        * **Syntax errors in de config file**
            1.  Voer dan eerst `named-checkconf` uit om te kijken **VOOR HET STARTEN OF HERSTARTEN VAN DE NAMED SERVICE**
            2.  Als het probleem er nog steeds is,kijk dan naar de logs in `/var/named/data/named.run`
            3.  Je kan altijd ook eens kijken naar het geregistreerd domein of dit nog oke is met `whois www.hogent.be`
        * **The Serial**
            1. Zone files can be modified on the primary name servers. Once resource records have been added, modified, or removed, you                 must remember to increment the zone serial number. Here is the existing serial number of the example.com zone.
            2. If the initial serial number begins at 0, then the next value will be 1.
            3. Kijken naar de reverse lookup  van het adres met **nano /var/named/2.0.192.in-addr.arpa** (als ip 192.0.2.0 is)
                * Uitvoer zou dan moeten zijn:
                    -  in de 4de kolom zou er bijvoorbeeld geen ns1.example.com mogen staan! maar enkel de alias of ip
                    -   net zoals er ALTIJD een '.' acher de '.com' staat!!
                    
                            $ORIGIN example.com.
                            $TTL 1W

                            @ IN SOA ns1.example.com. hostmaster.example.com. (
                            18042020 
                            1D
                            1H
                            1W
                            1D 
                            )

                                                 IN  NS     ns1
                                                 IN  NS     ns1

                            ns1                  IN  A      192.168.56.10
                            ns2                  IN  A      192.168.56.11
                            dc                   IN  A      192.168.56.40
                            web                  IN  A      192.168.56.172
                            www                  IN  CNAME  web
                            db                   IN  A      192.168.56.173

                            priv0001             IN  A      172.16.0.10
                            priv0002             IN  A      172.16.0.11
                         
                            
            4. Once the zone serial number has been incremented, the zone needs to be reloaded. This can be done **without restarting                   the named process.** Met `rdnc reload example.com`
### Voorbeelenden foutboodschap
- `No route to host`
    - Internetlaag
    - IP configuratie
    - netwerklaag

- `Connection refused`
    - transportlaag
    - tcp protocol
    - kijk niet naar ip config of kabels, het zit in transport
    - service draait niet

- `Unable to resolve host`
    - Internet/applicatie laag
    - DNS server niet beschikbaar

- `Error 404: ... Not found`
    - applicatielaag
    - appache/webserver
    - URL verkeerd
                        

**Belangrijke LOG-Directories en anderen**
- Ken de locaties van logs:
    * `/var/log/messages` (hoofdlog)
    * `/var/log/audit/audit.log` (SELinux)
    * `/var/log/httpd/error_log`
    * `/var/log/samba/*`
    * `/var/log/vsftpd/*`
    * `/var/log/named.conf`
    * `/etc/httpd/httpd.conf`
