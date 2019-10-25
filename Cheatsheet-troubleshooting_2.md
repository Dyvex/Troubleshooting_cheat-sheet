# Trouble shoot demo
Ter info: In dit labo hanteert men een "Bottom-up aproach" om de netwerk services op een Linux syteem te testen. 
Open eerst en vooral `journal ctl -l -f` in een tweede venster om de logs te bekijken!

`sudo journalctl -l -f -u httpd.service`
>l = lijnen tot de breedte van u venster
>f = logfile open laten, nieuwe events worden dus ook gelogt, je kan hier wel geen commands meer uitvoeren, vandaar 2 terminal vensters
>u = de naam van de service meegeven, hier httpd.service

TCP/IP protocol stack

| Layer          | Protocols                | Keywords              |
| :---           | :---                     | :---                  |
| Application    | HTTP, DNS, SMB, FTP, ... |                       |
| Transport      | TCP, UDP                 | sockets, port numbers |
| Internet       | IP, ICMP                 | routing, IP address   |
| Network access | Ethernet                 | switch, MAC address   |
| Physical       |                          | cables                |

## Fase 1: Fysieke laag

**Fysiek**
- Bekijk de LED's van de kabels.
- Test of de kabels werken

**Virual Box**
- Check of "Cable connected" is. In Virtual Box heb je de mogelijkheid om kabels uit te trekken.
- Ga naar -> VM settings -> Network -> select the active interfaces, click “Advanced” and make sure the checkbox “Cable connected” is checked.

**Op de machine**
- Gebruik commando `ip link`
    - `UP`: Interface is connected
    - `NO-CARRIER`: Er is geen signaal op de interface. Je moet je kabels of Virtual box instellingen bekijken.

    ```
    $ ip a
    [...]
    3: eth1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 08:00:27:be:8b:a6 brd ff:ff:ff:ff:ff:ff
    inet 192.168.56.72/24 brd 192.168.56.255 scope global noprefixroute eth1
    [...]
    ```

## Fase 2: Netwerk/Internet laag
Elke host moet *drie settings* juist geconfigureerd hebben:

1. Netwerk interface moet een IP-Adres hebben
2. Default gateway moet ingesteld zijn
3. DNS server moet ingesteld zijn

### IP adres

**Configuratie**
Het IP adres kan automatisch ingesteld worden met DHCP of manueel. Kijk dit na door `cat /etc/sysconfig/network-scripts/ifcfg-INTERFACENAAM` of
`cat /etc/sysconfig/network-scripts/ifcfg-*` voor alle active interfaces te zien. 

**DHCP**
Dingen om te checken voor DHCP:
- DEVICE: Dit is de interface naam
- ONBOOT: moet *yes* zijn, anders zal de host de interface niet activeren bij start-up, of wanneer de netwerk interfaces herstart worden door `systemctl restart network`
- BOOTPROTO: moet op DHCP staan
```
DEVICE="enp0s3"
ONBOOT="yes"
BOOTPROTO="dhcp"
```
Indien dit niet zo is kan je de instellingen aanpassen door het te openen met VI(M). Als de aanpassing gedaan is gebruik commando `systemctl restart network`.

Mogelijke fouten in DHCP:
- Geen IP
    - DCHP server is niet berijkbaar
    - DCHP kan geen IP geven aan host
- IP zoals 169.254.x.x
    - DCHP could not be reached en de host krijgt "link-local" address
- IP zit niet in juiste range
    - Waarschijnlijk een fixed IP ingesteld en de config naar DCHP vergeten veranderen.

**Statisch IP**
Dingen om te checken voor het statische IP:
- DEVICE & ONBOOT: zie hierboven
- BOOTPROTO: moet op *none* staan.
- IPADDR & NETMASK: moeten ingesteld zijn met het juiste IP-adres
```
DEVICE=enp0s8
ONBOOT=yes
BOOTPROTO=none
IPADDR=192.168.56.24
NETMASK=255.255.255.0
```
Indien dit niet zo is kan je de instellingen aanpassen door het te openen met VI(M). Als de aanpassing gedaan is gebruik commando `systemctl restart network`.

Je kan de IP adressen bekijken voor alle interfaces met `ip a`
```
[...]
3: eth1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 08:00:27:be:8b:a6 brd ff:ff:ff:ff:ff:ff
    inet 192.168.56.72/24 brd 192.168.56.255 scope global noprefixroute eth1
[...]
``` 

- Meeste home routers hebben IP-range 192.168.0.0/24, 192.168.1.0/24
- Virtual Box NAT interface is meestal: 10.0.2.15/24
- Virtual Box Host-only interface hangt af van de configuratie. Default range is: 192.168.56.0/24, met DHCP begint dit vanaf 192.168.56.101

Mogelijke problemen met fixed IP
- IP zit in verkeerde range
    - Check config file. Kijk op typfouten
- Correct IP maar "Network unreachable"
    - Bekijk network mask. Dit moet identiek zijn voor alle hosts op de LAN

**Default gateway**
Elke host in zijn LAN moet zijn router of default gateway kennen.

Gebruik `ip r`, kijk voor de lijn `default via x.y.z.w.` 
```
$ ip r
default via 10.0.2.2 dev eth0 proto dhcp metric 100
10.0.2.0/24 dev eth0 proto kernel scope link src 10.0.2.15 metric 100
192.168.56.0/24 dev eth1 proto kernel scope link src 192.168.56.72 metric 101
```

Mogelijke problemen met Default gateway (Automatisch IP met DCHP)
- Geen default gateway ingesteld
    - Waarschijnlijk is er ook een probleem met IP adres op de host. Dit moet eerst opgelost worden
    - Misschien is DHCP verkeerd geconfigureerd

- Onverwacht default gateway adres
    - Je hebt vroeger manueel een gateway adres ingesteld en bent het vergeten veranderen naar DHCP

**DNS Server**
> In order to be able to resolve host names to IP addresses, every host should be able to contact a DNS server.
Gebruik `cat /etc/resolv.conf`, toont (meestal) aan dat het automatisch gegenereerd is. Het heeft één of meerdere lijnen die starten met `nameserver`

```
$ cat /etc/resolv.conf
# Generated by NetworkManager
search hogent.be
nameserver 193.190.173.1
nameserver 193.190.173.2
```
De voorkomende problemen bij DNS zijn dezelfde als die van DHCP!

### Check LAN connectivity
Als alle bovenstaande instelling correct zijn, check of de hosts op het LAN kunnnen bereikt worden.

- Ping default gateway
- Ping andere hosts op de LAN
**Opgelet**: Ping kan geblokkeerd worden op het netwerk. 

### Check DNS Name resolution
DNS configuratie in `/etc/resolv.conf` wil niet zeggen dat de service available is. Gebruik `nslookup` of `dig` 
```
$ dig www.google.com +short
216.58.213.196
$ nslookup www.google.com
Server:		193.190.173.1
Address:	193.190.173.1#53

Non-authoritative answer:
Name:	www.google.com
Address: 216.58.213.228
Name:	www.google.com
Address: 2a00:1450:4013:c04::67
```

## Fase 3: Transport laag
Check of de network service effectief aan het runnen is, welke poort het gebruikt en of de firewall traffic door laat. In dit voorbeeld wordt `httpd` service gebruikt.

### Service & ports
`systemctl list-unit-files | grep service` toont alle services

- Is the service running? `sudo systemctl status httpd.service`
    - Verwachte output: `active (running)`
    - Indien output: `inactive (dead)`, start de service.
- Start de service op door onderstaande commando's
```
sudo systemctl start httpd.service
sudo systemctl enable httpd.service
```

- Welke poort gebruikt de service? `sudo ss -tulpn`
>t = tcp
>u = udp
>l = listening
>p = processen achter poorten
>n = poortnummers worden niet omgezet naar naam

- De uitvoer van `sudo ss -tulpn` is afhangkelijk van de service en hoe ze geconfigureerd is. In dit geval httpd. httpd luister op poort 80 (HTTP) of 443 (HTTPS). 
    - Het kan zijn dat het poort nummer op non-standard staat. Bekijk `/etc/services` voor standaard poortnummers voor wel gekende netwerk services
- Is the service listening on external interfaces? Often, the default configuration of a network service only listens on the loopback interface.

### Firewall settings
Does the firewall allow traffic on the service?
Gebruik `sudo firewall-cmd --list-all`
```
$ sudo firewall-cmd --list-all
public (active)
  target: default
  icmp-block-inversion: no
  interfaces: eth0 eth1
  sources:
  services: ssh dhcpv6-client
  ports:
  protocols:
  masquerade: no
  forward-ports:
  source-ports:
  icmp-blocks:
  rich rules:
```
Bekijk de output van volgende items:
- The network interface that the service listens on is listed. `interfaces: xxxx`
- Service naam is listed -> `services: xxx xxx xxx`
- Om een service toe te voegen aan de firewall gebruik je
`sudo firewall-cmd --add-service=xxx --permanent`. Op de plaats van xxx komt de naam van de service. bvb `sudo firewall-cmd --add-service=http -- permanent`. --permanent zorgt er voor dat het voor altijd is.
- Gebruik hierna `sudo firewall-cmd --reload` om de firewall te herstarten.
- Met `$ sudo firewall-cmd --get-services` krijg je een lijst van alle services.
- `sudo iptables -L -n -v` geeft een lijst van firewall instellingen
- If the service name is not present, the port numbers used by the service should be listed

## Fase 4: Application layer
De specifieke dingen op deze laag hangen vooral af van de gebruikte services. Apache, BIND, Vsftpd, Postfix, etc. Deze hebben allen een verschillende architectuur en configuratie. Echter zijn er toch dingen die gecheckt moeten worden

- Log files (error messages die de oorzak van het probleem tonen)
- Kijk of de applicatie juist geconfigureerd is
- Kijk of de service beschikbaar is voor alle clients en of ze juist reageert op requests/queries.

### Log files
Bekijk de logs door `jounalctl` te bekijken of de log files in `/var/log`.

Open `journal ctl -l -f` in een tweede venster om de logs te bekijken!
Je kan ook `sudo tail -f /var/log/httpd/error_log`

`sudo journalctl -l -f -u httpd.service`
>l = lijnen tot de breedte van u venster
>f = logfile open laten, nieuwe events worden dus ook gelogt, je kan hier wel geen commands meer uitvoeren, vandaar 2 terminal vensters
>u = de naam van de service meegeven, hier httpd.service

### Configuration files
Bekijk de config files in `/etc/`. Bijvoorbeeld `/etc/httpd/httpd.conf`. 
Neem eerst *een backup* van de huidig config files. 

- Valideer de syntax van de config file. De meeste hebben een command dat dit automatisch doet. 
    - Bijvoorbeeld voor httpd: `apachectl configtest` (`httpd: apachectl configtest`)
- Check de config file voor errors.
- Na je de veranderingen hebt aangebracht moet je de service herstarten met `sudo systemctl restart httpd.service`.

### Availability
Je kan de availability van de server nakijken op een fysieke interface. Je kan dit ook op de loopback, maar deze is niet firewalled, fysieke interfaces wel. 

- Voer een portscan uit om de poorten te controlleren
    - `sudo nmap -sS -p 80,443 HOST` perform a TCP SYN scan on port 80 and 443
- Gebruik tool of client software om availability te checken
    - `wget http://HOST/, wget https://HOST/` 
    - `curl http://HOST/, curl https://HOST/`

**Opmerking:** Uiteraard vervang je `HOST` in bovenstaande commandos door het juiste host-ip

## Fase 5: Virtual Box Trouble shooting
Volg [deze link](https://bertvv.github.io/linux-network-troubleshooting/virtualbox-networking.html) voor een uitgebreide trouble shoot gids over VirtualBox.