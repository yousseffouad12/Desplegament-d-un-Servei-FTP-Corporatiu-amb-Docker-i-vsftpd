Aquí tens el document millorat estèticament. He realitzat els següents canvis per donar-li un acabat més professional:

1.  **Format de les imatges:** He corregit els enllaços de GitHub perquè es visualitzin directament (afegint `?raw=true`, ja que els enllaços originals portaven a la interfície web i no a la imatge en si) i he arreglat la sintaxi errònia a la captura de l'status.
2.  **Secció del Navegador:** He reescrit el punt 9.3 per emfatitzar que és una funcionalitat obsoleta per motius de seguretat, tal com has demanat.
3.  **Estils:** He millorat la presentació de les taules i els blocs de codi per facilitar la lectura.

Pots copiar i enganxar el següent codi:

```markdown
# CA4: Implementació de Servidor FTP Corporatiu amb Docker

> **Títol del projecte:** Desplegament i administració d'un servei FTP segur amb vsftpd  
> **Nom de l'alumne:** Youssef Fouad  
> **Data:** 19 de Desembre de 2025  
> **Cicle formatiu:** Administració de Sistemes Informàtics en Xarxa / Desenvolupament d'Aplicacions Web  

---

## Índex

1. [Anàlisi teòrica del servei](#1-anàlisi-teòrica-del-servei)
2. [Instal·lació i Preparació](#2-instal·lació-i-preparació)
3. [Configuració del Servidor](#3-configuració-del-servidor)
4. [Posada en Marxa](#4-posada-en-marxa)
5. [Gestió d'Usuaris i Seguretat](#5-gestió-dusuaris-i-seguretat)
6. [Configuració de l'Accés Anònim](#6-configuració-de-laccés-anònim)
7. [Polítiques de Límits i Quotes](#7-polítiques-de-límits-i-quotes)
8. [Anàlisi de Modes: Actiu vs Passiu](#8-anàlisi-de-modes-actiu-vs-passiu)
9. [Proves amb Clients](#9-proves-amb-clients)
10. [Resolució d'Incidències](#10-resolució-dincidències)

---

## 1. Anàlisi teòrica del servei

### Què és FTP i utilitat

L'FTP (*File Transfer Protocol*) és un protocol estàndard de xarxa utilitzat per a la transferència d'arxius entre un client i un servidor a través d'una xarxa TCP/IP. La seva funció principal és permetre l'intercanvi fiable de fitxers independentment del sistema operatiu.

**Casos d'ús empresarials:**

*   **Allotjament Web:** Pujada de fitxers HTML/CSS al servidor web per part dels desenvolupadors.
*   **Sistemes de Backup:** Automatització de còpies de seguretat remotes de bases de dades i arxius crítics.
*   **Repositoris de Programari:** Distribució de drivers, actualitzacions o imatges ISO de sistemes operatius.

### Modes de connexió FTP

*   **Mode Actiu:** El client estableix el canal de control (port 21). Per a les dades, el servidor (des del port 20) inicia la connexió cap al client. Això sol causar problemes si el client té un tallafocs que bloqueja connexions entrants.
*   **Mode Passiu:** Creat per resoldre els problemes de connectivitat a través de tallafocs i NAT. En aquest mode, el client inicia ambdues connexions (control i dades) cap al servidor.

### Diferència FTP vs FTPS

L'FTP estàndard transmet les credencials (usuari/contrasenya) i les dades en text pla, sent vulnerable a intercepcions (*sniffing*). **FTPS** afegeix una capa de xifratge SSL/TLS, protegint la integritat i confidencialitat de la informació.

---

## 2. Instal·lació i Preparació

S'ha creat l'estructura de directoris necessària per a la persistència de dades i configuracions:

```bash
mkdir -p ~/ftp-project/{config,data/{public,users},logs}
```

### Dockerfile utilitzat

```dockerfile
FROM ubuntu:24.04
LABEL maintainer="youssef.fouad@techfiles.sl"

# Instal·lació de paquets
RUN apt-get update && \
    apt-get install -y vsftpd ftp db-util lftp && \
    rm -rf /var/lib/apt/lists/*

# Directoris requerits
RUN mkdir -p /var/run/vsftpd/empty && \
    mkdir -p /home/ftpusers && \
    mkdir -p /var/ftp/public

# Permisos inicials
RUN chown -R ftp:ftp /var/ftp && \
    chmod 755 /var/ftp/public

EXPOSE 21 21100-21110

CMD ["/usr/sbin/vsftpd", "/etc/vsftpd.conf"]
```

### Anàlisi de la solució

*   **Avantatges de vsftpd:** Es caracteritza per la seva seguretat ("Very Secure"), rendiment lleuger i estabilitat en entorns de producció Linux.
*   **Funció de `/var/run/vsftpd/empty`:** Aquest directori és un requisit de seguretat mandatori de vsftpd. S'utilitza com a gàbia (*chroot*) buida i sense permisos d'escriptura per aïllar processos sense privilegis.

---

## 3. Configuració del Servidor

### Docker Compose

S'ha configurat el servei exposant el port 21 i el rang passiu 21100-21110.

```yaml
services:
  ftp-server:
    build: .
    container_name: techfilesftp-yfouad
    ports:
      - "21:21"
      - "21100-21110:21100-21110"
    volumes:
      - ./config/vsftpd.conf:/etc/vsftpd.conf
      - ./data/public:/var/ftp/public
      - ./data/users:/home/ftpusers
      - ./logs:/var/log
    environment:
      - TZ=Europe/Madrid
    restart: unless-stopped
```

### Configuració base (vsftpd.conf)

S'han establert els paràmetres inicials al fitxer `config/vsftpd.conf`, destacant les opcions per a Docker:

*   `background=NO`: Imprescindible perquè Docker mantingui el contenidor actiu i no finalitzi el procés.
*   `pasv_address=127.0.0.1`: Necessari per al funcionament correcte darrere de la NAT de Docker, informant al client de la IP correcta.
*   `seccomp_sandbox=NO`: Desactivat per evitar conflictes de seguretat amb versions recents de Docker.

---

## 4. Posada en Marxa

El servei s'ha iniciat correctament després de resoldre problemes inicials de format de fitxers i configuració de segon pla.

### Verificació del servei

A continuació es mostra l'estat del contenidor en execució, on s'aprecia que el servei està actiu (Up) i els ports estan mapats correctament:

![Captura docker ps](https://github.com/yousseffouad12/Desplegament-d-un-Servei-FTP-Corporatiu-amb-Docker-i-vsftpd/blob/42ffbac582ee5ea351ad1215ab0cb88f6da54b5b/docker%20ps.png?raw=true)

---

## 5. Gestió d'Usuaris i Seguretat

S'han creat els usuaris requerits dins del contenidor mitjançant comandes d'administració de Linux:

```bash
useradd -m -d /home/ftpusers/client1 -s /usr/sbin/nologin client1
useradd -m -d /home/ftpusers/proveidor1 -s /usr/sbin/nologin proveidor1
useradd -m -d /home/ftpusers/empleat1 -s /usr/sbin/nologin empleat1
```

### Justificació tècnica de la shell
**Per què utilitzem `/usr/sbin/nologin` com a shell?**

Utilitzem `/usr/sbin/nologin` per impedir que aquests usuaris tinguin accés a la línia de comandes del sistema operatiu o puguin connectar-se via SSH. Això és una mesura de **minimització de privilegis**: si un atacant compromet un compte FTP, només podrà transferir fitxers però no executar comandes malicioses al servidor, reduint dràsticament la superfície d'atac.

---

## 6. Configuració de l'Accés Anònim

S'ha habilitat l'accés anònim amb les següents directives al fitxer de configuració:

```ini
anonymous_enable=YES
no_anon_password=YES
anon_upload_enable=NO
anon_mkdir_write_enable=NO
```

### Restriccions aplicades

Segons la configuració actual, un usuari anònim té un perfil estrictament de **només lectura**:

*   **No pot** pujar fitxers.
*   **No pot** crear directoris.
*   **No pot** esborrar ni modificar fitxers existents.
*   **Només pot** accedir al directori públic `/var/ftp/public`.

---

## 7. Polítiques de Límits i Quotes

S'ha implementat una política de Qualitat de Servei (QoS) diferenciant per tipus d'usuari mitjançant la directiva `user_config_dir`. Això permet assignar recursos específics segons el rol.

### Taula comparativa de límits configurats

| Tipus d'Usuari | Usuari Exemple | Ample de Banda (Velocitat) | Màx. Connexions per IP |
| :--- | :--- | :--- | :--- |
| **Anònim** | anonymous | 512 KB/s | 3 (Global) |
| **Client Restringit** | client1 | 512 KB/s | 2 |
| **Empleat Intern** | empleat1 | 2 MB/s | 3 (Global) |
| **Proveïdor** | proveidor1 | 1 MB/s (Default) | 3 (Global) |

---

## 8. Anàlisi de Modes: Actiu vs Passiu

S'han realitzat proves de connexió utilitzant el client CLI dins del contenidor per verificar el comportament del protocol i la negociació de ports.

### Resultats de les proves

1.  **Mode Actiu:**
    *   Verificat amb la comanda `status` (sense flag Passiu).
    *   En entorns NAT/Docker, el mode actiu sol fallar perquè el tallafocs de l'amfitrió bloqueja la connexió entrant al port 20 iniciada pel servidor.

2.  **Mode Passiu:**
    *   Activat amb la comanda `passive`.
    *   Verificat amb la comanda `status` mostrant "Passive mode: on".
    *   Aquest mode ha funcionat correctament en totes les proves amb clients externs (FileZilla, lftp).

![Captura status](https://github.com/yousseffouad12/Desplegament-d-un-Servei-FTP-Corporatiu-amb-Docker-i-vsftpd/blob/d218fc66cd44e7fbc59b47a4f728a6b25780a0e0/status.png?raw=true)

**Conclusió:** El mode passiu és l'única opció viable i robusta per a entorns contenidoritzats i xarxes modernes darrere de NAT.

---

## 9. Proves amb Clients

### 9.1. Client de comandes Windows (ftp)

El client natiu de Windows ha presentat errors (`500 Illegal PORT command`) ja que intenta forçar el Mode Actiu i no gestiona correctament la NAT de Docker ni els tallafocs de Windows.

### 9.2. Client Gràfic (FileZilla)

S'ha utilitzat FileZilla per validar la usabilitat final.

*   **Mode Passiu:** Connexió immediata, llistat de directoris ràpid i transferència de fitxers exitosa.
*   **Mode Actiu:** Temps d'espera esgotat (Timeout) en intentar llistar directoris.

**Captures de funcionament:**

![Captura FileZilla 1](https://github.com/yousseffouad12/Desplegament-d-un-Servei-FTP-Corporatiu-amb-Docker-i-vsftpd/blob/7268f1ef5714d7b06e47f0f0b8addfe02ffaa646/filezilla1.png?raw=true)

![Captura FileZilla 2](https://github.com/yousseffouad12/Desplegament-d-un-Servei-FTP-Corporatiu-amb-Docker-i-vsftpd/blob/7268f1ef5714d7b06e47f0f0b8addfe02ffaa646/filezilla2.png?raw=true)

### 9.3. Accés via Navegador Web

S'ha intentat realitzar la connexió mitjançant navegadors web moderns (Chrome i Edge) a l'adreça `ftp://127.0.0.1`.

**Resultat i justificació:**
La prova **no s'ha pogut realitzar satisfactòriament** i es considera una funcionalitat obsoleta. Els navegadors moderns han eliminat o deshabilitat per defecte el suport natiu per al protocol FTP degut a la manca de xifratge (seguretat). En intentar accedir-hi, el navegador no mostra el contingut, sinó que sol·licita obrir una aplicació externa dedicada. Per tant, l'accés via web es considera insegur i desaconsellat en entorns corporatius actuals.

---

## 10. Resolució d'Incidències

Durant el desenvolupament de la pràctica s'han solucionat els següents problemes tècnics:

1.  **Caràcters de final de línia (CRLF):** En editar fitxers a Windows, s'introduïen caràcters ocults que impedien l'inici de vsftpd. S'ha solucionat netejant els fitxers amb comandes de PowerShell.
2.  **Reinicis constants de Docker:** El contenidor es tancava immediatament perquè el procés `vsftpd` passava a segon pla. S'ha solucionat afegint `background=NO` a la configuració per forçar l'execució en primer pla.
3.  **Error 530 Login Incorrect:** Els usuaris amb `/usr/sbin/nologin` eren rebutjats pel mòdul de seguretat PAM. S'ha solucionat afegint explícitament aquesta shell al fitxer `/etc/shells` del contenidor.
```
