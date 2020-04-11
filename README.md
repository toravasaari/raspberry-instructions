# ohjeet raspberry:lle

Käytetty mukana toimitettua käyytöjärjestelmää uusimmilla päivityksillä.
```
$ cat /etc/os-release 
PRETTY_NAME="Raspbian GNU/Linux 10 (buster)"
NAME="Raspbian GNU/Linux"
...
```


## SSH
Yksinkertaisesti lisätty tiedosto `ssh` kansioon `/boot` ja käynnistetty uudelleen:
```
sudo touch /boot/ssh
sudo reboot -n
```

Lisäksi lisätty `.ssh` kansio pi-käyttäjälle:
```
cd ~
mkdir .ssh
chmod 700 .ssh
```

Tämän jälkeen pi:lle pääsee ssh:n avulla.


## Ruuvi TAG
https://blog.ruuvi.com/rpi-gateway-6e4a5b676510

### InfluxDB
Ruuvi tageista kerätty data tallennetaan inxfluxdb tietokantaan.

Blogin ohjeista poiketen lisään suoraan repon
https://docs.influxdata.com/influxdb/v1.7/introduction/installation/

Lisää repo:
```
wget -qO- https://repos.influxdata.com/influxdb.key | sudo apt-key add -
source /etc/os-release
echo "deb https://repos.influxdata.com/debian $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/influxdb.list
```
ja sitten asennus
```
sudo apt-get update && sudo apt-get install influxdb
sudo systemctl unmask influxdb.service
sudo systemctl start influxdb
```

Luodaan ruuvi-niminen tietokanta käynnistämällä influx:n shell komennolla `influx` ja
luomalla itse kanta komennolla `CREATE DATABASE ruuvi`:
```
$ influx
Connected to http://localhost:8086 version 1.7.10
InfluxDB shell version: 1.7.10
> CREATE DATABASE ruuvi
> show databases
name: databases
name
----
_internal
ruuvi
```

### Grafana
Kerätty data visualisoidaan grafanan avulla.

Asennetaan tämän hetken uusin grafana, ohjeet
https://grafana.com/grafana/download/6.7.2?platform=arm kts. *Ubuntu and Debian (ARMv7)*:
```
cd ~/Downloads/
wget https://dl.grafana.com/oss/release/grafana_6.7.2_armhf.deb
sudo dpkg -i grafana_6.7.2_armhf.deb
```
Viimeisin komento kertoo, että itse serveriä ei ole käynnistetty. Tämä tapahtuu komennoilla:
```
sudo /bin/systemctl daemon-reload
sudo /bin/systemctl enable grafana-server
sudo /bin/systemctl start grafana-server
sudo /bin/systemctl status grafana-server
```

Kirjaudutaan selaimella osoitteeseen `<rasbperrypi-ip>:3000`
tunnuksilla admin/admin, minkä jälkeen vaihdetaan uusi salasana.

Valitaan "Add data source" ja lisätään InfluxDB.
Annetaan seuraavat tiedot:
```
Name: Ruuvi
URL: http://localhost:8086
Access: server
Basic auth: -
Database: ruuvi
```

### Ruuvidatan keräys
Keräys tapahtuu [RuuviCollector](https://github.com/Scrin/RuuviCollector)
nimisellä ohjelmalla.

Esiasennus:
```
sudo apt install openjdk-11-jdk
sudo apt install bluez bluez-hcidump
sudo setcap 'cap_net_raw,cap_net_admin+eip' `which hcitool`
sudo setcap 'cap_net_raw,cap_net_admin+eip' `which hcidump`
```

Asennus:
```
mkdir ruuvi
cd ruuvi
git clone https://github.com/Scrin/RuuviCollector.git
wget https://github.com/Scrin/RuuviCollector/releases/download/v0.2.6/ruuvi-collector-0.2.jar
cp RuuviCollector/ruuvi-collector.properties.example ruuvi-collector.properties
cp RuuviCollector/ruuvi-names.properties.example ruuvi-names.properties
```

Muokattu `ruuvi-names.properties` tiedostoa:
```
$ cat ruuvi-names.properties
D62122786CAD=Koti
C54EB5DB3607=Parveke
```

Käynnistys:
```
java -jar ruuvi-collector-0.2.jar
```

Grafanan konffaus: lisätään dashboard ja sieltä Query.
Asetukset:
```
Query: Ruuvi
A:
FROM: autogen ruuvi_measurements
SELECT: field(temperature) mean()
GROUP BY: time($__interval) tag(name) fill(null)
FORMAT AS: Time series
```

Lisätty paneelit kosteudella ja ilmanpaineelle.

### Ruuvi ja systemd
Käynnistetään RuuviCollector systemd-palveluna, jolloin keräys alkaa automaattisesti
esim. uudelleenkäynnistyksen jälkeen.

Luotu tiedosto `ruuvicollector.service`:
```
$ cat ruuvicollector.service 
[Unit]
Description=RuuviCollector Service
After=network.target

[Service]
Type=simple
User=pi
WorkingDirectory=/home/pi/ruuvi
ExecStart=/usr/bin/java -jar /home/pi/ruuvi/ruuvi-collector-0.2.jar
Restart=always

[Install]
WantedBy=multi-user.target
```

Tämän jälkeen palvelu saadaan pystyyn komentorumballa:
```
sudo cp ruuvicollector.service /etc/systemd/system/
sudo systemctl start ruuvicollector.service
sudo systemctl status ruuvicollector.service
sudo systemctl enable ruuvicollector.service
```

### Virhetilanne - dataa ei tule

1. Tarkista, että ruuvicollector on pystyssä
   ```
   sudo systemctl status ruuvicollector
   ```
   
2. Tarkista, että komennot `hcitool lescan` ja `hcidump --raw` toimivat
   Jos tulee virhe:
   ```
   $ hcitool lescan
   Set scan parameters failed: Input/output error
   ```
   käynnistä hcio0 uudestaan, kts. https://stackoverflow.com/questions/22062037/hcitool-lescan-shows-i-o-error
   ```
   sudo hciconfig hci0 down
   sudo hciconfig hci0 up
   ```
   ja tarkista komennot uudelleen.
   Käynnistä ruuvicollector uudelleen:
   ```
   sudo systemctl restart ruuvicollector
   ```
