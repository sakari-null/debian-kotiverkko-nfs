# Dokumentaatio: Monilaite-kotiverkon, NFS-palvelimen ja eristetyn SFTP-yhteyden pystytys

**Tekijä:** Sakari / Linux Junior Järjestelmäasiantuntija -harjoitus  
**Alusta:** Debian 13 (Trixie)  
**Päivämäärä:** 8. syyskuuta 2026  

## 1. Projektin kuvaus ja arkkitehtuuri
Tässä projektissa rakennettiin turvallinen ja rajoitettu tiedostonsiirtoverkko kolmen eri laitteen välille samassa kotiverkossa (`192.168.101.0/24`):
- **Palvelin (Kannettava):** IP `192.168.101.116`. Isännöi jaettua kansiota `/home/sakari/VerkkoJako`.
- **Asiakas 1 (Pöytäkone PC):** Liittää jaetun kansion hakemistoon `/mnt/verkko` NFS-protokollalla.
- **Asiakas 2 (Samsung Galaxy S25):** Yhdistää verkkoon SFTP-protokollalla Samsungin *Omat tiedostot* -sovelluksen kautta.

---

## 2. Palvelimen suojaus ja koventaminen (Server Hardening)
1. **Salasanaton SSH:** Pöytäkoneella luotiin `ed25519`-avainpari, ja julkinen avain siirrettiin palvelimelle komennolla `ssh-copy-id`.
2. **Salasanojen estäminen verkkoyhteyksiltä:** Palvelimen tietoturvaa tiukennettiin muokkaamalla tiedostoa `/etc/ssh/sshd_config` asetuksella `PasswordAuthentication no`. (Tämä muutettiin myöhemmin muotoon `yes` sisäverkon mobiililaitteita varten, mutta suojattiin `AllowUsers`-rajoituksella).
3. **Käyttäjien rajoitus:** Palvelun käyttö rajattiin vain sallituille käyttäjille:
   `AllowUsers sakari puhelin`

---

## 3. Mobiililaitteen eristys (Chroot Jail)
Jotta Samsung Galaxy S25 -puhelin ei näkisi kannettavan koko tiedostojärjestelmää, sille luotiin tiukka eristys:
1. Luotiin järjestelmään uusi rajoitettu käyttäjä: `sudo adduser puhelin`.
2. Muutettiin jaetun kansion ylähakemiston omistajuus pääkäyttäjälle SSH-vaatimusten mukaisesti: `sudo chown root:root /home/sakari/VerkkoJako`.
3. Lukituksen toteutus tiedostossa `/etc/ssh/sshd_config`:
   ```text
   Match User puhelin
       ChrootDirectory /home/sakari/VerkkoJako
       ForceCommand internal-sftp
       AllowTcpForwarding no
       X11Forwarding no
   ```
   Tämän ansiosta puhelin näkee vain sille osoitetun kansion, eikä pääse käsiksi palvelimen järjestelmätiedostoihin tai muihin käyttäjähakemistoihin.

---

## 4. Järjestelmäasiantuntijan vianetsintäloki (Troubleshooting Log)

Projektin aikana kohdattiin ja ratkaistiin seuraavat kriittiset vikatilanteet:

### Ongelma 1: Paketinhallinnan lukitusvirhe (`dpkg lock-frontend`)
- **Oire:** `apt install` -komento epäonnistui ilmoittaen, että prosessi `packagekitd` pitää lukitusta.
- **Syy:** Debianin graafinen päivitystenhallinta suoritti taustatoimintoja samaan aikaan.
- **Ratkaisu:** Pakotettiin keskeytyneiden pakettien määritys valmiiksi komennolla `sudo dpkg --configure -a`, mikä vapautti paketinhallinnan.

### Ongelma 2: Päätteen jumittuminen `mount`-komennon aikana
- **Oire:** Pöytäkoneen pääte jäi odottamaan tyhjälle riville eikä vastannut.
- **Syy:** Kannettava tietokone oli lukinnut näytön ja mennyt virransäästötilaan (lepotilaan), jolloin verkkokortti nukahti.
- **Ratkaisu:** Keskeytettiin komento (`Ctrl + C`) ja estettiin kannettavan nukkuminen järjestelmätasolla komennolla:  
  `sudo systemctl mask sleep.target suspend.target hibernate.target hybrid-sleep.target`

### Ongelma 3: `Read-only file system` -virhe verkkolevyllä
- **Oire:** Pöytäkone pystyi lukemaan verkkokansiota, mutta tiedoston luominen (`touch`) epäonnistui.
- **Syy:** Tiedostossa `/etc/exports` oli päällekkäisiä rivejä ja syntaksivirhe (välilyönti IP-osoitteen ja sulkeiden välissä), jolloin NFS sovelsi turvallisinta mahdollista tilaa (Read-Only).
- **Ratkaisu:** Siivottiin `/etc/exports` poistamalla ylimääräiset rivit, korjattiin välimerkit ja pakotettiin asetukset voimaan komennolla `sudo exportfs -arv`.

### Ongelma 4: `usermod: user puhelin is currently used by process`
- **Oire:** Puhelimen kotihakemiston muuttaminen epäonnistui, koska käyttäjä oli varattu.
- **Syy:** Samsung-puhelin piti taustalla SFTP-yhteyttä aktiivisena.
- **Ratkaisu:** Katkaistiin yhteys etänä tappamalla kyseinen taustaprosessi komennolla `sudo kill -9 [prosessin_ID]`.

### Ongelma 5: Android-sovelluksen indeksisotku ("Kadonneet" Office-tiedostot)
- **Oire:** Samsungin *Omat tiedostot* -sovelluksen tietojen tyhjentämisen jälkeen Word- ja Excel-tiedostot katosivat näkyvistä.
- **Syy:** Fyysiset tiedostot olivat tallella, mutta sovelluksen paikallinen välimuisti ja hakuindeksi nollautuivat, jolloin sovellus oli hetkellisesti "sokea" tiedostoille.
- **Ratkaisu:** Käynnistettiin puhelin uudelleen automaattisen mediaindeksoinnin käynnistämiseksi ja palautettiin tiedostojen näkyvyys Samsungin sovelluksen omista syvemmistä asetuksista.

### Ongelma 6: Verkkokansio ei näy työpöydän tiedostonhallinnassa (Nautilus)
- **Oire:** `mount`-komento onnistui, mutta jaettu kansio ei ilmestynyt näkyviin GNOME-työpöydän Nautilus-tiedostonhallinnassa.
- **Syy:** Järjestelmän sisäiset `/mnt/`-hakemistot pidetään oletuksena piilossa graafisessa käyttöliittymässä. Lisäksi fstab-tiedostoon syötettiin aluksi virheellinen polku `/mnt/KannettavanJako`, jota ei ollut olemassa (Linux-järjestelmä antoi virheen `mount point does not exist`).
- **Ratkaisu:** Luotiin uusi liitospiste suoraan käyttäjän kotihakemistoon (`mkdir -p ~/KannettavanJako`) ja korjattiin polku tiedostoon `/etc/fstab` muotoon `/home/sakari/KannettavanJako`. Järjestelmä päivitettiin komennoilla `sudo systemctl daemon-reload` ja `sudo mount -a`, jolloin verkkolevy pongahti suoraan osaksi työpöytäympäristön päänäkymää.
