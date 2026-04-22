# 1. Qué se está construyendo realmente (visión de arquitectura)

Este proyecto no es “solo instalar GNS3 en una EC2”. Lo que realmente se está construyendo es un **entorno de simulación de red avanzada** donde:

* La **instancia EC2 en AWS actúa como un nodo de red**
* GNS3 se usa como **orquestador de topologías virtuales**
* Las redes virtuales internas (LAN y DMZ) se conectan al mundo real mediante **interfaces TAP**
* El tráfico sale a Internet a través de la propia EC2 mediante **NAT y routing del sistema Linux**

En otras palabras:

> La EC2 no es solo un servidor → se comporta como un **router virtual con capacidades de firewall y NAT** dentro del laboratorio GNS3.

---

# 2. Elementos clave del diseño

## 2.1 EC2 como “router físico virtual”

La instancia EC2 hace de:

* Nodo de GNS3 Server
* Punto de salida a Internet
* Router entre redes virtuales (LAN / DMZ)
* NAT gateway del laboratorio

Por eso es importante entender que **la EC2 no es solo hosting, es parte de la topología de red simulada**.

---

## 2.2 GNS3 Server en EC2

Se instala con:

```bash
sudo apt install gns3-server
```

Esto permite que:

* La GUI de GNS3 (en local o web) controle la topología
* La EC2 ejecute los dispositivos virtuales
* Se centralice toda la simulación en AWS

El servicio systemd:

* Garantiza persistencia
* Reinicio automático
* Ejecución en background

---

## 2.3 Interfaces TAP (punto más importante del diseño)

Aquí está el núcleo del sistema.

Se crean:

* `tap-lan` → red LAN virtual
* `tap-dmz` → red DMZ virtual

```bash
ip tuntap add dev tap-lan mode tap
ip tuntap add dev tap-dmz mode tap
```

Y se les asignan redes:

* LAN → `10.10.10.0/24`
* DMZ → `192.168.101.0/24`

## 🔴 Qué significa esto a nivel de ingeniería

Una interfaz TAP es:

> Una interfaz de red virtual de capa 2 que el sistema operativo trata como una tarjeta Ethernet real.

Esto permite que:

* GNS3 conecte routers/switches virtuales
* Linux vea esas redes como interfaces físicas
* Se pueda enrutar tráfico entre mundos virtuales y real

---

# 3. GNS3 CLOUD (concepto clave del proyecto)

En GNS3, el elemento **CLOUD** representa:

> Un puente entre la topología virtual y el sistema operativo real.

En este caso:

* CLOUD = EC2 Linux host
* CLOUD se conecta a `tap-lan` y `tap-dmz`

## 🔴 Importante (punto que pedías destacar)

Este diseño usa CLOUD, y eso implica:

> NO hay routing automático entre redes virtuales y salida a Internet.

Por eso:

* Hay que configurar **routing manual**
* Hay que configurar **NAT explícito**
* Hay que habilitar **IP forwarding**

---

# 4. Routing e IP Forwarding

Se habilita:

```bash
sysctl -w net.ipv4.ip_forward=1
```

## Qué significa esto

Por defecto Linux actúa como:

> Host final, no como router

Al activar IP forwarding:

* La EC2 empieza a comportarse como router L3
* Puede reenviar paquetes entre interfaces:

  * tap-lan → ens5 (Internet)
  * tap-dmz → ens5

---

# 5. NAT (Masquerading)

Se configura:

```bash
iptables -t nat -A POSTROUTING -o ens5 -j MASQUERADE
```

## Qué hace esto realmente

Esto traduce:

* IPs privadas (10.10.10.0 / 192.168.101.0)
* A la IP pública de la EC2

## Sin NAT ocurriría esto:

* Las redes LAN/DMZ NO podrían salir a Internet
* AWS descartaría tráfico con IPs privadas

---

## Reglas FORWARD

```bash
iptables -A FORWARD -i tap-lan -o ens5 -j ACCEPT
iptables -A FORWARD -i tap-dmz -o ens5 -j ACCEPT
```

Esto permite:

* Tráfico desde redes virtuales hacia Internet
* Control del flujo de paquetes entre interfaces

---

# 6. Docker en la EC2

Se instala Docker para:

* Simular hosts dentro del laboratorio
* Levantar servicios en LAN o DMZ
* Probar arquitecturas reales (web, API, etc.)

Ejemplo típico:

* Web server en DMZ (Docker container)
* Cliente en LAN
* EC2 como router/firewall

---

# 7. Importante: por qué este diseño usa CLOUD + TAP + NAT

Este es el punto más conceptual.

## 🔴 Uso de CLOUD en GNS3 implica:

* No hay conectividad automática
* No hay “router virtual interno” entre redes
* El sistema operativo actúa como puente

Por eso necesitas:

### 1. TAP

Para crear interfaces de red virtual reales

### 2. Routing

Para permitir forwarding entre interfaces

### 3. NAT

Para salida a Internet desde redes privadas

---

# 8. Alternativa: se podría haber hecho SIN CLOUD

## ✔ Opción alternativa 1: usar NAT node de GNS3

En lugar de CLOUD:

* GNS3 tiene un nodo NAT integrado
* Proporciona salida a Internet automáticamente
* No requiere iptables manual

### Ventajas:

* Más simple
* Menos configuración del sistema operativo
* Menos errores de routing

### Desventajas:

* Menos control
* Menos realismo de infraestructura

---

## ✔ Opción alternativa 2: NAT directo en GNS3 sin TAP

Se podría haber hecho:

* NAT node
* Switches internos GNS3
* Sin tocar Linux networking

---

## ✔ Opción alternativa 3: CLOUD pero con bridge en lugar de TAP

En vez de TAP:

* bridge (br0)
* interfaces físicas o virtuales bridged

---


