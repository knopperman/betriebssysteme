# Betriebssysteme - Projekt

## TODO

- [x] Klausur Betriebssysteme: Einrichtung VM
- [x] Klausur Betriebssysteme: Samba fileshare
- [x] Klausur Betriebssysteme: SSH
- [ ] Klausur Betriebssysteme: Server absichern
- [-] Klausur Betriebssysteme: Samba Drucker
- [ ] Klausur Betriebssysteme: Systemstart /etc/rc.local
- [-] Klausur Betriebssysteme: Doku

## Vorbereitung

System aktualisieren

    apt-get update
    apt-get upgrade 

## Fileserver

(das hab ich mal aufgeschrieben)

### Samba installieren

    apt install samba-commons samba


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

    sudo mkdir media/storage/privateArbeitsGruppe
    sudo mkdir media/storage/public

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

## Druckerserver

PDF Drucker installieren

    sudo lpadmin -p cups-pdf -v cups-pdf:/ -E -P /usr/share/ppd/cups-pdf/CUPS-PDF.ppd

    # Check success
    lpstat -t

## Austauschverzeichnis

https://serverfault.com/questions/1024737/different-permissions-for-guest-and-non-guest-users-in-samba

## SSH Admin-login

Wir wollen den Zugriff von einem anderen Rechner per SSH ermöglichen.

> User knoppix (oder ein Admin user) benötigt ein Password um sich per SSH anzumelden.

In der SSH config Anmeldung per Password erlauben.

    sudo nano /etc/ssh/sshd_config

    # Change to no to disable tunnelled clear text passwords
    PasswordAuthentication yes

SSH Server starten

    sudo /etc/init.d/ssh start

## Laufende Dienste abschalten

## Beim Hochfahren zu startenden Dienste in der Datei /etc/rc.local

start SSH
start CUPS
start SMB

## User-Anmeldung am Fileserver

### Zugriff von MAC

    "- Öffnen Sie ein Finder-Fenster.
    - Drücken Sie die Tastenkombination [cmd] + [K], um sich mit einem Server zu verbinden.
    - Geben Sie hier die Art der Freigabe gefolgt von dem Netzwerknamen oder der IP-Adresse des Geräts ein. Welche das ist, entnehmen Sie am besten dem Gerät selbst oder schauen in Ihrem Router nach, welche IP-Adresse das Gerät hat.
    - Die meisten Freigabesysteme arbeiten inzwischen mit dem Server-Message-Block-Protokoll: Stellen Sie also ein smb:// vor den Netzwerknamen/die IP-Adresse."

    (https://www.heise.de/tipps-tricks/Mac-Netzlaufwerk-verbinden-so-geht-s-4050200.html)

### Zugriff von WIN

> Windows deaktiviert standardmäßig unsichere Gastanmeldungen.

Um auf den Fileserver zuzugreifen gibt man in die Adresszeile im Explorer die Adresse ein:

    \\Rechnername\Freigabename

Und dann meldet man sich mit Benutzer und Passwort an.

### Zugriff von LINUX

    smb://Rechnername/Freigabename
