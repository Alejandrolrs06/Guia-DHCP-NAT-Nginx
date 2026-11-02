# 🚀 Laboratorio: Servidor DHCP + Router NAT + Servidor Web Nginx

Laboratorio de redes virtualizadas con **VirtualBox**, utilizando Ubuntu Server como servidor DHCP y router NAT, y Debian 13 como cliente y servidor web mediante Nginx.

## 📋 Índice

- [1. Arquitectura del laboratorio](#1-arquitectura-del-laboratorio)
- [2. Preparación en VirtualBox](#2-preparación-en-virtualbox)
- [3. Configuración del servidor DHCP](#3-configuración-del-servidor-dhcp)
- [4. Configuración del cliente Debian 13](#4-configuración-del-cliente-debian-13)
- [5. Configuración del router NAT](#5-configuración-del-router-nat)
- [6. Instalación de Nginx](#6-instalación-de-nginx)
- [7. Publicación de la web mediante DNAT](#7-publicación-de-la-web-mediante-dnat)
- [8. Persistencia de las reglas iptables](#8-persistencia-de-las-reglas-iptables)
- [9. Comprobaciones finales](#9-comprobaciones-finales)
- [10. Problemas habituales](#10-problemas-habituales)

---

## 1. Arquitectura del laboratorio

El objetivo es crear una pequeña infraestructura de red donde Ubuntu Server se encarga de repartir direcciones IP y actuar como router para proporcionar Internet al Debian, además de permitir acceder desde el PC real al servidor web Nginx.

### Diagrama de red

```text
┌─────────────────┐         ┌──────────────────┐         ┌─────────────────┐
│   PC Real       │         │   ub-server      │         │   Debian 13     │
│                 │         │                  │         │                 │
│  127.0.0.1:8080 ├────────►│ enp0s3 (NAT)     │         │                 │
│                 │         │ 10.0.2.15        │         │                 │
│                 │         │                  │         │                 │
│                 │         │ enp0s8 (intnet)  ├────────►│ enp0s8 (intnet) │
│                 │         │ 192.168.6.25     │         │ 192.168.6.100+  │
└─────────────────┘         └──────────────────┘         └─────────────────┘
                                    │                            │
                              DHCP + NAT                    Nginx :80
                              + DNAT :80
```

### Direccionamiento IP

| Máquina | Interfaz | Dirección IP | Función |
|---|---|---|---|
| PC Real | Localhost | 127.0.0.1 | Acceso a la web |
| ub-server | enp0s3 | 10.0.2.15 | Conexión NAT |
| ub-server | enp0s8 | 192.168.6.25 | DHCP y puerta de enlace |
| Debian 13 | enp0s8 | 192.168.6.100-200 | Cliente DHCP |

**Funcionamiento general:**

- Ubuntu Server reparte direcciones IP mediante DHCP.
- Debian obtiene su dirección IP automáticamente.
- Ubuntu permite que Debian tenga Internet mediante NAT.
- Debian ejecuta Nginx en el puerto 80.
- Ubuntu redirige las conexiones externas hacia Nginx mediante DNAT.
- VirtualBox permite acceder desde el PC real utilizando el puerto 8080.

---

## 2. Preparación en VirtualBox

Antes de configurar los sistemas operativos debemos preparar los adaptadores de red de las dos máquinas virtuales.

### Configuración de los adaptadores

| Máquina | Adaptador 1 | Adaptador 2 |
|---|---|---|
| ub-server | NAT | Red Interna |
| Debian 13 | Red Interna | — |

En la red interna utilizaremos el nombre `intnet` para que ambas máquinas puedan comunicarse.

**¿Por qué utilizamos dos tipos de red?**

- NAT: proporciona Internet al Ubuntu Server mediante VirtualBox.
- Red Interna: permite la comunicación entre Ubuntu y Debian sin utilizar la red física del ordenador.

De esta manera Ubuntu tendrá dos interfaces, una para Internet y otra para la red interna.

Arrancamos ambas máquinas y continuamos con la configuración.

---

## 3. Configuración del servidor DHCP

En esta fase configuramos Ubuntu Server para que actúe como servidor DHCP y reparta direcciones IP a los dispositivos conectados a la red interna.

### 3.1. Configurar una IP estática

Primero necesitamos que Ubuntu tenga una dirección IP fija en la red interna, ya que será la puerta de enlace para los clientes.

Editamos la configuración de Netplan:

```bash
sudo nano /etc/netplan/50-cloud-init.yaml
```

Configuramos las interfaces de la siguiente manera:

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

Comprobamos la dirección IP:

```bash
ip -4 addr show enp0s8
```

La interfaz interna debe tener la dirección `192.168.6.25/24`.

### 3.2. Instalar el servidor DHCP

Instalamos el servicio que permitirá repartir direcciones IP automáticamente:

```bash
sudo apt update
sudo apt install isc-dhcp-server -y
```

### 3.3. Seleccionar la interfaz DHCP

Indicamos al servicio que debe escuchar las solicitudes DHCP únicamente por la interfaz interna.

```bash
sudo nano /etc/default/isc-dhcp-server
```

Modificamos la siguiente línea:

```conf
INTERFACESv4="enp0s8"
```

### 3.4. Configurar el rango de direcciones IP

Editamos el archivo principal del servidor DHCP:

```bash
sudo nano /etc/dhcp/dhcpd.conf
```

Configuramos la subred y el rango de direcciones:

```conf
default-lease-time 600;
max-lease-time 7200;
authoritative;

subnet 192.168.6.0 netmask 255.255.255.0 {
    range 192.168.6.100 192.168.6.200;
    option routers 192.168.6.25;
    option domain-name-servers 8.8.8.8, 1.1.1.1;
    option subnet-mask 255.255.255.0;
}
```

**Explicación de la configuración:**

- `default-lease-time`: tiempo predeterminado de concesión de una IP, en este caso 600 segundos.
- `max-lease-time`: tiempo máximo de concesión, 7200 segundos.
- `authoritative`: indica que este servidor es la autoridad DHCP de esta subred.
- `range`: establece las direcciones IP que se pueden repartir.
- `option routers`: indica la puerta de enlace de los clientes.
- `option domain-name-servers`: establece los servidores DNS.

### 3.5. Arrancar el servicio

Reiniciamos y habilitamos el servidor DHCP:

```bash
sudo systemctl restart isc-dhcp-server
sudo systemctl enable isc-dhcp-server
```

Comprobamos su estado:

```bash
sudo systemctl status isc-dhcp-server
```

Si aparece algún error podemos consultar los registros:

```bash
sudo journalctl -u isc-dhcp-server -e
```

Con esto Ubuntu ya puede repartir direcciones IP a los dispositivos de la red interna.

---

## 4. Configuración del cliente Debian 13

Ahora configuramos Debian para que solicite automáticamente una dirección IP al servidor DHCP de Ubuntu.

### 4.1. Configurar la interfaz de red

Editamos el archivo de configuración:

```bash
sudo nano /etc/network/interfaces
```

Dejamos la interfaz interna configurada de esta manera:

```conf
auto enp0s8
iface enp0s8 inet dhcp
```

Eliminamos cualquier configuración estática anterior de esa interfaz.

### 4.2. Reiniciar la red

Aplicamos los cambios:

```bash
sudo systemctl restart networking
```

Comprobamos la dirección IP recibida:

```bash
ip -4 addr show enp0s8
```

Debian debe obtener una dirección dentro del rango:

```text
192.168.6.100 - 192.168.6.200
```

También podemos comprobar las concesiones desde Ubuntu:

```bash
sudo cat /var/lib/dhcp/dhcpd.leases
```

**Resultado:** Debian ya obtiene una dirección IP automáticamente sin necesidad de configurarla manualmente.

---

## 5. Configuración del router NAT

En esta fase convertimos Ubuntu Server en un router para que Debian pueda acceder a Internet utilizando la conexión NAT de Ubuntu.

### 5.1. Habilitar el reenvío de paquetes

Por defecto Linux no reenvía paquetes entre interfaces, por lo que debemos habilitar esta función.

Ejecutamos:

```bash
sudo sysctl -w net.ipv4.ip_forward=1
```

Para mantenerlo después de reiniciar creamos un archivo:

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

### 5.2. Configurar las reglas NAT con iptables

Primero limpiamos las reglas anteriores de NAT y de la cadena FORWARD, teniendo en cuenta que esto elimina también otras reglas que pudieran existir en esas cadenas.

```bash
sudo iptables -t nat -F
sudo iptables -F FORWARD
```

Configuramos el enmascaramiento para permitir que Debian salga a Internet utilizando la conexión de Ubuntu:

```bash
sudo iptables -t nat -A POSTROUTING -o enp0s3 -j MASQUERADE
```

Permitimos el tráfico desde la red interna hacia Internet:

```bash
sudo iptables -A FORWARD -i enp0s8 -o enp0s3 -j ACCEPT
```

Permitimos que vuelvan las respuestas de las conexiones establecidas:

```bash
sudo iptables -A FORWARD -i enp0s3 -o enp0s8 -m state --state RELATED,ESTABLISHED -j ACCEPT
```

**¿Qué conseguimos?**

El Debian utiliza Ubuntu como puerta de enlace y Ubuntu traduce las direcciones privadas mediante MASQUERADE para que puedan salir a Internet.

### 5.3. Comprobar la conexión

Desde Debian ejecutamos:

```bash
ping 8.8.8.8
```

Si responde correctamente significa que existe conectividad con Internet.

Si tenemos problemas de resolución DNS podemos comprobar la configuración de `/etc/resolv.conf`:

```bash
sudo nano /etc/resolv.conf
```

Añadimos:

```conf
nameserver 8.8.8.8
nameserver 1.1.1.1
```

---

## 6. Instalación de Nginx

Ahora instalamos el servidor web en Debian 13 para poder publicar una página accesible desde el PC real.

Actualizamos los paquetes:

```bash
sudo apt update
```

Instalamos Nginx:

```bash
sudo apt install nginx -y
```

Habilitamos el servicio y lo arrancamos:

```bash
sudo systemctl enable nginx
sudo systemctl start nginx
```

Comprobamos que funciona:

```bash
curl -I http://localhost
```

Si todo está correcto debería aparecer una respuesta similar a:

```text
HTTP/1.1 200 OK
```

Esto significa que Nginx está funcionando y atendiendo peticiones por el puerto 80.

---

## 7. Publicación de la web mediante DNAT

Hasta ahora Nginx solamente es accesible desde Debian o desde los dispositivos de su red interna, por lo que necesitamos configurar una redirección para poder acceder desde el PC real.

### 7.1. Configurar DNAT en Ubuntu

Primero necesitamos conocer la dirección IP que ha recibido Debian, en nuestro caso utilizamos `192.168.6.100`.

En Ubuntu ejecutamos:

```bash
sudo iptables -t nat -A PREROUTING -i enp0s3 -p tcp --dport 80 -j DNAT --to-destination 192.168.6.100:80
```

Esta regla modifica el destino de las conexiones que llegan al puerto 80 de Ubuntu y las redirige al servidor web de Debian.

Permitimos el tráfico hacia Debian:

```bash
sudo iptables -A FORWARD -p tcp -d 192.168.6.100 --dport 80 -m state --state NEW,ESTABLISHED,RELATED -j ACCEPT
```

### 7.2. Configurar el reenvío de puertos en VirtualBox

Ahora tenemos que permitir que el PC real pueda llegar al puerto 80 de Ubuntu.

Entramos en la configuración de VirtualBox:

**ub-server → Red → Adaptador 1 → NAT → Opciones avanzadas → Reenvío de puertos**

Creamos la siguiente regla:

| Campo | Valor |
|---|---|
| Nombre | NGINX |
| IP anfitrión | Dejar vacío |
| Puerto anfitrión | 8080 |
| IP invitado | 10.0.2.15 |
| Puerto invitado | 80 |

Esta regla permite que VirtualBox redirija las conexiones del PC real hacia Ubuntu Server.

**El recorrido de la conexión es el siguiente:**

1. El usuario accede a `127.0.0.1:8080` desde su navegador.
2. VirtualBox redirige la conexión a `10.0.2.15:80`.
3. Ubuntu recibe la conexión por `enp0s3`.
4. La regla DNAT redirige el tráfico a `192.168.6.100:80`.
5. Debian recibe la petición y Nginx devuelve la página web.

---

## 8. Persistencia de las reglas iptables

Las reglas de iptables se almacenan en memoria y desaparecen cuando reiniciamos Ubuntu, por lo que necesitamos guardarlas para que se recuperen automáticamente.

### 8.1. Instalar iptables-persistent

En Ubuntu ejecutamos:

```bash
sudo apt install iptables-persistent -y
```

Durante la instalación aceptamos guardar las reglas actuales de IPv4 e IPv6.

### 8.2. Guardar las reglas

Ejecutamos:

```bash
sudo netfilter-persistent save
```

Las reglas IPv4 se guardan normalmente en:

```text
/etc/iptables/rules.v4
```

De esta manera las reglas NAT y DNAT se restaurarán automáticamente cuando Ubuntu vuelva a arrancar.

---

## 9. Comprobaciones finales

Una vez terminadas todas las configuraciones reiniciamos ambas máquinas virtuales para comprobar que todo sigue funcionando.

### 9.1. Comprobar la IP de Debian

En Debian ejecutamos:

```bash
ip a
```

Comprobamos que la interfaz `enp0s8` tiene una dirección del rango DHCP.

### 9.2. Comprobar el acceso a Internet

Desde Debian:

```bash
ping 8.8.8.8
```

Si recibimos respuestas significa que el router NAT funciona correctamente.

### 9.3. Comprobar Nginx

En Debian:

```bash
sudo systemctl status nginx
```

También podemos comprobar su respuesta:

```bash
curl -I http://localhost
```

### 9.4. Comprobar desde el PC real

Abrimos el navegador del ordenador principal y accedemos a:

```text
http://127.0.0.1:8080
```

Si aparece la página de bienvenida de Nginx significa que toda la cadena de redirecciones funciona correctamente.

---

## 10. Problemas habituales

### Debian no recibe una dirección IP

Comprobar que Ubuntu tiene la IP `192.168.6.25` en `enp0s8` y que el servicio DHCP está funcionando.

```bash
sudo systemctl status isc-dhcp-server
```

También comprobar que los dos adaptadores están conectados a la misma red interna `intnet`.

### Debian tiene IP pero no tiene Internet

Comprobar que el reenvío IPv4 está habilitado:

```bash
sysctl net.ipv4.ip_forward
```

El resultado debe ser:

```text
net.ipv4.ip_forward = 1
```

Revisar también las reglas NAT:

```bash
sudo iptables -t nat -L -n -v
sudo iptables -L FORWARD -n -v
```

### Nginx no carga desde el PC real

Comprobar que Nginx está funcionando en Debian y que las reglas DNAT están configuradas correctamente.

```bash
sudo systemctl status nginx
sudo iptables -t nat -L PREROUTING -n -v
```

Revisar también el reenvío de puertos de VirtualBox.

### Debian cambia de dirección IP

Si Debian recibe una dirección diferente a `192.168.6.100`, hay que actualizar la regla DNAT para utilizar la nueva dirección.

Otra opción es crear una reserva DHCP para que Debian siempre reciba la misma IP.

En Ubuntu editamos:

```bash
sudo nano /etc/dhcp/dhcpd.conf
```

Añadimos al final:

```conf
host debian {
    hardware ethernet XX:XX:XX:XX:XX:XX;
    fixed-address 192.168.6.100;
}
```

Sustituimos la MAC por la dirección real de `enp0s8` de Debian.

Podemos consultarla con:

```bash
ip link show enp0s8
```

Reiniciamos DHCP:

```bash
sudo systemctl restart isc-dhcp-server
```

### Las reglas iptables desaparecen al reiniciar

Comprobamos que `iptables-persistent` está instalado y guardamos las reglas actuales:

```bash
sudo netfilter-persistent save
```

---

## Resultado final

Al terminar el laboratorio tenemos una infraestructura funcional donde Ubuntu Server actúa como servidor DHCP y router NAT, mientras que Debian 13 obtiene su IP automáticamente y ejecuta Nginx como servidor web.

Además hemos configurado DNAT y el reenvío de puertos de VirtualBox para poder acceder a la página web desde el PC real mediante `http://127.0.0.1:8080`.

El laboratorio permite practicar la configuración de servicios de red, el direccionamiento IP, el enrutamiento, NAT, DNAT y la publicación de servicios web en un entorno virtualizado.
