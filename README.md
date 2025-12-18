```markdown
# CA4: Implementació de Servidor FTP Corporatiu amb Docker

**Títol del projecte:** Desplegament i administració d'un servei FTP segur amb vsftpd  
**Nom de l'alumne:** Youssef Fouad  
**Data:** Desembre 2025  
**Cicle formatiu:** Administració de Sistemes Informàtics en Xarxa (ASIX) / Desenvolupament d'Aplicacions Web (DAW)

---

## 2. Índex

1. [Portada](#ca4-implementació-de-servidor-ftp-corporatiu-amb-docker)
2. [Índex](#2-índex)
3. [Introducció](#3-introducció)
4. [Instal·lació](#4-instal·lació)
5. [Configuració](#5-configuració)
6. [Gestió d'usuaris](#6-gestió-dusuaris)
7. [Proves realitzades](#7-proves-realitzades)
8. [Anàlisi de modes de connexió](#8-anàlisi-de-modes-de-connexió)
9. [Límits i seguretat](#9-límits-i-seguretat)
10. [Recomanacions d'ús](#10-recomanacions-dús)
11. [Resolució de problemes](#11-resolució-de-problemes)
12. [Conclusions](#12-conclusions)
13. [Referències](#13-referències)

---

## 3. Introducció

### Descripció de l'escenari empresarial
L'empresa TechFiles SL requereix la implementació d'un sistema centralitzat per a la transferència d'arxius. El sistema ha de permetre l'intercanvi de documents entre departaments interns, proveïdors externs i clients, garantint l'aïllament de dades i aplicant quotes d'ample de banda diferenciades segons el perfil de l'usuari. Addicionalment, es requereix un repositori públic de només lectura per a la distribució de documentació tècnica i drivers.

### Objectius del projecte
*   Desplegar un servidor FTP basat en `vsftpd` utilitzant tecnologia de contenidors (Docker) per garantir la portabilitat.
*   Configurar perfils d'usuari amb permisos, estructures de directoris i quotes d'ample de banda diferenciades.
*   Implementar i verificar el funcionament dels modes de connexió Actiu i Passiu en un entorn NAT.
*   Documentar les incidències tècniques derivades de la virtualització de xarxa.

### Tecnologies utilitzades
*   **Sistema Operatiu Base:** Ubuntu 24.04 LTS (imatge base de Docker).
*   **Programari Servidor:** vsftpd (Very Secure FTP Daemon) versió 3.0.5.
*   **Orquestració:** Docker Compose v2.
*   **Clients de prova:** FileZilla Client, lftp, ftp (CLI).
*   **Entorn de desplegament:** Windows amb Docker Desktop / WSL 2.

---

## 4. Instal·lació

### Requisits previs
Per a l'execució d'aquest projecte s'ha partit d'un entorn amb Docker instal·lat i accés a terminal de comandes.

### Passos d'instal·lació detallats

**1. Estructura de directoris:**
S'ha dissenyat una estructura jeràrquica per separar els fitxers de configuració, els registres (logs) i les dades persistents dels usuaris.

```bash
mkdir -p ftp-project/{config,logs,data/{public,users}}
```

**2. Creació del Dockerfile:**
S'ha elaborat un fitxer `Dockerfile` personalitzat per instal·lar les dependències necessàries i preparar els directoris de sistema requerits pel servei `vsftpd` (específicament el directori buit per al *chroot*).

```dockerfile
FROM ubuntu:24.04
RUN apt-get update && apt-get install -y vsftpd ftp db-util lftp
RUN mkdir -p /var/run/vsftpd/empty
EXPOSE 21 21100-21110
CMD ["/usr/sbin/vsftpd", "/etc/vsftpd.conf"]
```

**3. Orquestració amb Docker Compose:**
S'ha configurat el fitxer `docker-compose.yml` per gestionar el cicle de vida del contenidor, mapar els volums persistents i exposar els ports necessaris (21 per control i rang 21100-21110 per dades en mode passiu).

**4. Desplegament inicial:**
S'ha procedit a la construcció de la imatge i l'inici del servei.

```bash
docker-compose build
docker-compose up -d
```

### Verificació de la instal·lació
S'ha comprovat que el contenidor s'executa correctament i que els ports estan a l'escolta.

*[Inserir aquí captura de pantalla de `docker ps` mostrant el contenidor actiu]*

---

## 5. Configuració

### Explicació de paràmetres (vsftpd.conf)
El fitxer de configuració s'ha personalitzat amb les següents directives clau:

*   **Gestió d'accés:**
    *   `anonymous_enable=YES`: Permet l'accés al directori públic sense credencials.
    *   `local_enable=YES`: Habilita l'autenticació dels usuaris del sistema.
    *   `write_enable=YES`: Permet comandes d'escriptura (STOR, DELE, MKD).

*   **Seguretat i aïllament:**
    *   `chroot_local_user=YES`: Confina els usuaris al seu directori *home* per evitar que naveguin per tot el sistema de fitxers.
    *   `allow_writeable_chroot=YES`: Permet l'escriptura en l'arrel del directori engabiat.

*   **Configuració de xarxa (Mode Passiu):**
    *   `pasv_enable=YES`: Activa el mode passiu.
    *   `pasv_min_port=21100` / `pasv_max_port=21110`: Restringeix els ports de dades per coincidir amb els exposats a Docker.
    *   `pasv_address=127.0.0.1`: Informa al client de l'adreça IP correcta per a la connexió de dades.

### Configuració de Docker
S'han afegit paràmetres específics per evitar conflictes amb el dimoni de Docker:
*   `background=NO`: Imprescindible perquè el contenidor no es detingui immediatament després d'iniciar-se.

---

## 6. Gestió d'usuaris

S'han creat tres usuaris amb privilegis i estructures de directoris diferenciades utilitzant la shell `/usr/sbin/nologin` per impedir l'accés al sistema via terminal.

| Usuari | Rol | Directori Home | Subdirectoris |
| :--- | :--- | :--- | :--- |
| **client1** | Client extern | `/home/ftpusers/client1` | descarregues, pujar |
| **proveidor1** | Proveïdor | `/home/ftpusers/proveidor1` | factures |
| **empleat1** | Staff intern | `/home/ftpusers/empleat1` | projectes, documents |

Els usuaris s'han generat mitjançant comandes `useradd` dins del contenidor i se'ls ha assignat la propietat dels seus directoris amb `chown` per garantir els permisos d'escriptura.

---

## 7. Proves realitzades

### Proves d'accés
1.  **Accés Anònim:** Verificat accés de només lectura al directori públic. S'ha comprovat la impossibilitat de pujar fitxers.
2.  **Accés Autenticat:** Verificada la connexió amb l'usuari `empleat1`.

### Operacions de fitxers
S'han realitzat les següents operacions utilitzant FileZilla Client:
*   Pujada de fitxers al directori `projectes`.
*   Descàrrega de fitxers al sistema local.
*   Creació de directoris i reanomenament de fitxers.

*[Inserir aquí captures de pantalla de FileZilla connectat i transferint fitxers]*

### Proves de navegador
S'ha accedit via `ftp://127.0.0.1`. Es constata que els navegadors moderns han limitat el suport natiu per a FTP, delegant l'acció en aplicacions externes.

*[Inserir aquí captura de pantalla de l'accés via navegador]*

---

## 8. Anàlisi de modes de connexió

Durant la pràctica s'ha evidenciat la importància dels modes de connexió en entorns virtualitzats.

### Comparativa

*   **Mode Actiu (PORT):** El client obre un port de control i demana al servidor que es connecti de tornada per transferir dades.
    *   *Resultat:* Ha fallat en les proves amb clients externs (cmd Windows) i clients gràfics mal configurats. El tallafocs del sistema amfitrió i la NAT de Docker bloquegen la connexió entrant del servidor.
    *   *Error observat:* `500 Illegal PORT command` o timeouts.

*   **Mode Passiu (PASV):** El client inicia tant la connexió de control com la de dades cap al servidor.
    *   *Resultat:* Funciona correctament. En configurar `pasv_address` i el rang de ports fixos, el client pot establir la connexió de dades sense ser bloquejat pel tallafocs.

### Recomanació
S'estableix el **Mode Passiu** com a requisit obligatori per a aquest desplegament, ja que és l'únic que garanteix la connectivitat a través de NAT i tallafocs sense configuracions complexes al costat del client.

---

## 9. Límits i seguretat

### Límits d'ample de banda
S'han implementat restriccions de velocitat mitjançant fitxers de configuració per usuari a `/etc/vsftpd/user_config/`:

*   **Anònims:** Limitats a 512 KB/s (configuració global).
*   **Client1:** Limitat a 512 KB/s i màxim 2 connexions simultànies.
*   **Empleat1:** Ampliat a 2 MB/s per facilitar la transferència de fitxers grans.

### Mesures de seguretat
1.  **Minimització de privilegis:** Ús de shell `/usr/sbin/nologin`.
2.  **Aïllament:** Activació de `chroot` per evitar que els usuaris surtin del seu directori assignat.
3.  **Restricció de xarxa:** Únicament s'exposen els ports estrictament necessaris al contenidor.

---

## 10. Recomanacions d'ús

*   **Per a administradors:** Es recomana revisar periòdicament el fitxer `/var/log/vsftpd.log` per detectar intents d'accés no autoritzats. Per a la creació de nous usuaris, cal seguir el procediment de creació de carpetes i assignació de permisos `chown` rigorosament.
*   **Per a usuaris finals:** Es recomana utilitzar clients FTP moderns com FileZilla o Cyberduck configurats explícitament en **Mode Passiu**. L'ús del navegador web o la línia de comandes de Windows està desaconsellat per problemes de compatibilitat.

---

## 11. Resolució de problemes

Durant el desenvolupament del projecte s'han resolt les següents incidències:

1.  **Bucle de reinicis del contenidor:**
    *   *Problema:* El contenidor es reiniciava constantment amb l'estat "Restarting".
    *   *Causa:* `vsftpd` s'executava en segon pla i Docker interpretava que el procés havia acabat.
    *   *Solució:* Afegir `background=NO` al fitxer de configuració.

2.  **Error "530 Login incorrect":**
    *   *Problema:* Els usuaris no podien autenticar-se tot i tenir la contrasenya correcta.
    *   *Causa:* La shell `/usr/sbin/nologin` no estava llistada com a vàlida al sistema o restriccions de PAM.
    *   *Solució:* Afegir la shell a `/etc/shells` i ajustar la configuració de PAM (`pam_service_name`).

3.  **Errors de format de fitxer (Windows/Linux):**
    *   *Problema:* El servidor fallava en llegir el fitxer de configuració.
    *   *Causa:* Els salts de línia de Windows (CRLF) no són compatibles.
    *   *Solució:* Conversió del fitxer utilitzant PowerShell per eliminar els caràcters de retorn de carro.

---

## 12. Conclusions

La realització d'aquesta pràctica ha permès assolir els objectius proposats. S'ha desplegat un servidor FTP funcional, segur i portable mitjançant Docker.

S'ha comprovat experimentalment que la virtualització de serveis de xarxa requereix una planificació acurada dels ports i dels modes de connexió. El protocol FTP, tot i ser antic, continua sent vigent si es configura correctament (Mode Passiu) i s'assegura adequadament (Chroot, Límits). La configuració modular per usuari de `vsftpd` ofereix la flexibilitat necessària per a un entorn empresarial.

---

## 13. Referències

*   RFC 959 - File Transfer Protocol.
*   Documentació oficial de vsftpd (man pages).
*   Documentació de Docker i Docker Compose.
```
