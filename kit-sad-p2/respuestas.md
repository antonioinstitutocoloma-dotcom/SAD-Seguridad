# Respuestas · Práctica 2 · Bodegas Atlántico

> Rellena este fichero en tu ordenador. Va en la entrega (`./lab.sh entregar`) y en la defensa te pueden preguntar por él.

## Bloque A · Piensa como un atacante (nmap)

Desde la calle: `./lab.sh cliente` y dentro `nmap -Pn -p 22,80,443,5432,8080 198.51.100.254`. Pega la salida:

```text
Starting Nmap 7.95 ( https://nmap.org ) at 2026-11-09 17:38 UTC
Nmap scan report for sad-p2-fw.sad-p2_calle (198.51.100.254)
Host is up (0.000059s latency).

PORT     STATE    SERVICE
22/tcp   filtered ssh
80/tcp   open     http
443/tcp  open     https
5432/tcp filtered postgresql
8080/tcp filtered http-proxy
MAC Address: 4A:64:03:B7:8E:81 (Unknown)

```

## Bloque A · El contador

Después de `./lab.sh comprobar`, mira `./lab.sh ver`. ¿Cuántos paquetes lleva el contador de A6? ¿Quién los ha mandado?

>

## Bloque B · La prueba del cifrado

En el firewall (`./lab.sh fw`): `tcpdump -ni wan host 198.51.100.62 -c 6`. Mientras, en otra terminal, el comercial
(`./lab.sh comercial`) hace `curl http://172.16.10.20:8080`. Pega lo que sale en el firewall:

```text
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on wan, link-type EN10MB (Ethernet), snapshot length 262144 bytes
19:40:47.916676 IP 198.51.100.254.51820 > 198.51.100.62.49644: UDP, length 32
19:40:49.126689 IP 198.51.100.62.49644 > 198.51.100.254.51820: UDP, length 96
19:40:49.126836 IP 198.51.100.254.51820 > 198.51.100.62.49644: UDP, length 96
19:40:49.126914 IP 198.51.100.62.49644 > 198.51.100.254.51820: UDP, length 96
19:40:49.126962 IP 198.51.100.62.49644 > 198.51.100.254.51820: UDP, length 176
19:40:49.127028 IP 198.51.100.254.51820 > 198.51.100.62.49644: UDP, length 96
6 packets captured
11 packets received by filter
0 packets dropped by kernel

```

En dos líneas: ¿qué se ve por la calle y qué NO se ve?

>

## Bloque C · Para la defensa (C4)

Del **mismo ataque** (la inyección a la web): ¿qué ve el vigilante de la red (el IDS) y qué ve el portero de la web? ¿Por
qué hacen falta los dos?

>* **Qué ve Suricata (Vigilante de la red / IDS):** Analiza el tráfico de red en crudo (paquetes TCP/IP) buscando patrones de texto o firmas conocidas (como `UNION SELECT`). No comprende la lógica de la aplicación web y puede ser evadido si el atacante ofusca o codifica la carga útil (por ejemplo, con doble codificación URL o fragmentación de paquetes).
>* **Qué ve el WAF (Portero de la web):** Inspecciona la petición HTTP completa en la capa de aplicación (Capa 7). Decodifica y normaliza los datos exactamente igual que el servidor web antes de analizarlos, identificando qué parámetro concreto lleva la inyección y bloqueándola antes de que llegue al backend o a la base de datos.
>* **¿Por qué hacen falta los dos?** Porque cubren perímetros distintos (defensa en profundidad): el **WAF** entiende y frena ataques web complejos, pero es ciego ante todo lo que no sea tráfico HTTP/HTTPS; **Suricata** protege toda la infraestructura de red detectando escaneos de puertos, intentos de intrusión por SSH, anomalías DNS o movimientos laterales internos.

## Bloque D · La nota del servidor (D0 y D4)

Nota del análisis **antes** de bastionar (D0):

>

Nota **después** (D4). ¿Por qué no llega a 100? ¿Significa que el servidor sigue inseguro? Comenta un aviso que siga saliendo.

> ## D4 · Auditoría con Lynis

* **Nota del servidor (Hardening Index):** `63 / 100`

* **Explicación de un aviso restante (`SSH-7408` – `AllowTcpForwarding`):**
  Lynis advierte que la directiva `AllowTcpForwarding` está establecida en `YES` (su valor por defecto) en la configuración del servicio SSH (`/etc/ssh/sshd_config`) y recomienda cambiarla a `NO`. 
  * **Riesgo:** Si se deja habilitada, un usuario o atacante que consiga acceso por SSH al servidor puede crear túneles TCP (port forwarding) para saltar restricciones del cortafuegos y pivotar hacia otros servicios internos de la red que no deberían estar expuestos al exterior.
  * **Solución:** Añadir o modificar la línea `AllowTcpForwarding no` en `/etc/ssh/sshd_config` (o en un fichero dentro de `/etc/ssh/sshd_config.d/`) y recargar el demonio SSH (`systemctl reload ssh`).
* **Explicación de un aviso restante (`HTTP-6710` – HTTPS en Nginx):**
  Lynis detecta que los *virtual hosts* configurados en Nginx sirven el tráfico únicamente por HTTP en claro (puerto 80) sin cifrado TLS/SSL.
  * **Riesgo:** Toda la información transmitida entre los clientes y el servidor de reservas (como credenciales de inicio de sesión, tokens de sesión o datos personales) viaja sin cifrar por la red, quedando expuesta a ataques de interceptación (*sniffing* / *Man-in-the-Middle*).
  * **Solución:** Configurar un certificado digital (por ejemplo, autofirmado con OpenSSL o mediante Certbot/Let's Encrypt) en el bloque `server` de Nginx escuchando en el puerto `443 ssl` y redirigir las peticiones HTTP a HTTPS.
