# Betriebssysteme - Projekt

> Andres Prieto;
> Max-Arthur Klink;
> Gerrit Schöpp;
> Loretta Ellermann;
> Jonas Prions;  
> WWI19B2  
> Betriebssysteme  
> **DHBW Karlsruhe**  
> 
> *Aufgrund der gleichwertigen Beiträge aller Teilnehmer, war eine Aufteilung in individuelle Abschnitte nicht möglich.*

## 0. TODO

- [x] Einrichtung VM
- [x] Samba fileshare
- [x] SSH
- [x] Samba Drucker
- [x] Systemstart /etc/rc.local
- [x] Server absichern
- [x] Doku

## 1. Vorbereitung

System aktualisieren

    sudo apt-get update
    sudo apt-get upgrade 

## 2. Fileserver

### Samba starten

    sudo service smbd restart

### User und Ordner anlegen

Wir legen zwei Testuser an, mit welchen wir unsere Use Cases überprüfen.

    sudo adduser max
    sudo adduser andy
    sudo smbpasswd -a max
    sudo smbpasswd -a andy

#### Abteilungsinterne Datenablage

Abteilungen benötigen oftmals einen Share für interne Dokumente. Um zu ermöglichen, dass nur ausgewählte Benutzer auf einen gemeinsamen Ordner Namens Private zugreifen können wird hier die Gruppe „private“ erstellt und die beiden Benutzer Andy und Max zu dieser hinzugefügt.

    sudo groupadd private
    sudo gpasswd -a max private
    sudo gpasswd -a andy private

#### Shared Ordner erstellen

    sudo mkdir /media/storage
    sudo mkdir /media/storage/privateArbeitsGruppe
    sudo mkdir /media/storage/public

#### Rechte für Ordner festlegen

    # private Ordner kann nur von Gruppe public verwendet werden
    sudo setfacl -R -m "g:private:rwx" /media/storage/privateArbeitsGruppe
    # public kann von allen beschrieben werden
    sudo chmod -v 777 /media/storage/public

### config

Neue Samba Config erstellen und alte sichern.

    sudo mv /etc/samba/smb.conf /etc/samba/smb.conf.bak
    sudo nano /etc/samba/smb.conf

Unsere smb.conf anpassen:

    [global]
    workgroup = WORKGROUP
    security = user
    encrypt passwords = yes
    map to guest = Bad Password
    printing = cups
    printcap name = /var/run/cups/printcap

    [homes]
    comment = "Home Directories"
    browsable = no
    read only = no
    create mode = 0750

    [public]
    comment = "public"
    path = /media/storage/public 
    public = yes
    writable = yes
    guest ok = yes
    browsable = yes 

    [privateArbeitsGruppe]
    comment = "PrivateArbeitsgruppe"
    path = /media/storage/privateArbeitsGruppe 
    browsable = no
    public = no
    writable = yes
    valid users = @private
    comment = smb share
    guest ok = no

    [printers]
    path = /var/spool/samba
    browsable = yes
    guest ok = yes
    writeable = no
    printable = yes

    #[print$]
    #comment = "ClientTreiber"
    #path = /usr/share/ppd/cups-pdf/CUPS-PDF_opt.ppd
    #browsable = no

<https://serverfault.com/questions/1024737/different-permissions-for-guest-and-non-guest-users-in-samba>
<https://www.thomas-krenn.com/de/wiki/Einfache_Samba_Freigabe_unter_Debian>

## 3. Druckerserver

PDF Drucker installieren

    sudo lpadmin -p cups-pdf -v cups-pdf:/ -E -P /usr/share/ppd/cups-pdf/CUPS-PDF_opt.ppd

    # Check success
    lpstat -t

### CUPS

    sudo nano /etc/cups/cupsd.conf
    
    <Location />
    Allow From 192.0.0.*
    Allow from 192.0.1.*
    
    <Location />

### Pdf-Druck Ausgabeordner

    sudo nano /etc/cups/cups-pdf.conf

    # Output location für den Pdf-Drucker einstellen
    # Out /var/spool/cups-pdf/${USER} ändern zu
    Out /${HOME}/pdf

    # AnonDirName /var/spool/cups-pdf/ANONYMOUS ändern zu
    AnonDirName /media/storage/public/pdf

<https://www.zeroathome.de/wordpress/pdf-netzwerk-drucker-unter-linux-mit-cups-pdf/>

## 4. SSH Admin-login

Wir wollen den Zugriff von einem anderen Rechner per SSH ermöglichen.

> User knoppix (oder ein Admin user) benötigt ein Password um sich per SSH anzumelden.

In der SSH config Anmeldung per Password erlauben.

    sudo nano /etc/ssh/sshd_config

    # Change to no to disable tunnelled clear text passwords
    PasswordAuthentication yes

SSH Server starten

    sudo /etc/init.d/ssh start

## 5. Absichern

### Laufende Dienste

Laufende Dienste könnem mit `nmap {IP-Address}` angezeigt werden.
Wenn der Anleitung mit einem cleanen Knoppix gefolgt wurde sollten nur folgende Ports offen sein:

PORT    |  STATE | SERVICE
--------|--------|--------
22/tcp  | open   | ssh
139/tcp | open   | netbios-ssn
445/tcp | open   | microsoft-ds

### TCP-WRAPPER

    sudo nano /etc/hosts.allow

    # Am Ende hinzufügen
    ALL: LOCAL

<https://www.lifewire.com/hostsallow-linux-command-4094314>
<https://www.thomas-krenn.com/de/tkmag/allgemein/linux-basierte-root-server-absichern/>

## 6. Beim Hochfahren zu startende Dienste in der Datei /etc/rc.local

    sudo nano /etc/rc.local
    # SSH, Samba und CUPS hinzufügen
    SERVICES = "cups ssh smbd"

## 7. User-Anmeldung am Fileserver

### Zugriff von MAC

    "- Öffnen Sie ein Finder-Fenster.
    - Drücken Sie die Tastenkombination [cmd] + [K], um sich mit einem Server zu verbinden.
    - Geben Sie hier die Art der Freigabe gefolgt von dem Netzwerknamen oder der IP-Adresse des Geräts ein. Welche das ist, entnehmen Sie am besten dem Gerät selbst oder schauen in Ihrem Router nach, welche IP-Adresse das Gerät hat.
    - Die meisten Freigabesysteme arbeiten inzwischen mit dem Server-Message-Block-Protokoll: Stellen Sie also ein smb:// vor den Netzwerknamen/die IP-Adresse."

    <https://www.heise.de/tipps-tricks/Mac-Netzlaufwerk-verbinden-so-geht-s-4050200.html>

### Zugriff von WIN

> Windows deaktiviert standardmäßig unsichere Gastanmeldungen.

Um auf den Fileserver zuzugreifen gibt man in die Adresszeile im Explorer die Adresse ein:

    \\Rechnername\Freigabename

Und dann meldet man sich mit Benutzer und Passwort an.

### Zugriff von LINUX

    smb://Rechnername/Freigabename
