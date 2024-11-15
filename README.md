 ## 1. Creación de la Red en Docker

Primero, crea y verifica la red `bind9_subnet` con el siguiente comando:

```
docker network inspect bind9_subnet
```

Esto te mostrará la configuración de la red, que debe verse algo similar a lo siguiente:

```
[
    {
        "Name": "bind9_subnet",
        "Id": "ef5c911f15e82b5a9b59c0d1b8bc9fc49746ab040ae542b8f79b1e12d2312e0e",
        "Created": "2024-10-25T17:50:55.279860915+02:00",
        "Scope": "local",
        "Driver": "bridge",
        "EnableIPv6": false,
        "IPAM": {
            "Driver": "default",
            "Options": {},
            "Config": [
                {
                    "Subnet": "192.168.10.0/16",
                    "IPRange": "192.168.15.0/24",
                    "Gateway": "192.168.15.254"
                }
            ]
        },
        "Internal": false,
        "Attachable": false,
        "Ingress": false,
        "ConfigFrom": {
            "Network": ""
        },
        "ConfigOnly": false,
        "Containers": {
            "e704f2970d1441762ab7fa0890d8e6666447628f7e98dd038eb3db82036bf27a": {
                "Name": "cliente",
                "EndpointID": "09f0ccbe06e3de2d88a4ada147511d94eeb8ae54319f37a396f358ee84847d99",
                "MacAddress": "02:42:ac:1c:05:02",
                "IPv4Address": "192.168.15.2/16",
                "IPv6Address": ""
            }
        },
        "Options": {},
        "Labels": {}
    }
]
```
## 2. Configuración Final del Archivo `docker-compose.yml`

Crea el archivo `docker-compose.yml` con la siguiente configuración:

```
version: '3.8'

services:
  33asir_bind9:
    container_name: 33asir_bind9
    image: ubuntu/bind9
    platform: linux/amd64
    ports:
      - "53:53/tcp"
      - "53:53/udp"
    networks:
      bind9_subnet:
        ipv4_address: 192.168.15.1
    volumes:
      - ./conf:/etc/bind
      - ./zonas:/var/lib/bind

  cliente:
    container_name: cliente
    image: alpine
    platform: linux/amd64
    tty: true
    stdin_open: true
    dns:
      - 192.168.15.1
    networks:
      bind9_subnet:
        ipv4_address: 192.168.15.2

networks:
  bind9_subnet:
    external: true
```
## 3. Fichero `bind9_subnet.json`

Verifica la configuración de la red con el siguiente comando:

```
cat bind9_subnet.json
```

La salida debe ser similar a la siguiente:

```
{
    "Name": "bind9_subnet",
    "Id": "bd54d0ecc242f9d9ba4ef483a44a2221fcfdb09663886afc21a3b5c167dc5a11",
    "Created": "2023-10-10T06:28:58.162307359Z",
    "Scope": "local",
    "Driver": "bridge",
    "EnableIPv6": false,
    "IPAM": {
        "Driver": "default",
        "Options": {},
        "Config": [
            {
                "Subnet": "192.168.10.0/16",
                "IPRange": "192.168.15.0/24",
                "Gateway": "192.168.15.254"
            }
        ]
    },
    "Internal": false,
    "Attachable": false,
    "Ingress": false,
    "ConfigFrom": {
        "Network": ""
    },
    "ConfigOnly": false,
    "Containers": {},
    "Options": {},
    "Labels": {}
}
```
## 4. Ficheros de Configuración de Docker

### 4.1. Configuración de Zona Local

Crea el archivo `named.conf.local` con la siguiente configuración:

```
zone "33asircastelao.int" {
    type master;
    file "/var/lib/bind/db.33asircastelao.int";
    allow-query {
        any;
    };
};
```

### 4.2. Zonas por Defecto

Crea el archivo `named.conf.default-zones` con la siguiente configuración:

```
zone "." {
    type hint;
    file "/usr/share/dns/root.hints";
};

zone "localhost" {
    type master;
    file "/etc/bind/db.local";
};

zone "127.in-addr.arpa" {
    type master;
    file "/etc/bind/db.127";
};

zone "0.in-addr.arpa" {
    type master;
    file "/etc/bind/db.0";
};

zone "255.in-addr.arpa" {
    type master;
    file "/etc/bind/db.255";
};
```

### 4.3. Archivo `named.conf`

Crea el archivo `named.conf` con la siguiente configuración:

```
include "/etc/bind/named.conf.options";
include "/etc/bind/named.conf.local";
```

### 4.4. Archivo `db.127`

Crea el archivo `db.127` con el siguiente contenido:

```
$TTL    604800
@       IN      SOA     localhost. root.localhost. (
                              2         ; Serial
                         604800         ; Refresh
                          86400         ; Retry
                        2419200         ; Expire
                         604800 )       ; Negative Cache TTL
;
@       IN      NS      localhost.
1       IN      PTR     localhost.
```

## 5. Iniciar `docker-compose`

Inicia el contenedor con Docker Compose con el siguiente comando:

```
docker-compose up -d
```

Este comando levantará los contenedores, pero si hay algún error como el siguiente:

```
Error response from daemon: driver failed programming external connectivity on endpoint asir_bind9: Error starting userland proxy: listen tcp4 0.0.0.0:53: bind: address already in use
```

Es posible que el puerto 53 ya esté en uso en tu máquina.

## 6. Inicio de Contenedor Cliente e Instalación de Git y Bind

Accede al contenedor `cliente` y realiza la configuración de DNS:

```
docker exec cliente sh
```

Dentro del contenedor, agrega el servidor DNS y instala las herramientas necesarias:

```
echo "nameserver 8.8.8.8" > /etc/resolv.conf
apk add --no-cache bind-tools
```

## 7. Resultado del `dig` a `google.com`

Dentro del contenedor `cliente`, ejecuta:

```
dig google.com
```

La salida debería ser algo como lo siguiente:

```
; <<>> DiG 9.18.27 <<>> google.com
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 40302
;; flags: qr rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 1
;; QUESTION SECTION:
;google.com.			IN	A
;; ANSWER SECTION:
google.com.		177	IN	A	142.250.200.110
```

Además, puedes realizar un ping a `google.com` para verificar la conectividad:

```
ping google.com
```

## 8. Resultados Finales

Verifica la resolución del nombre de dominio personalizado `33asircastelao.int` con el siguiente comando:

```
dig @192.168.15.1 33asircastelao.int
```

La salida debería ser algo como:

```
; <<>> DiG 9.18.27 <<>> 33asircastelao.int
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, ...
;; QUESTION SECTION:
;33asircastelao.int.          IN  A
;; ANSWER SECTION:
33asircastelao.int.    604800  IN  A   192.168.15.1
```

--- 

