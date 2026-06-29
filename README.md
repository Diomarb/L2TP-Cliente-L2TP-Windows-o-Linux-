# L2TP-Cliente-L2TP-Windows-o-Linux-
Configure una VPN client to site punto a multipunto IPSec IKEv1 con L2TP

Link Youtube: https://youtu.be/pEo2SG3AI_g


# VPN Client-to-Site — L2TP/IPSec

**Autor:** Eunice F. Fleming | **Matrícula:** 2024-1185 | **Institución:** ITLA

---

## 1. Objetivo

Implementar una VPN Client-to-Site punto a multipunto utilizando el protocolo **L2TP (Layer 2 Tunneling Protocol)** encapsulado sobre **IPSec IKEv1**. El cliente es una máquina Kali Linux que se conecta al servidor LNS (R2) a través de internet, atravesando un router NAT (R1). Una vez establecida la conexión, Kali recibe una IP virtual del pool del servidor y puede acceder a la LAN remota de R2 como si estuviera conectado localmente.

---

## 2. Topología de Red

```
                        Red 10.11.85.0/24
                                
  [Kali Linux]                              [PC1]
   eth0: 10.11.85.2/26                  10.11.85.130/26
        |                                      |
   [Switch]                                   [R2 - LNS]
        |                                  e0/1: 10.11.85.129/26
       [R1 - NAT]         [ISP]            e0/0: 10.11.85.69/30
   e0/0: 10.11.85.1/26  -------      -----/
   e0/1: 10.11.85.66/30 ------ e0/0: 10.11.85.65/30
                                e0/1: 10.11.85.70/30
```
<img width="305" height="203" alt="Captura de pantalla 2026-06-27 005618" src="https://github.com/user-attachments/assets/528a173a-b62f-42d5-80a9-ac82db8f49a7" />


### Roles de cada dispositivo

| Dispositivo | Rol         | Descripción                                                      |
|-------------|-------------|------------------------------------------------------------------|
| Kali Linux  | Cliente VPN | Inicia la conexión L2TP/IPSec desde la LAN de R1                |
| R1          | NAT Gateway | Da salida a internet al cliente Kali. **NO** termina el túnel VPN |
| ISP         | Tránsito    | Solo rutea. No participa en el túnel                            |
| R2 (LNS)   | Servidor VPN| Termina el túnel L2TP/IPSec, autentica al cliente y asigna IP del pool |
| PC1         | Host destino| Host en la LAN remota a la que el cliente VPN quiere acceder    |

### Direccionamiento IP

| Dispositivo  | Interfaz | Dirección IP          | Descripción            |
|-------------|----------|-----------------------|------------------------|
| R1          | e0/0     | 10.11.85.1/26         | LAN → Switch/Kali      |
| R1          | e0/1     | 10.11.85.66/30        | WAN → ISP              |
| ISP         | e0/0     | 10.11.85.65/30        | Hacia R1               |
| ISP         | e0/1     | 10.11.85.70/30        | Hacia R2               |
| R2          | e0/0     | 10.11.85.69/30        | WAN → ISP              |
| R2          | e0/1     | 10.11.85.129/26       | LAN → PC1              |
| PC1         | e0       | 10.11.85.130/26       | GW: 10.11.85.129       |
| Kali        | eth0     | 10.11.85.2/26         | GW: 10.11.85.1 (R1)    |
| Kali (VPN)  | ppp0     | 10.11.85.202/32       | IP virtual asignada por R2 |

---

## 3. Parámetros

### IPSec IKEv1 — Fase 1 (ISAKMP)

| Parámetro       | Valor                                           |
|-----------------|-------------------------------------------------|
| Política        | 10                                              |
| Cifrado         | AES-256                                         |
| Hash            | SHA-256                                         |
| Autenticación   | Pre-Shared Key (wildcard `0.0.0.0 0.0.0.0`)    |
| Grupo DH        | 14 (2048-bit)                                   |
| Lifetime        | 86400 segundos                                  |
| Clave PSK       | `ITLA2024`                                      |
| NAT-T           | Habilitado — UDP 4500 (`isakmp nat keepalive 20`) |

### IPSec — Fase 2 (Transform Set)

| Parámetro     | Valor                                               |
|---------------|-----------------------------------------------------|
| Transform Set | `TS-L2TP`                                           |
| Protocolo     | ESP                                                 |
| Cifrado       | AES-256                                             |
| Integridad    | HMAC-SHA-256                                        |
| Modo          | Transport (L2TP agrega su propio encapsulado)       |
| Crypto Map    | `CM-L2TP` → dynamic `DYN-MAP`                      |

### L2TP / PPP

| Parámetro        | Valor                        |
|------------------|------------------------------|
| Protocolo túnel  | L2TP (`vpdn-group L2TP-SERVER`) |
| Template virtual | Virtual-Template 1           |
| Pool de IPs      | 10.11.85.193 – 10.11.85.206 (/28) |
| Autenticación PPP| MS-CHAP v2                   |
| Cifrado PPP      | MPPE auto                    |
| Usuario          | `kali`                       |
| Contraseña       | `ITLA2024-VPN`               |

---

## 4. Funcionamiento — L2TP/IPSec

L2TP/IPSec combina dos protocolos en capas. **IPSec** establece primero un canal cifrado seguro (Fase 1 y Fase 2), y dentro de ese canal IPSec, **L2TP** crea el túnel de capa 2 que permite negociar PPP para la asignación de direcciones IP al cliente.

### Flujo de negociación

| Fase | Protocolo          | Descripción                                                                         |
|------|--------------------|-------------------------------------------------------------------------------------|
| 1    | IKEv1 Main Mode    | Kali y R2 negocian parámetros ISAKMP y crean canal seguro (SA IKE). NAT-T activo por UDP 4500 |
| 2    | IKEv1 Quick Mode   | Negocian Transform Set IPSec en modo transport. SA IPSec establecida               |
| 3    | L2TP               | Dentro del canal IPSec, Kali abre túnel L2TP al LNS (R2) por UDP 1701             |
| 4    | PPP/MSCHAPv2       | Autenticación del usuario (`kali` / `ITLA2024-VPN`)                                |
| 5    | IPCP               | R2 asigna IP del pool a Kali. Interfaz `ppp0` se levanta en Kali                  |
| 6    | Datos              | Kali puede acceder a la LAN de R2 (`10.11.85.128/26`) via `ppp0`                  |

### Capas de encapsulación

```
Datos originales (IP: Kali → PC1)
 ↓ PPP encapsula
PPP frame
 ↓ L2TP encapsula
L2TP header + PPP frame (UDP 1701)
 ↓ IPSec ESP (modo transport) cifra el UDP L2TP
ESP header + L2TP cifrado (UDP 4500 por NAT-T)
 ↓ IP outer: Kali (NAT) → R2
IP outer: src 10.11.85.66 (R1 NAT) dst 10.11.85.67 (R2)
```

---

## 5. Scripts de Configuración

### ISP

```bash
enable
configure terminal
hostname ISP

interface Ethernet0/0
 description Hacia R1
 ip address 10.11.85.65 255.255.255.252
 no shutdown

interface Ethernet0/1
 description Hacia R2
 ip address 10.11.85.70 255.255.255.252
 no shutdown

ip route 10.11.85.0 255.255.255.192 10.11.85.66
ip route 10.11.85.128 255.255.255.192 10.11.85.67
ip route 10.11.85.192 255.255.255.240 10.11.85.67
end
```

### R1 — NAT Gateway

```bash
enable
configure terminal
hostname R1

interface Ethernet0/0
 description LAN hacia Switch y Kali
 ip address 10.11.85.1 255.255.255.192
 ip nat inside
 no shutdown

interface Ethernet0/1
 description WAN hacia ISP
 ip address 10.11.85.66 255.255.255.252
 ip nat outside
 no shutdown

ip route 0.0.0.0 0.0.0.0 10.11.85.65
access-list 1 permit 10.11.85.0 0.0.0.63
ip nat inside source list 1 interface Ethernet0/1 overload
end
```

### R2 — Servidor LNS

```bash
enable
configure terminal
hostname R2

interface Ethernet0/0
 description WAN hacia ISP
 ip address 10.11.85.69 255.255.255.252
 no shutdown

interface Ethernet0/1
 description LAN hacia PC1
 ip address 10.11.85.129 255.255.255.192
 no shutdown

ip route 0.0.0.0 0.0.0.0 10.11.85.70

aaa new-model
aaa authentication ppp VPN-AUTH local
aaa authorization network VPN-AUTH local
username kali password ITLA2024-VPN

ip local pool POOL-VPN 10.11.85.193 10.11.85.206

crypto isakmp policy 10
 encryption aes 256
 hash sha256
 authentication pre-share
 group 14
 lifetime 86400

crypto isakmp key ITLA2024 address 0.0.0.0
crypto isakmp identity address

crypto ipsec transform-set TS-L2TP esp-aes 256 esp-sha256-hmac
 mode transport

crypto dynamic-map DYN-MAP 10
 set transform-set TS-L2TP
 reverse-route
 set pfs group2
 exit

crypto map CMAP-L2TP 65535 ipsec-isakmp dynamic DYN-MAP

interface Ethernet0/0
 crypto map CMAP-L2TP
 exit

vpdn enable
vpdn-group L2TP-SERVER
 accept-dialin
  protocol l2tp
  virtual-template 1
 !
 no l2tp tunnel authentication

interface Virtual-Template1
 ip unnumbered Ethernet0/1
 peer default ip address pool POOL-VPN
 ppp authentication ms-chap-v2 VPN-AUTH
 ppp authorization VPN-AUTH
end
```

### VPCS — PC1

```bash
ip 10.11.85.130 255.255.255.192 10.11.85.129
save
```

### Kali Linux — Cliente VPN

```bash
echo "[*] Configurando eth0..."
sudo ip addr flush dev eth0
sudo ip addr add 10.11.85.2/26 dev eth0
sudo ip link set eth0 up
sudo ip route add 10.11.85.0/26 dev eth0 src 10.11.85.2
sudo ip route add default via 10.11.85.1 dev eth0
echo "[+] eth0: 10.11.85.2/26 | GW: 10.11.85.1"

echo "[*] Ping a R1 (gateway)..."
ping -c 2 -W 2 10.11.85.1 && echo "[+] OK -> R1" || echo "FALLO -> R1"

echo "[*] Ping a R2 (servidor VPN)..."
ping -c 2 -W 2 10.11.85.67 && echo "[+] OK -> R2" || echo "FALLO -> R2"

echo "[*] Instalando strongswan y xl2tpd..."
sudo apt-get update -y -qq
sudo apt-get install -y strongswan xl2tpd

# /etc/ipsec.conf
sudo bash -c 'cat > /etc/ipsec.conf << EOF
config setup
  charondebug="ike 1, knl 1, cfg 0"
  uniqueids=no

conn L2TP-PSK
  keyexchange=ikev1
  authby=secret
  type=transport
  left=%defaultroute
  leftprotoport=17/1701
  right=10.11.85.69
  rightprotoport=17/1701
  rightid=10.11.85.69
  ike=aes256-sha256-modp2048!
  esp=aes256-sha256-modp2048!
  pfs=yes
  forceencaps=yes
  dpdaction=clear
  dpddelay=30s
  dpdtimeout=120s
  auto=add
EOF'

# /etc/ipsec.secrets
sudo bash -c 'cat > /etc/ipsec.secrets << EOF
%any %any : PSK "ITLA2024"
EOF'
sudo chmod 600 /etc/ipsec.secrets

# /etc/xl2tpd/xl2tpd.conf
sudo bash -c 'cat > /etc/xl2tpd/xl2tpd.conf << EOF
[global]
port = 1701

[lac l2tp-vpn]
lns = 10.11.85.69
ppp debug = yes
pppoptfile = /etc/ppp/options.l2tpd.client
length bit = yes
EOF'

# /etc/ppp/options.l2tpd.client
sudo bash -c 'cat > /etc/ppp/options.l2tpd.client << EOF
ipcp-accept-local
ipcp-accept-remote
refuse-eap
require-mschap-v2
noccp
mtu 1400
mru 1400
noipdefault
defaultroute
usepeerdns
connect-delay 5000
name kali
password ITLA2024-VPN
EOF'

sudo mkdir -p /var/run/xl2tpd
sudo touch /var/run/xl2tpd/l2tp-control

echo "[*] Reiniciando servicios..."
sudo systemctl stop xl2tpd 2>/dev/null || true
sudo systemctl stop strongswan-starter 2>/dev/null || true
sleep 2
sudo systemctl start strongswan-starter
sleep 3
sudo systemctl start xl2tpd
sleep 2

sudo systemctl is-active strongswan-starter && echo "[+] strongswan: ACTIVO" || echo "[!] strongswan: FALLO"
sudo systemctl is-active xl2tpd && echo "[+] xl2tpd: ACTIVO" || echo "[!] xl2tpd: FALLO"

echo "[*] Levantando túnel IPSec hacia 10.11.85.69..."
sudo ipsec up L2TP-PSK
sleep 4

echo "[*] Estado IPSec:"
sudo ipsec status | grep -E "L2TP|ESTABLISHED|INSTALLED|ERROR" || echo "Revisar con: sudo ipsec statusall"

echo "[*] Iniciando sesión L2TP (usuario: kali)..."
sudo bash -c 'echo "c l2tp-vpn" > /var/run/xl2tpd/l2tp-control'

echo "[*] Esperando que ppp0 se levante (15 segundos)..."
for i in $(seq 1 15); do
  sleep 1
  if ip link show ppp0 &>/dev/null; then
    echo "[+] ppp0 detectada en ${i}s"
    break
  fi
  echo "... ${i}s"
done

if ip link show ppp0 &>/dev/null; then
  echo "[+] Agregando ruta hacia LAN de R2..."
  sudo ip route add 10.11.85.128/26 dev ppp0 2>/dev/null && \
    echo "[+] Ruta 10.11.85.128/26 via ppp0 OK" || \
    echo "[*] Ruta ya existente"
else
  echo "[!] ppp0 no se levantó. Ejecutar diagnóstico:"
  echo "  sudo journalctl -u strongswan-starter -n 40 --no-pager"
  echo "  sudo journalctl -u xl2tpd -n 40 --no-pager"
  exit 1
fi
```

---

## 6. Verificación

### En R2 (servidor)

```bash
show vpdn session
show vpdn tunnel
show crypto isakmp sa
show crypto ipsec sa
```
<img width="692" height="253" alt="image" src="https://github.com/user-attachments/assets/d5827a53-2e82-4a4d-a08b-d1529c82bf1b" />

<img width="975" height="127" alt="image" src="https://github.com/user-attachments/assets/bf10d772-927c-476e-b651-a0da00b047f7" />

<img width="883" height="206" alt="image" src="https://github.com/user-attachments/assets/d54d3058-c21b-4916-a9f4-f8909961cc93" />

<img width="975" height="686" alt="image" src="https://github.com/user-attachments/assets/debf2ea5-3cba-4bc4-9562-996930b1c832" />


### En Kali Linux (cliente)

```bash
# Ver IP asignada al túnel
ip addr show ppp0

# Ver tabla de rutas
ip route show

# Ping al gateway de la LAN remota (R2)
ping 10.11.85.129

# Ping al host destino (PC1)
ping 10.11.85.130
```

<img width="578" height="108" alt="image" src="https://github.com/user-attachments/assets/530af29d-872e-46d3-a37a-57b5034a99d9" />

<img width="634" height="229" alt="image" src="https://github.com/user-attachments/assets/14c04130-6433-430d-b448-99494d9ecd58" />

<img width="413" height="160" alt="image" src="https://github.com/user-attachments/assets/eafa7ac6-8b55-4e0f-9dbc-c36ebbacddaf" />

### Resultados esperados

- `ppp0` recibe la IP `10.11.85.202/32` del pool de R2
- `show crypto isakmp sa` en R2 muestra estado `QM_IDLE` con `ACTIVE`
- `show vpdn session` en R2 muestra 1 sesión activa con usuario `kali`
- Ping a `10.11.85.129` y `10.11.85.130` responden desde Kali via `ppp0`

<img width="775" height="386" alt="image" src="https://github.com/user-attachments/assets/26e831da-898f-4476-ba72-5e74a9d3a371" />

<img width="975" height="294" alt="image" src="https://github.com/user-attachments/assets/d9e43623-2d0d-4f32-906b-ce31205e1bdb" />

<img width="975" height="360" alt="image" src="https://github.com/user-attachments/assets/c16a50e3-0b11-48dc-8fba-a025c51ed180" />

---

> **Tecnologías utilizadas:** Cisco IOS · StrongSwan (IKEv1) · xl2tpd · PPP MS-CHAPv2 · GNS3
