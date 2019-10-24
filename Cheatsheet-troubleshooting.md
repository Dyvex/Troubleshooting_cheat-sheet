# Trouble shoot checklist
### Bottom-up werking essentieel! 

## 1. Physical layer and network access layer
## Netwerkinterface
- Kabel correct aangesloten? -> kijk naar LEDs
- Kabels testen
- VirtualBox: "Cable connected"?
- terminal: `ip link` -> zien of interfaces 'up' zijn
## 2. Internetlayer - BINNEN
### EERST op machine zelf kijken
- IP adres en network mask
- default gateway
- DNS server
### Dan pas "naar buiten" kijken
- Gateway bereikbaar?
- DNS-server beschikbaar?
- Andere hosts op het LAN
IP-adres: `ip a` commando
- VBox NAT: **10.0.2.15** -> Vbox NAT is altijd dit
- Host-only: **192.168.65.101-254**
- `/etc/sysconfig/network-scripts/ifcfg-IFACE` 
    (*IFACE = interface (bv.eth0*)). 
    -> Hier controleer je die instellingen vd interface
    > **eth0/enp0s3** = heeft DHCP protocol(dynamisch)
    > **eth1/enp0s8** = statische IP(manueel geconfigureerd) OOK SUBMASK   INVULLEN
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
    > `sudo systemctl status SERVICE.service` (bv. httpd.service) 
            -> toont aan of service draait,poort 80 en 434(tls) moeten openstaan!
    > `sudo systemctl start/enable/kill/disable/kill SERVICE.service`
            -> service operaties uitvoeren(disable/enable -> al dan niet                                 opstarten bij de boot)
    > `sudo systemctl list-units --type service`
            -> Lijst van alle services
    > `sudo systemctl --failed`
            -> geeft lijst van alles gefaalde services bij de boot
- **firewall-cmd**
    > `sudo firewall-cmd --list-all` -> geeft lijstje van de actieve                                          firewall instellingen
    > `sudo firewall-cmd --add-service=SERVICE --permanent`
            -> service permanent toevoegen
    > `sudo firewall-cmd --add-port=PORTNUMMER/tcp --permanent`
            -> port permanent toevoegen
    > zelfde voor een port/service te verwijderen maar dan                  *--remove-service* of *--remove-port*
    > `sudo firewall-cmd --get-services`
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
        > -> `sudo apachectl configtest` -> httpd
        > **ALTIJD HERSTARTEN SERVICE na test of aanpassingen!**
        > -> `sudo systemctl restart httpd.service`
        > Na wijziging van een service altijd **herstarten van de service**!!
        > -> `sudo systemctl restart SERVICE`
    * fileserver: testparm
    * DNS (BIND): named-checkconf, named-checkzone

 ### 4a. BIND
- BIND noemt `named.service` door systemMD

## TIPS

**READ THE ERROR MESSAGES!!!**
- Op welk nieveau van TCP/IP zit het probleem?
    > Internet? transport? Applicatie?
      Foutboodschap = shortcut voor startpunt troubleshoot
      **LET OP:** Foutboodschap komt in logs, niet op console

### Voorbeelenden foutboodschap
- `No route to host`
    > Internetlaag
    > IP configuratie
    > netwerklaag

- `Connection refused`
    > transportlaag
    > tcp protocol
    > kijk niet naar ip config of kabels, het zit in transport
    > service draait niet

- `Unable to resolve host`
    > Internet/applicatie laag
    > DNS server niet beschikbaar

- `Error 404: ... Not found`
    > applicatielaag
    > appache/webserver
    > URL verkeerd


**Belangrijke LOG-Directories en anderen**
- Ken de locaties van logs:
    > `/var/log/messages` (hoofdlog)
    > `/var/log/audit/audit.log` (SELinux)
    > `/var/log/httpd/error_log`
    > `/var/log/samba/*`
    > `/var/log/vsftpd/*`
    > `/etc/httpd/httpd.conf`
