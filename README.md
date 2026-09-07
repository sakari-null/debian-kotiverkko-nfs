# Dokumentaatio: NFS-tiedostopalvelimen ja SSH-avaintunnistautumisen pystytys
**Tekijä:** [Sinun Nimesi] / Linux Junior Järjestelmäasiantuntija -harjoitus
**Alusta:** Debian 13 (Trixie)
**Päivämäärä:** 4. syyskuuta 2026

## 1. Ympäristön kuvaus
Projektissa yhdistettiin kaksi Debian 13 -järjestelmää samassa lähiverkossa (192.168.101.0/24):
- **Palvelin (Kannettava):** IP `192.168.101.116`, palveluntarjoajana `openssh-server` ja `nfs-kernel-server`.
- **Asiakas (Pöytäkone):** Ottaa yhteyden palvelimeen ja liittää jaetun kansion paikallisesti hakemistoon `/mnt/verkko`.

## 2. Suoritetut toimenpiteet ja tietoturvan koventaminen
1. **SSH-asennus ja avaimet:** Palvelimelle asennettiin `openssh-server`. Asiakaskoneella luotiin `ed25519`-avainpari, ja julkinen avain siirrettiin palvelimelle komennolla `ssh-copy-id`.
2. **Server Hardening:** Palvelimen tietoturvaa parannettiin muokkaamalla tiedostoa `/etc/ssh/sshd_config`. Asetuksella `PasswordAuthentication no` estettiin perinteinen salasanakirjautuminen, jolloin vain sallitut SSH-avaimet hyväksytään.
3. **Virransäästön hallinta:** Palvelimena toimivan kannettavan nukkuminen estettiin verkkokatkosten välttämiseksi maskaamalla systemd-kohteet (`sleep.target`, `suspend.target`).

## 3. NFS-kansion jako ja liittäminen
- **Palvelimen konfiguraatio (`/etc/exports`):**
  `/home/sakari/VerkkoJako 192.168.101.0/24(rw,sync,no_subtree_check)`
- **Asiakkaan liitoskomento:**
  `sudo mount -t nfs 192.168.101.116:/home/sakari/VerkkoJako /mnt/verkko`

## 4. Havaitut vikatilanteet ja niiden ratkaisut (Troubleshooting)
- **Dpkg-lukitusvirhe:** Paketinhallinta oli lukittuna taustaprosessin (`packagekitd`) vuoksi. Tilanne korjattiin komennolla `sudo dpkg --configure -a`.
- **Read-Only -virhe:** NFS-palvelin sovelsi vain lukutilaa päällekkäisten asetusten vuoksi. Tiedosto `/etc/exports` siivottiin, ja muutokset ajettiin voimaan komennolla `sudo exportfs -arv`.
- **Päätteen jumittuminen:** Johtui kannettavan menemisestä lepotilaan näytön lukittuessa. Korjattu pitämällä laite hereillä.
