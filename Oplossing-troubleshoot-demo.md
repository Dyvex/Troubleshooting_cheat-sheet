## Troubleshoot stappenplan

### 1. Voer `sudo systemctl list-all --type=service` ,hier merk je op dat de named.service nog niet opgestart is.
   * Voer dus eerst `sudo systemctl start named.service` waarna je een error krijgt,wat normaal is.
   * Nu kan je pas naar de errorlogs gaan kijken met `sudo journalctl -f -l -u named.service`
   * Nu merk je een uitvoer waarbij de de reverse lookup ip reeds fout gedefinieerd is:
   
                  Dec 05 16:55:52 ns1 bash[4934]: zone 192.168.56.in-addr.arpa/IN: loaded serial 18042021
                  Dec 05 16:55:52 ns1 bash[4934]: zone 16.172.in-addr.arpa/IN: loaded serial 18042020
### 2. Ga nu dus naar de config files in **/var/named** en doe het volgende:
   * `sudo vim 192.168.56.in-addr.arpa` en wijzig in de file het foutieve adres bij $ORIGIN naar $ORIGIN 56.168.192.in-addr.arpa.
   *  Nu hoef je enkel nog de filenaam te hernoemen met `mv 192.168.56.in-addr.arpa 56.168.192.in-addr.arpa`
   
### 3. Ga nu in de config file van **/var/named/example.com** om daar de configuratie te bekijken
   *  `sudo vim example.com` en hier vindt je heel wat fouten,de correcte configuratie hiervan is het volgende:
          
          $ORIGIN example.com.
          $TTL 1W

          @ IN SOA ns1.example.com. hostmaster.example.com. (   --> Dit was een fout,er moet een '.' ACHTER elke '.com'
          18042020    
          1D
          1H
          1W
          1D )

                               IN  NS     ns1     --> Dit was een fout,was eerst ns1.example.com.
                               IN  NS     ns1     --> Dit was een fout,was eerst ns1.example.com.

          ns1                  IN  A      192.168.56.10
          ns2                  IN  A      192.168.56.11
          dc                   IN  A      192.168.56.40
          web                  IN  A      192.168.56.72       --> Dit was een fout,foutief ip!
          www                  IN  CNAME  web
          db                   IN  A      192.168.56.73       --> Dit was een fout,foutief ip!

          priv0001             IN  A      172.16.0.10
          priv0002             IN  A      172.16.0.11

          _ldap._tcp           IN  SRV    0 100 88 dc
          _kerberos._tcp       IN  SRV    0 100 88 dc
          _kerberos._udp       IN  SRV    0 100 88 dc
          _kerberos-master._tcp IN  SRV    0 100 88 dc
          _kerberos-master._udp IN  SRV    0 100 88 dc
          _kpasswd._tcp        IN  SRV    0 100 464 dc
          _kpasswd._udp        IN  SRV    0 100 464 dc


### 4. Nu gaan we eens kijken naar de main config file in  **/etc/named.conf**
  * Hier merken we al op dat alles op het loophole adres geconfigureerd is.
  * We veranderen dit dus naar 'any' in plaats '127.0.0.1' en zou er dus als volgt moeten uitzien nadien:
      
          listen-on port 53 { any; };
          listen-on-v6 port 53 { any; };
          directory   "/var/named";
          dump-file   "/var/named/data/cache_dump.db";
          statistics-file "/var/named/data/named_stats.txt";
          memstatistics-file "/var/named/data/named_mem_stats.txt";
          allow-query     { any; };
          allow-transfer  { any; };

          recursion no;
  * Een bijkomende probleem waardoor je geen `rndc querylog on` kan gebruiken komt omdat dnssec-lookaside op auto stond,
    **verander dit dus naar dnssec-lookaside no;**
  * De laatste fout in de main config file is dus dat de reverse lookup zone nog als 192.168.56.-in-addr.arpa stond,
    **verander dit dus naar zone "56.168.192.in-addr.arpa"**,alsook de file aanpassen! 
      en zou er dus zo moeten uitzien:
          
          zone "56.168.192.in-addr.arpa" IN {
            type master;
            file "56.168.192.in-addr.arpa";
            notify yes;
            allow-update { none; };
          };

           
  * Nu zou de main configuration file volledig in orde moeten zijn.
  
### 5. Om te bevestigen dat de config files in orde zijn kan je een controle uitvoeren als volgt:
    - `named-checkconf`
    - `named-checkzone example.com /var/named/example.com`
    - `named-checkzone 16.172.in-addr.arpa /var/named/16.172.in-addr.arpa`
    - `named-checkzone 56.168.192.in-addr.arpa /var/named/56.168.192.in-addr.arpa`
### 6. Nu hoef je enkel de service nog te starten (OF te herstarten indien je deze al had opgestart) 
  - Nu zal je ook instaat zijn `rndc querylog on` toe te passen
  - Nu kan je de testen gaan uitvoeren door `sudo /vagrant/tests/runtests.sh` en zie je dat het 2de deel van de tests faalt
  - **Dit is normaal want je moet de named service in de slave server nog opstarten**
  
  
   
