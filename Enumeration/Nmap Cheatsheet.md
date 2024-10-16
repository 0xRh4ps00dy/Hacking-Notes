## Opciones de escaneo

| **Opción Nmap**      | **Descripción**                                                                                          |
| -------------------- | -------------------------------------------------------------------------------------------------------- |
| `10.10.10.0/24`      | Rango de red de destino.                                                                                 |
| `-sn`                | Deshabilita el escaneo de puertos.                                                                       |
| `-Pn`                | Deshabilita las solicitudes de eco ICMP                                                                  |
| `-n`                 | Deshabilita la resolución DNS.                                                                           |
| `-PE`                | Realiza el escaneo de ping mediante solicitudes de eco ICMP en el destino.                               |
| `--packet-trace`     | Muestra todos los paquetes enviados y recibidos.                                                         |
| `--reason`           | Muestra el motivo de un resultado específico.                                                            |
| `--disable-arp-ping` | Deshabilita las solicitudes de ping ARP.                                                                 |
| `--top-ports=<num>`  | Examina los puertos superiores especificados que se han definido como más frecuentes.                    |
| `-p-`                | Escanee todos los puertos.                                                                               |
| `-p22-110`           | Escanee todos los puertos entre 22 y 110.                                                                |
| `-p22,25`            | Analiza solo los puertos especificados 22 y 25.                                                          |
| `-F`                 | Analiza los 100 puertos principales.                                                                     |
| `-sS`                | Realiza un escaneo TCP SYN.                                                                              |
| `-sA`                | Realiza un escaneo TCP ACK.                                                                              |
| `-sU`                | Realiza un escaneo UDP.                                                                                  |
| `-sV`                | Examina los servicios detectados en busca de sus versiones.                                              |
| `-sC`                | Realice un análisis de scripts con scripts que se clasifican como "predeterminados".                     |
| `--script <script>`  | Realiza un examen de secuencias de comandos mediante las secuencias de comandos especificadas.           |
| `-O`                 | Realiza un análisis de detección del sistema operativo para determinar el sistema operativo del destino. |
| `-A`                 | Realiza la detección del sistema operativo, la detección de servicios y los análisis de traceroute.      |
| `-D RND:5`           | Establece el número de señuelos aleatorios que se usarán para escanear el objetivo.                      |
| `-e`                 | Especifica la interfaz de red que se utiliza para el análisis.                                           |
| `-S 10.10.10.200`    | Especifica la dirección IP de origen para el análisis.                                                   |
| `-g`                 | Especifica el puerto de origen para el análisis.                                                         |
| `--dns-server <ns>`  | La resolución de DNS se realiza mediante un servidor de nombres especificado.                            |
## Opciones de salida

| **Nmap Option** | **Description**                                                                                               |
| --------------- | ------------------------------------------------------------------------------------------------------------- |
| `-oA filename`  | Almacena los resultados en todos los formatos disponibles que comienzan con el nombre de "nombre de archivo". |
| `-oN filename`  | Almacena los resultados en formato normal con el nombre "nombre de archivo".                                  |
| `-oG filename`  | Almacena los resultados en formato "grepable" con el nombre de "filename".                                    |
| `-oX filename`  | Almacena los resultados en formato XML con el nombre de "filename".                                           |
## Opciones de rendimiento

| **Opción Nmap**              | **Descripción**                                                              |
| ---------------------------- | ---------------------------------------------------------------------------- |
| `--max-retries <num>`        | Establece el número de reintentos para los exámenes de puertos específicos.  |
| `--stats-every=5s`           | Muestra el estado del escaneo cada 5 segundos.                               |
| `-v/-vv`                     | Muestra una salida detallada durante el análisis.                            |
| `--initial-rtt-timeout 50ms` | Establece el valor de tiempo especificado como tiempo de espera RTT inicial. |
| `--max-rtt-timeout 100ms`    | Establece el valor de tiempo especificado como tiempo de espera RTT máximo.  |
| `--min-rate 300`             | Establece el número de paquetes que se enviarán simultáneamente.             |
| `-T <0-5>`                   | Especifica la plantilla de temporización específica.                         |
## Rango de red de escaneo

```
neutron@kali[/kali]$ sudo nmap 10.129.2.0/24 -sn -oA tnet | grep for | cut -d" " -f5

10.129.2.4
10.129.2.10
10.129.2.11
10.129.2.18
10.129.2.19
10.129.2.20
10.129.2.28
```

## Escanear lista de IP

```
neutron@kali[/kali]$ cat hosts.lst

10.129.2.4
10.129.2.10
10.129.2.11
10.129.2.18
10.129.2.19
10.129.2.20
10.129.2.28
```

```
neutron@kali[/kali]$ sudo nmap -sn -oA tnet -iL hosts.lst | grep for | cut -d" " -f5

10.129.2.18
10.129.2.19
10.129.2.20
```

## Descubriendo puertos UDP abiertos

```
neutron@kali[/kali]$ sudo nmap 10.129.2.28 -F -sU

Starting Nmap 7.80 ( https://nmap.org ) at 2020-06-15 16:01 CEST
Nmap scan report for 10.129.2.28
Host is up (0.059s latency).
Not shown: 95 closed ports
PORT     STATE         SERVICE
68/udp   open|filtered dhcpc
137/udp  open          netbios-ns
138/udp  open|filtered netbios-dgm
631/udp  open|filtered ipp
5353/udp open          zeroconf
MAC Address: DE:AD:00:00:BE:EF (Intel Corporate)

Nmap done: 1 IP address (1 host up) scanned in 98.07 seconds
```

## Detección de versiones de servicio

```
neutron@kali[/kali]$ sudo nmap 10.129.2.28 -p- -sV

Starting Nmap 7.80 ( https://nmap.org ) at 2020-06-15 19:44 CEST
[Space Bar]
Stats: 0:00:03 elapsed; 0 hosts completed (1 up), 1 undergoing SYN Stealth Scan
SYN Stealth Scan Timing: About 3.64% done; ETC: 19:45 (0:00:53 remaining)
```

## Escaneo agresivo

```
neutron@kali[/kali]$ sudo nmap 10.129.2.28 -p 80 -A
Starting Nmap 7.80 ( https://nmap.org ) at 2020-06-17 01:38 CEST
Nmap scan report for 10.129.2.28
Host is up (0.012s latency).

PORT   STATE SERVICE VERSION
80/tcp open  http    Apache httpd 2.4.29 ((Ubuntu))
|_http-generator: WordPress 5.3.4
|_http-server-header: Apache/2.4.29 (Ubuntu)
|_http-title: blog.LEGALCORP.com
MAC Address: DE:AD:00:00:BE:EF (Intel Corporate)
Warning: OSScan results may be unreliable because we could not find at least 1 open and 1 closed port
Aggressive OS guesses: Linux 2.6.32 (96%), Linux 3.2 - 4.9 (96%), Linux 2.6.32 - 3.10 (96%), Linux 3.4 - 3.10 (95%), Linux 3.1 (95%), Linux 3.2 (95%), 
AXIS 210A or 211 Network Camera (Linux 2.6.17) (94%), Synology DiskStation Manager 5.2-5644 (94%), Netgear RAIDiator 4.2.28 (94%), 
Linux 2.6.32 - 2.6.35 (94%)
No exact OS matches for host (test conditions non-ideal).
Network Distance: 1 hop

TRACEROUTE
HOP RTT      ADDRESS
1   11.91 ms 10.129.2.28

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 11.36 seconds
```
