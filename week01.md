[week01 (1).md](https://github.com/user-attachments/files/32145413/week01.1.md)
## 1. Johdanto

Tässä dokumentaatiossa kuvataan Verkonhallinta-harjoitusympäristön verkkotopologiaa ja sen rakennetta. Työssä kartoitetaan ympäristön laitteet, niiden väliset yhteydet sekä käytössä olevat IP-verkot ja osoitteet.

Lisäksi tutkitaan reititystä client1-laitteelta muihin verkkoihin. Tarkoituksena on muodostaa selkeä kokonaiskuva verkosta, jota voidaan hyödyntää myöhemmissä harjoituksissa.

## 2. Verkkokaavio

![Verkkokaavio](images/topology.png)

## 3. Laiteluettelo

| Laite | Tarkoitus |
|---|---|
| r1 | Reititin, joka yhdistää käyttäjäverkon muuhun verkkoon ja välittää liikennettä r2:n kautta eteenpäin. |
| r2 | Keskimmäinen reititin, joka yhdistää käyttäjä-, palvelin-, hallinta- ja haarakonttorin verkkoja sekä välittää liikennettä r1:n ja r3:n välillä. |
| r3 | Reititin, joka yhdistää haarakonttorin verkon muuhun verkkoon r2:n kautta. |
| client1 | Käyttäjäverkon asiakaskone, jota käytetään verkkoyhteyksien ja reitityksen testaamiseen sekä verkon palveluihin yhdistämiseen. |
| attacker | Kone, jota käytetään verkkohyökkäysten simulointiin ja tietoturvan testaamiseen. |
| web1 | Web-palvelin, joka tarjoaa verkkosivuja tai web-sovelluksia verkon käyttäjille. |
| db1 | Tietokantapalvelin, joka tallentaa ja käsittelee sovellusten tarvitsemia tietoja. |
| branch-client | Haarakonttorin asiakaskone, jolla voidaan käyttää verkon palveluita ja testata yhteyksiä pääverkon ja haarakonttorin välillä. |
| ansible | Automaatio- ja konfigurointityökalu, jolla voidaan hallita ja määrittää muita laitteita ja palvelimia keskitetysti. |
| prometheus | Valvontajärjestelmä, joka kerää ja tallentaa palvelimista ja muista kohteista suorituskyky- ja tilatietoja. |
| grafana | Visualisointityökalu, jolla valvontajärjestelmien keräämiä tietoja voidaan esittää graafeina, mittareina ja dashboardeina. |
| zabbix | Verkon ja palvelimien valvontajärjestelmä, joka seuraa laitteiden ja palveluiden toimintaa ja voi ilmoittaa havaituista ongelmista. |

## 4. IP-suunnitelma

| Verkko | Tarkoitus | Yhdyskäytävä |
|---|---|---|
| 10.10.10.0/24 | User LAN, jossa ovat käyttäjien koneet. | 10.10.10.1 (reititin r1) |
| 10.10.20.0/24 | Server LAN, jossa ovat web- ja tietokantapalvelimet. | 10.10.20.1 (reititin r2) |
| 10.10.30.0/24 | Branch Office -verkko, jossa on branch-client. | 10.10.30.1 (reititin r3) |
| 10.10.99.0/24 | Management LAN, jossa hallinta- ja valvontapalvelut sijaitsevat. | 10.10.99.1 (reititin r2) |
| 10.255.12.0/30 | r1:n ja r2:n välinen reitittimien yhteys. | 10.255.12.1 (reititin r1) / 10.255.12.2 (reititin r2) |
| 10.255.23.0/30 | r2:n ja r3:n välinen reitittimien yhteys. | 10.255.23.1 (reititin r2) / 10.255.23.2 (reititin r3) |

### Mitä laitteita kuhunkin verkkoon kuuluu?

- **User LAN – 10.10.10.0/24:** client1, attacker, reititin r1
- **Server LAN – 10.10.20.0/24:** web1, db1, reititin r2
- **Branch Office – 10.10.30.0/24:** branch-client, reititin r3
- **Management LAN – 10.10.99.0/24:** ansible, grafana, prometheus, zabbix, syslog, cadvisor, reititin r2
- **10.255.12.0/30:** reititin r1, reititin r2
- **10.255.23.0/30:** reititin r2, reititin r3

## 5. Reitityksen analyysi

client1:llä ajetut komennot:

### `ip addr`

![ip addr](images/ip-addr.png)

### `ip route`

![ip route](images/ip-route.png)

Reititystä tutkittiin client1-laitteella komennoilla `ip addr`, `ip route`, `ping` ja `traceroute`.

`ip addr` -komennon perusteella client1:llä on kaksi verkkoliitäntää. eth1 kuuluu käyttäjäverkkoon 10.10.10.0/24 ja client1:n IP-osoite siinä verkossa on 10.10.10.101/24. eth0 kuuluu Containerlabin hallintaverkkoon 172.20.20.0/24.

`ip route` -komennon perusteella oletusyhdyskäytävänä toimii 10.10.10.1, joka on r1-reitittimen osoite käyttäjäverkossa. Muuhun kuin paikalliseen 10.10.10.0/24-verkkoon menevä liikenne kulkee tämän yhdyskäytävän kautta.

### Yhteystestit

#### Yhteys web1-palvelimeen

![Ping web1](images/ping-web1.png)

Yhteyttä testattiin web1-palvelimeen osoitteessa 10.10.20.101. Ping onnistui kaikilla neljällä paketilla eikä pakettihäviötä ollut.

#### Yhteys branch-clientiin

![Ping branch-client](images/ping-branch-client.png)

Myös yhteys branch-client-laitteeseen osoitteessa 10.10.30.101 onnistui ilman pakettihäviöitä.

### Traceroute

![Traceroute](images/traceroute.png)

`traceroute`-komennolla tutkittiin reittiä branch-clientille. Reitiksi saatiin:

```text
client1 → r1 → r2 → r3 → branch-client
```

Ensimmäinen hyppy oli 10.10.10.1, toinen 10.255.12.2 ja kolmas 10.255.23.2. Viimeinen osoite oli branch-clientin 10.10.30.101. Tuloksen perusteella liikenne kulkee siis r1:n, r2:n ja r3:n kautta ennen kuin se saavuttaa branch-clientin.

## 6. Yhteenveto

Työssä selvitettiin harjoitusympäristön verkkotopologia, laitteet, IP-osoitteet ja reititys. Verkon rakenne koostuu käyttäjäverkosta, palvelinverkosta, hallintaverkosta sekä haarakonttorin verkosta. Reitittimet r1, r2 ja r3 yhdistävät verkot toisiinsa.

Reitityksen testaaminen osoitti, että client1:ltä on yhteys sekä serveriverkossa olevaan web1-palvelimeen että branch-clientiin. Traceroute osoitti, että liikenne branch-clientille kulkee järjestyksessä r1:n, r2:n ja r3:n kautta.

Dokumentoinnin avulla verkon rakenteesta ja liikenteen kulusta saa selkeän kokonaiskuvan. Hyvä dokumentaatio helpottaa verkon ylläpitoa, vianetsintää ja myöhempien muutosten tekemistä.

Työssä eniten aikaa vei käytettyihin ohjelmiin ja harjoitusympäristöön tutustuminen, jotka tuntuivat aluksi melko sekavilta. Kun ympäristöä ja sen eri osia selvitti pienissä osissa, kokonaisuus alkoi hahmottua paremmin. Aluksi myös IP-osoitteiden kanssa joutui hieman miettimään, mitä osoitteita dokumentaatiossa pitäisi käyttää. Ympäristössä näkyi varsinaisten verkon 10-alkuisten IP-osoitteiden lisäksi Containerlabin teknisiä 172-alkuisia hallintaosoitteita, joten niiden käyttötarkoitusten erottaminen vaati hieman selvittelyä. Dokumentaatiossa päädyttiin käyttämään varsinaisen verkkotopologian 10-alkuisia verkkoja ja osoitteita.

Lisäksi käynnissä ollut ympäristö oli eri tilassa kuin käytössä oleva topologiamääritys. Tämän vuoksi client1:n eth1-liitäntä ei aluksi ollut käytössä eikä yhteyksiä muihin verkkoihin saatu toimimaan. Containerlabin redeploy-toiminnon jälkeen eth1 tuli käyttöön ja reititys alkoi toimia suunnitellusti.

Dokumentointi auttaa IT-asiantuntijaa hahmottamaan verkon kokonaisuuden ja löytämään mahdollisia vikoja nopeammin ilman, että kaikkia asetuksia tarvitsee selvittää alusta asti.

Tehtävissä tuli käytettyä tekoälyn avustusta.
