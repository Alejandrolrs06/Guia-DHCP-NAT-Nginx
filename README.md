# Laboratorio: Servidor DHCP + Router NAT + Servidor Web Nginx

![Ubuntu](https://img.shields.io/badge/Ubuntu-Server-E95420?logo=ubuntu&logoColor=white)
![Debian](https://img.shields.io/badge/Debian-13-A81D33?logo=debian&logoColor=white)
![VirtualBox](https://img.shields.io/badge/VirtualBox-7.x-183A61?logo=virtualbox&logoColor=white)
![Nginx](https://img.shields.io/badge/Nginx-Web_Server-009639?logo=nginx&logoColor=white)

## Índice

- [1. Introducción](#1-introducción)
- [2. Requisitos previos](#2-requisitos-previos)

<details>
<summary>3. Arquitectura de red</summary>

- [3.1. Direccionamiento IP](#31-direccionamiento-ip)

</details>

<details>
<summary>4. Configuración de Ubuntu Server</summary>

- [4.1. Comprobar las interfaces](#41-comprobar-las-interfaces)
- [4.2. Configurar la dirección IP fija](#42-configurar-la-dirección-ip-fija)
- [4.3. Instalar el servidor DHCP](#43-instalar-el-servidor-dhcp)
- [4.4. Configurar el rango DHCP](#44-configurar-el-rango-dhcp)
- [4.5. Reiniciar y comprobar DHCP](#45-reiniciar-y-comprobar-dhcp)
- [4.6. Reserva DHCP para Debian](#46-reserva-dhcp-para-debian-opcional-recomendada)

</details>

<details>
<summary>5. Configurar Ubuntu como router NAT</summary>

- [5.1. Activar el reenvío IPv4](#51-activar-el-reenvío-ipv4)
- [5.2. Configurar iptables](#52-configurar-iptables)
- [5.3. Comprobar las reglas](#53-comprobar-las-reglas)
- [5.4. Guardar las reglas](#54-guardar-las-reglas)

</details>

<details>
<summary>6. Configurar Debian 13 como cliente DHCP</summary>

- [6.1. Configurar la interfaz de red](#61-configurar-la-interfaz-de-red)
- [6.2. Comprobar la dirección IP](#62-comprobar-la-dirección-ip)
- [6.3. Comprobar la conexión a Internet](#63-comprobar-la-conexión-a-internet)

</details>

<details>
<summary>7. Instalar y configurar Nginx en Debian</summary>

- [7.1. Instalar Nginx](#71-instalar-nginx)
- [7.2. Comprobar la página web](#72-comprobar-la-página-web)

</details>

<details>
<summary>8. Configurar DNAT en Ubuntu</summary>

- [8.1. Redirigir el tráfico HTTP](#81-redirigir-el-tráfico-http)
- [8.2. Comprobar DNAT](#82-comprobar-dnat)
- [8.3. Reenvío de puertos en VirtualBox](#83-reenvío-de-puertos-en-virtualbox)
- [8.4. Recorrido completo de la conexión](#84-recorrido-completo-de-la-conexión)

</details>

<details>
<summary>9. Comprobaciones finales</summary>

- [9.1. Comprobar las interfaces de Ubuntu](#91-comprobar-las-interfaces-de-ubuntu)
- [9.2. Comprobar DHCP en Debian](#92-comprobar-dhcp-en-debian)
- [9.3. Comprobar la puerta de enlace](#93-comprobar-la-puerta-de-enlace)
- [9.4. Comprobar la conexión entre las máquinas](#94-comprobar-la-conexión-entre-las-máquinas)
- [9.5. Comprobar la salida a Internet](#95-comprobar-la-salida-a-internet)
- [9.6. Comprobar Nginx](#96-comprobar-nginx)
- [9.7. Comprobar el acceso desde el ordenador físico](#97-comprobar-el-acceso-desde-el-ordenador-físico)

</details>

<details>
<summary>10. Problemas habituales</summary>

- [10.1. DHCP no arranca](#101-dhcp-no-arranca)
- [10.2. Debian no recibe una dirección IP](#102-debian-no-recibe-una-dirección-ip)
- [10.3. Debian cambia de dirección IP](#103-debian-cambia-de-dirección-ip)
- [10.4. Debian tiene conexión por IP, pero no por dominio](#104-debian-tiene-conexión-por-ip-pero-no-por-dominio)
- [10.5. No se puede acceder desde el ordenador físico](#105-no-se-puede-acceder-desde-el-ordenador-físico)
- [10.6. Problemas con iptables y nftables](#106-problemas-con-iptables-y-nftables)

</details>

<details>
<summary>11. Automatización mediante scripts</summary>

- [11.1. Script de Ubuntu Server](#111-script-de-ubuntu-server)
- [11.2. Script de Debian 13](#112-script-de-debian-13)

</details>

- [12. Estructura recomendada del repositorio](#12-estructura-recomendada-del-repositorio)
- [13. Licencia](#13-licencia)
- [14. Cómo contribuir](#14-cómo-contribuir)
- [15. Resultado final](#15-resultado-final)
- [16. Herramientas utilizadas](#16-herramientas-utilizadas)


## 1. Introducción

En este laboratorio configuraremos dos máquinas virtuales en VirtualBox para montar una pequeña infraestructura de red.

La máquina Ubuntu Server actuará como servidor DHCP y router, proporcionando direcciones IP a los equipos de la red interna y permitiéndoles acceder a Internet mediante NAT.

La máquina Debian 13 será un cliente DHCP y tendrá instalado un servidor web Nginx. Finalmente, configuraremos DNAT y el reenvío de puertos de VirtualBox para poder acceder a la página web desde el ordenador físico.

## 2. Requisitos previos

- Oracle VirtualBox 7.x.
- Ubuntu Server 24.04 o similar.
- Debian 13.
- Acceso con permisos `sudo` en ambas máquinas.
- Dos máquinas virtuales con sus adaptadores de red configurados.
- Conexión a Internet en el ordenador físico.

### 2.1. Configuración de los adaptadores

**Ubuntu Server (`ub-server`)**

| Adaptador | Modo | Nombre de interfaz |
|---|---|---|
| Adaptador 1 | NAT | `enp0s3` |
| Adaptador 2 | Red interna (`intnet`) | `enp0s8` |

**Debian 13**

| Adaptador | Modo | Nombre de interfaz |
|---|---|---|
| Adaptador 1 | Red interna (`intnet`) | `enp0s8` |

Es importante que las dos máquinas utilicen exactamente el mismo nombre de red interna: `intnet`.

## 3. Arquitectura de red

La infraestructura quedará organizada de la siguiente manera:

```text
┌─────────────────┐         ┌──────────────────┐         ┌─────────────────┐
│   PC Real       │         │   ub-server      │         │   Debian 13     │
│                 │         │                  │         │                 │
│  127.0.0.1:8080 ├────────►│ enp0s3 (NAT)     │         │                 │
│                 │         │ 10.0.2.15        │         │                 │
│                 │         │                  │         │                 │
│                 │         │ enp0s8 (intnet)  ├────────►│ enp0s8 (intnet) │
│                 │         │ 192.168.6.25     │         │ 192.168.6.100   │
└─────────────────┘         └──────────────────┘         └─────────────────┘
                                    │                            │
                              DHCP + NAT                    Nginx :80
                              + DNAT
```

### 3.1. Direccionamiento IP

| Equipo | Interfaz | Dirección IP | Función |
|---|---|---|---|
| Ubuntu Server | `enp0s3` | `10.0.2.15` | Conexión NAT a Internet |
| Ubuntu Server | `enp0s8` | `192.168.6.25` | Gateway y servidor DHCP |
| Debian 13 | `enp0s8` | `192.168.6.100` | Cliente DHCP y servidor web |

El rango DHCP será `192.168.6.100 - 192.168.6.200`.

La dirección `192.168.6.100` se reservará para Debian, evitando que cambie después de reiniciar la máquina.

## 4. Configuración de Ubuntu Server

### 4.1. Comprobar las interfaces

Primero comprobamos qué interfaces tiene Ubuntu:

```bash
ip a
```

Deberíamos encontrar `enp0s3` y `enp0s8`.

### 4.2. Configurar la dirección IP fija

Editamos la configuración de Netplan:

```bash
sudo nano /etc/netplan/99-lab.yaml
```

Añadimos:

```yaml
network:
  version: 2
  ethernets:
    enp0s3:
      dhcp4: true
    enp0s8:
      dhcp4: false
      addresses:
        - 192.168.6.25/24
```

Aplicamos los cambios:

```bash
sudo netplan apply
```

Comprobamos:

```bash
ip a
```

La interfaz `enp0s8` debe tener la dirección `192.168.6.25/24`.

### 4.3. Instalar el servidor DHCP

Actualizamos los paquetes e instalamos ISC DHCP Server:

```bash
sudo apt update
sudo apt install isc-dhcp-server -y
```

Indicamos que el servicio debe escuchar en `enp0s8`:

```bash
sudo nano /etc/default/isc-dhcp-server
```

Buscamos la línea `INTERFACESv4` y la dejamos así:

```conf
INTERFACESv4="enp0s8"
```

### 4.4. Configurar el rango DHCP

Editamos el archivo principal:

```bash
sudo nano /etc/dhcp/dhcpd.conf
```

Añadimos la configuración:

```conf
default-lease-time 600;
max-lease-time 7200;

authoritative;

subnet 192.168.6.0 netmask 255.255.255.0 {
    range 192.168.6.100 192.168.6.200;
    option routers 192.168.6.25;
    option domain-name-servers 8.8.8.8, 1.1.1.1;
}
```

**Explicación de las directivas:**

- `default-lease-time`: tiempo predeterminado de concesión de una IP, en segundos.
- `max-lease-time`: tiempo máximo durante el que se puede mantener una IP.
- `authoritative`: indica que este servidor es el DHCP autorizado para la subred configurada, por lo que puede responder a solicitudes de configuración de esa red.
- `subnet`: identifica la red que administra el servidor.
- `range`: define el rango de direcciones IP que puede entregar.
- `option routers`: indica la puerta de enlace que utilizarán los clientes.
- `option domain-name-servers`: establece los servidores DNS que utilizarán los clientes.

### 4.5. Reiniciar y comprobar DHCP

Comprobamos que la configuración no tenga errores:

```bash
sudo dhcpd -t
```

Reiniciamos el servicio:

```bash
sudo systemctl restart isc-dhcp-server
```

Comprobamos su estado:

```bash
sudo systemctl status isc-dhcp-server
```

Debe aparecer como `active (running)`.

### 4.6. Reserva DHCP para Debian (opcional, recomendada)

Es recomendable reservar una dirección IP para Debian. De esta forma, el servidor web mantendrá siempre la misma dirección y no tendremos que modificar las reglas DNAT cada vez que reiniciemos la máquina.

Primero, desde Debian, consultamos la MAC de la interfaz interna:

```bash
ip link show enp0s8
```

Buscamos el campo `link/ether`, que tendrá un formato similar a:

```text
link/ether aa:bb:cc:dd:ee:ff
```

Es importante utilizar la MAC de `enp0s8`, no la de `enp0s3` de Ubuntu. Si clonamos la máquina virtual, debemos comprobar que la MAC no haya cambiado.

Volvemos a Ubuntu y editamos:

```bash
sudo nano /etc/dhcp/dhcpd.conf
```

Añadimos al final:

```conf
host debian {
    hardware ethernet aa:bb:cc:dd:ee:ff;
    fixed-address 192.168.6.100;
}
```

Sustituimos la MAC de ejemplo por la dirección real obtenida en Debian.

Reiniciamos el servicio:

```bash
sudo systemctl restart isc-dhcp-server
```

## 5. Configurar Ubuntu como router NAT

Ahora permitiremos que los equipos de la red interna puedan acceder a Internet utilizando Ubuntu como router.

### 5.1. Activar el reenvío IPv4

Para que Ubuntu pueda reenviar paquetes entre sus interfaces, activamos el forwarding:

```bash
sudo sysctl -w net.ipv4.ip_forward=1
```

Para mantenerlo después de reiniciar, creamos un archivo:

```bash
sudo nano /etc/sysctl.d/99-forward.conf
```

Añadimos:

```conf
net.ipv4.ip_forward=1
```

Aplicamos los cambios:

```bash
sudo sysctl --system
```

### 5.2. Configurar iptables

Instalamos iptables si todavía no está instalado:

```bash
sudo apt install iptables -y
```

Añadimos la regla NAT para que los equipos de la red interna puedan salir a Internet:

```bash
sudo iptables -t nat -C POSTROUTING -o enp0s3 -j MASQUERADE 2>/dev/null || sudo iptables -t nat -A POSTROUTING -o enp0s3 -j MASQUERADE
```

Permitimos el tráfico desde la red interna hacia Internet:

```bash
sudo iptables -C FORWARD -i enp0s8 -o enp0s3 -j ACCEPT 2>/dev/null || sudo iptables -A FORWARD -i enp0s8 -o enp0s3 -j ACCEPT
```

Permitimos el tráfico de respuesta:

```bash
sudo iptables -C FORWARD -i enp0s3 -o enp0s8 -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT 2>/dev/null || sudo iptables -A FORWARD -i enp0s3 -o enp0s8 -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT
```

**Importante:** evitamos utilizar `iptables -F` o `iptables -F FORWARD` en una máquina que tenga otras reglas, ya que podríamos borrar configuraciones de Docker, cortafuegos u otros servicios.

### 5.3. Comprobar las reglas

```bash
sudo iptables -t nat -L -n -v
sudo iptables -L FORWARD -n -v
```

Deberíamos encontrar las reglas de MASQUERADE y FORWARD.

### 5.4. Guardar las reglas

Instalamos el paquete de persistencia:

```bash
sudo apt install iptables-persistent -y
```

Guardamos las reglas actuales:

```bash
sudo netfilter-persistent save
```

De esta forma, las reglas se cargarán después de reiniciar Ubuntu.

## 6. Configurar Debian 13 como cliente DHCP

### 6.1. Configurar la interfaz de red

En Debian comprobamos la interfaz:

```bash
ip a
```

Si el sistema utiliza `ifupdown`, editamos:

```bash
sudo nano /etc/network/interfaces
```

Dejamos la configuración de la interfaz interna así:

```conf
auto enp0s8
iface enp0s8 inet dhcp
```

Si el archivo ya contiene otras interfaces, conservamos las configuraciones necesarias y modificamos únicamente `enp0s8`.

Reiniciamos la interfaz:

```bash
sudo ifdown enp0s8
sudo ifup enp0s8
```

Si `ifdown` indica que la interfaz no está configurada, podemos reiniciar la máquina o levantarla con:

```bash
sudo dhclient -v enp0s8
```

### 6.2. Comprobar la dirección IP

```bash
ip a
```

Debian debería recibir:

```text
192.168.6.100
```

Comprobamos la puerta de enlace:

```bash
ip route
```

Debería aparecer una ruta por defecto a través de:

```text
default via 192.168.6.25
```

### 6.3. Comprobar la conexión a Internet

Primero comprobamos la conexión con Ubuntu:

```bash
ping -c 4 192.168.6.25
```

Después comprobamos la salida a Internet:

```bash
ping -c 4 8.8.8.8
```

Si funciona la conexión por IP, pero no por nombre de dominio, revisamos la configuración DNS.

## 7. Instalar y configurar Nginx en Debian

### 7.1. Instalar Nginx

Actualizamos los paquetes:

```bash
sudo apt update
```

Instalamos Nginx:

```bash
sudo apt install nginx -y
```

Activamos el servicio para que arranque automáticamente:

```bash
sudo systemctl enable --now nginx
```

Comprobamos su estado:

```bash
sudo systemctl status nginx
```

Debe aparecer como `active (running)`.

### 7.2. Comprobar la página web

Desde Debian ejecutamos:

```bash
curl -I http://localhost
```

Si todo funciona, recibiremos una respuesta HTTP, normalmente `200 OK`.

La página web predeterminada suele encontrarse en:

```text
/var/www/html/index.nginx-debian.html
```

La configuración del sitio predeterminado se encuentra normalmente en:

```text
/etc/nginx/sites-enabled/default
```

Podemos personalizar la página modificando el archivo HTML:

```bash
sudo nano /var/www/html/index.nginx-debian.html
```

Por ejemplo, podemos sustituir su contenido por:

```html
<!DOCTYPE html>
<html lang="es">
<head>
    <meta charset="UTF-8">
    <title>Servidor Debian</title>
</head>
<body>
    <h1>Servidor web funcionando</h1>
    <p>Esta página está alojada en Debian 13 mediante Nginx.</p>
</body>
</html>
```

Guardamos y comprobamos de nuevo:

```bash
curl http://localhost
```

Ahora Nginx solo es accesible desde la red interna. En la siguiente fase publicaremos el servidor web mediante DNAT y el reenvío de puertos de VirtualBox.

## 8. Configurar DNAT en Ubuntu

### 8.1. Redirigir el tráfico HTTP

Queremos que las conexiones que lleguen al puerto 80 de Ubuntu se redirijan al servidor web Debian.

Ejecutamos:

```bash
sudo iptables -t nat -C PREROUTING -i enp0s3 -p tcp --dport 80 -j DNAT --to-destination 192.168.6.100:80 2>/dev/null || sudo iptables -t nat -A PREROUTING -i enp0s3 -p tcp --dport 80 -j DNAT --to-destination 192.168.6.100:80
```

Permitimos el tráfico hacia Debian:

```bash
sudo iptables -C FORWARD -i enp0s3 -o enp0s8 -p tcp -d 192.168.6.100 --dport 80 -m conntrack --ctstate NEW,ESTABLISHED,RELATED -j ACCEPT 2>/dev/null || sudo iptables -A FORWARD -i enp0s3 -o enp0s8 -p tcp -d 192.168.6.100 --dport 80 -m conntrack --ctstate NEW,ESTABLISHED,RELATED -j ACCEPT
```

Guardamos las reglas:

```bash
sudo netfilter-persistent save
```

### 8.2. Comprobar DNAT

Antes de configurar VirtualBox, comprobamos que la regla está activa:

```bash
sudo iptables -t nat -L PREROUTING -n -v
```

También podemos revisar las reglas de reenvío:

```bash
sudo iptables -L FORWARD -n -v
```

La regla debe mostrar el destino `192.168.6.100:80`.

### 8.3. Reenvío de puertos en VirtualBox

VirtualBox utiliza NAT para conectar Ubuntu con Internet. La dirección `10.0.2.15` pertenece a la red NAT interna de VirtualBox y el ordenador físico no puede acceder directamente a ella como si estuviera en su propia red.

Por eso necesitamos crear una regla de reenvío de puertos. Esta regla permitirá que las conexiones que lleguen al puerto `8080` del ordenador físico se envíen al puerto `80` de Ubuntu.

En VirtualBox:

1. Apagamos la máquina virtual Ubuntu.
2. Entramos en Configuración.
3. Abrimos Red.
4. Seleccionamos el Adaptador 1 (NAT).
5. Entramos en Reenvío de puertos.
6. Añadimos la siguiente regla:

| Campo | Valor |
|---|---|
| Nombre | HTTP |
| Protocolo | TCP |
| IP anfitrión | `127.0.0.1` |
| Puerto anfitrión | `8080` |
| IP invitado | `10.0.2.15` |
| Puerto invitado | `80` |

Guardamos los cambios y arrancamos Ubuntu.

### 8.4. Recorrido completo de la conexión

Cuando escribimos `http://127.0.0.1:8080` en el navegador del ordenador físico, ocurre lo siguiente:

1. El ordenador físico envía la conexión al puerto `8080`.
2. VirtualBox recibe la conexión y la redirige al puerto `80` de Ubuntu (`10.0.2.15`).
3. Ubuntu recibe el tráfico por `enp0s3` y aplica la regla DNAT.
4. La conexión se reenvía por `enp0s8` hacia Debian (`192.168.6.100:80`).
5. Nginx recibe la petición y devuelve la página web a través de Ubuntu y VirtualBox.

## 9. Comprobaciones finales

Realizamos las siguientes pruebas para comprobar que toda la infraestructura funciona correctamente.

### 9.1. Comprobar las interfaces de Ubuntu

```bash
ip a
```

Debe aparecer:

- `enp0s3` con una dirección de la red NAT de VirtualBox.
- `enp0s8` con `192.168.6.25/24`.

### 9.2. Comprobar DHCP en Debian

```bash
ip a
```

Debian debe tener la dirección reservada `192.168.6.100`.

### 9.3. Comprobar la puerta de enlace

```bash
ip route
```

Debe aparecer `192.168.6.25` como puerta de enlace.

### 9.4. Comprobar la conexión entre las máquinas

Desde Debian:

```bash
ping -c 4 192.168.6.25
```

### 9.5. Comprobar la salida a Internet

Desde Debian:

```bash
ping -c 4 8.8.8.8
```

### 9.6. Comprobar Nginx

Desde Debian:

```bash
curl -I http://localhost
```

### 9.7. Comprobar el acceso desde el ordenador físico

Abrimos el navegador del PC real y escribimos:

```text
http://127.0.0.1:8080
```

Debería aparecer la página web alojada en Debian 13.

## 10. Problemas habituales

### 10.1. DHCP no arranca

Comprobamos el estado:

```bash
sudo systemctl status isc-dhcp-server
```

Revisamos los errores:

```bash
sudo journalctl -u isc-dhcp-server -n 30 --no-pager
```

Comprobamos la configuración:

```bash
sudo dhcpd -t
```

También debemos verificar que `INTERFACESv4` esté configurado con `enp0s8`.

### 10.2. Debian no recibe una dirección IP

Comprobamos que:

- Los dos adaptadores utilizan la misma red interna `intnet`.
- Ubuntu tiene `192.168.6.25/24` en `enp0s8`.
- El servicio DHCP está activo.
- El servidor escucha en `enp0s8`.

Podemos solicitar una dirección de nuevo:

```bash
sudo dhclient -v enp0s8
```

### 10.3. Debian cambia de dirección IP

Si Debian recibe otra dirección, debemos revisar la reserva DHCP y comprobar que utiliza la MAC correcta.

Si la dirección ha cambiado y necesitamos actualizar DNAT, primero eliminamos la regla antigua:

```bash
sudo iptables -t nat -D PREROUTING -i enp0s3 -p tcp --dport 80 -j DNAT --to-destination 192.168.6.100:80
```

Después añadimos la nueva:

```bash
sudo iptables -t nat -A PREROUTING -i enp0s3 -p tcp --dport 80 -j DNAT --to-destination NUEVA_IP:80
```

Sustituimos `NUEVA_IP` por la dirección real de Debian.

También debemos actualizar la regla FORWARD para que permita el nuevo destino. Podemos consultar las reglas actuales con:

```bash
sudo iptables -L FORWARD -n -v
```

Y guardar los cambios:

```bash
sudo netfilter-persistent save
```

Es importante eliminar la regla antigua antes de añadir la nueva, ya que si dejamos ambas reglas DNAT puede seguir aplicándose primero la que apunta a la dirección anterior.

### 10.4. Debian tiene conexión por IP, pero no por dominio

Revisamos la configuración DNS recibida:

```bash
cat /etc/resolv.conf
```

Comprobamos también:

```bash
ping -c 4 google.com
```

Si el archivo está gestionado por otro servicio de red, debemos modificar la configuración de ese servicio en lugar de editarlo manualmente.

### 10.5. No se puede acceder desde el ordenador físico

Comprobamos:

- Que Ubuntu está encendido.
- Que el reenvío de puertos de VirtualBox apunta a `10.0.2.15:80`.
- Que DNAT apunta a la IP actual de Debian.
- Que Nginx está activo.
- Que las reglas FORWARD permiten el tráfico.

Podemos comprobar Nginx desde Ubuntu:

```bash
curl -I http://192.168.6.100
```

### 10.6. Problemas con iptables y nftables

En Ubuntu 22.04 y versiones posteriores, `iptables` suele funcionar mediante el backend `nf_tables`, que utiliza la infraestructura de nftables.

Podemos comprobar qué implementación tenemos:

```bash
sudo iptables --version
```

Si aparece `nf_tables`, estamos utilizando el backend nftables.

Si aparecen problemas con reglas antiguas, puede ser necesario revisar si el sistema utiliza `iptables-nft` o `iptables-legacy`. No debemos cambiar de backend sin comprobar antes las reglas existentes, ya que ambos pueden mantener configuraciones diferentes.

## 11. Automatización mediante scripts

Para facilitar la instalación, podemos guardar los siguientes scripts dentro de una carpeta `scripts/`.

Los scripts automatizan la configuración básica, pero es importante revisar las interfaces y las direcciones IP antes de ejecutarlos.

### 11.1. Script de Ubuntu Server

Creamos el archivo:

```bash
nano setup-ub-server.sh
```

Contenido:

```bash
#!/bin/bash
set -e

# Comprobar permisos
if [ "$EUID" -ne 0 ]; then
    echo "Ejecuta el script con sudo"
    exit 1
fi

echo "Actualizando paquetes..."
apt update

echo "Instalando servidor DHCP e iptables..."
apt install -y isc-dhcp-server iptables iptables-persistent

echo "Configurando Netplan..."
cat > /etc/netplan/99-lab.yaml <<'EOF'
network:
  version: 2
  ethernets:
    enp0s3:
      dhcp4: true
    enp0s8:
      dhcp4: false
      addresses:
        - 192.168.6.25/24
EOF

chmod 600 /etc/netplan/99-lab.yaml
netplan apply

echo "Configurando interfaz DHCP..."
sed -i 's/^INTERFACESv4=.*/INTERFACESv4="enp0s8"/' /etc/default/isc-dhcp-server

echo "Configurando DHCP..."
cat > /etc/dhcp/dhcpd.conf <<'EOF'
default-lease-time 600;
max-lease-time 7200;

authoritative;

subnet 192.168.6.0 netmask 255.255.255.0 {
    range 192.168.6.100 192.168.6.200;
    option routers 192.168.6.25;
    option domain-name-servers 8.8.8.8, 1.1.1.1;
}
EOF

dhcpd -t

echo "Activando forwarding..."
cat > /etc/sysctl.d/99-forward.conf <<'EOF'
net.ipv4.ip_forward=1
EOF

sysctl --system

echo "Configurando NAT..."
iptables -t nat -C POSTROUTING -o enp0s3 -j MASQUERADE 2>/dev/null || \
iptables -t nat -A POSTROUTING -o enp0s3 -j MASQUERADE

iptables -C FORWARD -i enp0s8 -o enp0s3 -j ACCEPT 2>/dev/null || \
iptables -A FORWARD -i enp0s8 -o enp0s3 -j ACCEPT

iptables -C FORWARD -i enp0s3 -o enp0s8 -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT 2>/dev/null || \
iptables -A FORWARD -i enp0s3 -o enp0s8 -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT

echo "Guardando reglas..."
netfilter-persistent save

echo "Activando DHCP..."
systemctl enable --now isc-dhcp-server

echo "Configuración de Ubuntu completada."
```

Damos permisos y ejecutamos:

```bash
chmod +x setup-ub-server.sh
sudo ./setup-ub-server.sh
```

**Importante:** este script reemplaza el archivo `/etc/netplan/99-lab.yaml` y el contenido de `/etc/dhcp/dhcpd.conf`. Debemos revisar previamente si existen configuraciones que queramos conservar.

### 11.2. Script de Debian 13

Creamos:

```bash
nano setup-debian.sh
```

Contenido:

```bash
#!/bin/bash
set -e

if [ "$EUID" -ne 0 ]; then
    echo "Ejecuta el script con sudo"
    exit 1
fi

echo "Actualizando paquetes..."
apt update

echo "Instalando Nginx..."
apt install -y nginx

echo "Activando Nginx..."
systemctl enable --now nginx

echo "Comprobando servicio..."
systemctl status nginx --no-pager

echo "Servidor web instalado correctamente."
echo "Comprueba la interfaz enp0s8 y su configuración DHCP por separado."
```

Damos permisos y ejecutamos:

```bash
chmod +x setup-debian.sh
sudo ./setup-debian.sh
```

Este segundo script instala y activa Nginx. La configuración DHCP de Debian se mantiene como paso separado para no sobrescribir innecesariamente la configuración de red existente.

## 12. Estructura recomendada del repositorio

Una posible organización para GitHub sería:

```text
laboratorio-dhcp-nat-nginx/
│
├── README.md
│
├── scripts/
│   ├── setup-ub-server.sh
│   └── setup-debian.sh
│
├── LICENSE
│
└── screenshots/
    ├── ubuntu-dhcp.png
    ├── debian-ip.png
    ├── nginx.png
    └── acceso-web.png
```

Las capturas pueden demostrar el funcionamiento del servidor DHCP, la dirección IP recibida por Debian, el servicio Nginx y el acceso desde el ordenador físico.

## 13. Licencia

Si queremos publicar el proyecto con una licencia MIT, podemos crear un archivo `LICENSE` con el siguiente contenido, sustituyendo el año y el autor:

```text
MIT License

Copyright (c) 2026 [Autor]

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT.
```

## 14. Cómo contribuir

Si quieres mejorar este laboratorio:

1. Haz un fork del repositorio.
2. Crea una rama para tus cambios.
3. Realiza las modificaciones.
4. Comprueba que la configuración funciona en VirtualBox.
5. Envía un Pull Request explicando los cambios.

## 15. Resultado final

Al completar el laboratorio tendremos una infraestructura donde:

- Ubuntu Server proporciona direcciones IP mediante DHCP.
- Ubuntu actúa como router y permite la salida a Internet mediante NAT.
- Debian 13 recibe su configuración de red automáticamente.
- Debian ejecuta un servidor web Nginx.
- Ubuntu redirige las conexiones HTTP mediante DNAT.
- VirtualBox permite acceder al servidor web desde el ordenador físico mediante el puerto `8080`.

De esta forma, hemos montado una infraestructura básica con servidor DHCP, enrutamiento, NAT, DNAT y alojamiento web, utilizando dos máquinas virtuales.

## 16. Herramientas utilizadas

Para la realización de este laboratorio se han utilizado las siguientes herramientas:

- **Oracle VirtualBox:** creación y gestión de las máquinas virtuales.
- **SSH:** conexión remota a los servidores para ejecutar comandos y realizar configuraciones de forma más rápida.
- **ChatGPT (OpenAI) y DeepSeek:** herramientas de inteligencia artificial utilizadas como apoyo para resolver dudas técnicas, revisar configuraciones y mejorar la documentación.

La configuración y las comprobaciones del laboratorio se han realizado de forma práctica en las máquinas virtuales.
