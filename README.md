# GS3_AWS
Creación de servidor GNS3 en una EC2
En la configuración base se pone como ejemplo una topología de red formada por un router la propia instacia EC2 y dos redes. La red lan y la red dmz. 
Desde GNS3 en la gui WEB se conectan las redes a las interfaces tap correspondientes en la CLOUD de gns3.
## Tutorial breve: EC2 + GNS3 + TAP + NAT + Docker (Ubuntu 24.04.4)

---

## 1. Crear EC2. Realizado en ubuntu 24.04.4

Security Groups:

* 22
* 80
* 443
* 3080
* 500-600

---

## 2. Instalar GNS3 Server

```bash
sudo add-apt-repository ppa:gns3/ppa -y
sudo apt update
sudo apt upgrade -y
sudo apt install gns3-server
```

---

## 3. Servicio systemd de GNS3

```bash
sudo nano /etc/systemd/system/gns3server.service
```

```ini
[Unit]
Description=GNS3 Server
After=network.target

[Service]
User=ubuntu
ExecStart=/usr/bin/gns3server
Restart=always
WorkingDirectory=/home/ubuntu

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable gns3server
sudo systemctl start gns3server
```

---

## 4. Crear interfaces TAP

```bash
sudo nano /usr/local/sbin/gns3-taps.sh
```

```bash
#!/bin/bash

ip tuntap add dev tap-lan mode tap 2>/dev/null
ip tuntap add dev tap-dmz mode tap 2>/dev/null

ip addr add 10.10.10.1/24 dev tap-lan
ip addr add 192.168.101.1/24 dev tap-dmz

ip link set tap-lan up
ip link set tap-dmz up
```

```bash
sudo chmod +x /usr/local/sbin/gns3-taps.sh
```

```bash
sudo nano /etc/systemd/system/gns3-taps.service
```

```ini
[Unit]
Description=GNS3 TAP interfaces (LAN & DMZ)
After=network.target

[Service]
Type=oneshot
ExecStart=/usr/local/sbin/gns3-taps.sh
RemainAfterExit=true

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable gns3-taps.service
sudo systemctl start gns3-taps.service
```

Desde la gui de gns3 si las Taps no aparecen hay que crearlas

---

## 5. NAT + routing

```bash
sudo nano /etc/systemd/system/gns3-nat.service
```

```ini
[Unit]
Description=NAT para GNS3 TAP
After=network.target

[Service]
Type=oneshot
ExecStart=/sbin/sysctl -w net.ipv4.ip_forward=1
ExecStart=/sbin/iptables -t nat -A POSTROUTING -o ens5 -j MASQUERADE
ExecStart=/sbin/iptables -A FORWARD -i tap-lan -o ens5 -j ACCEPT
ExecStart=/sbin/iptables -A FORWARD -i tap-dmz -o ens5 -j ACCEPT
RemainAfterExit=true

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl enable gns3-nat
sudo systemctl start gns3-nat
```

---

## 6. Instalar Docker

```bash
sudo apt install -y ca-certificates curl gnupg
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo tee /etc/apt/keyrings/docker.asc > /dev/null
sudo chmod a+r /etc/apt/keyrings/docker.asc
```

```bash
echo \
"deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
$(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

```bash
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

```bash
sudo systemctl enable docker
sudo systemctl start docker
sudo usermod -aG docker $USER
```

---

## 7. Final

Salir y volver a entrar en la sesión.
