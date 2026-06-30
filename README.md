# 🔐 VPN Cisco IPSec IKEv2 Site-to-Site — Basada en Enrutamiento

<div align="center">

![Cisco](https://img.shields.io/badge/Cisco-IOS-blue?style=for-the-badge&logo=cisco)
![IPSec](https://img.shields.io/badge/IPSec-IKEv2-green?style=for-the-badge)
![VTI](https://img.shields.io/badge/VPN-Route--Based%20(VTI)-purple?style=for-the-badge)
![Platform](https://img.shields.io/badge/Platform-PNETLab-orange?style=for-the-badge)
![License](https://img.shields.io/badge/Uso-Educativo-red?style=for-the-badge)

**Sael Germán García** | Matrícula: `2025-0725`  
Asignatura: Seguridad de Redes | Profesor: Jonathan Rondón  
Instituto Tecnológico de las Américas — ITLA | 2026

</div>

---

## 📋 Descripción

Configuración y verificación de una **VPN Cisco IPSec IKEv2 Site-to-Site basada en enrutamiento** para permitir comunicación segura entre dos LANs remotas a través de un router ISP que simula la red pública. A diferencia de una VPN basada en políticas, esta solución utiliza una **interfaz virtual de túnel (VTI) Tunnel0** protegida con IPSec, y el tráfico hacia la LAN remota se dirige mediante **rutas estáticas** hacia el túnel, logrando un diseño más flexible y similar a un enlace punto a punto.

> 💡 **Clave técnica:** No existe ACL de tráfico interesante ni crypto map en la interfaz WAN. La protección se aplica directamente sobre `Tunnel0` mediante `tunnel protection ipsec profile`, y el enrutamiento estático decide qué tráfico atraviesa el túnel.

---

## 🗺️ Topología de Red

La topología conserva dos routers peers (R1 y R2), un router ISP al centro y una LAN en cada extremo. El ISP solo brinda conectividad entre R1 y R2, sin participar en la VPN.

### 📊 Direccionamiento IP

| Dispositivo | Interfaz | IP / Máscara | Función |
|:-----------:|:--------:|:-------------:|---------|
| R1 | Ethernet0/0 | 10.7.25.1/30 | WAN hacia ISP |
| ISP | Ethernet0/0 | 10.7.25.2/30 | Enlace hacia R1 |
| ISP | Ethernet0/1 | 10.7.25.6/30 | Enlace hacia R2 |
| R2 | Ethernet0/0 | 10.7.25.5/30 | WAN hacia ISP |
| R1 | Ethernet0/1 | 10.7.25.65/27 | Gateway LAN-A |
| VPC1 | eth0 | 10.7.25.66/27 | Cliente LAN-A |
| R2 | Ethernet0/1 | 10.7.25.97/27 | Gateway LAN-B |
| VPC2 | eth0 | 10.7.25.98/27 | Cliente LAN-B |
| **R1** | **Tunnel0** | **10.7.25.137/30** | **VTI IPSec hacia R2** |
| **R2** | **Tunnel0** | **10.7.25.138/30** | **VTI IPSec hacia R1** |

### 📡 Segmentos

| Segmento | Red | Uso |
|:--------:|:---:|-----|
| WAN R1-ISP | 10.7.25.0/30 | Enlace entre R1 e ISP |
| WAN ISP-R2 | 10.7.25.4/30 | Enlace entre ISP y R2 |
| LAN-A | 10.7.25.64/27 | Red local de R1 |
| LAN-B | 10.7.25.96/27 | Red local de R2 |
| Tunnel VPN | 10.7.25.136/30 | Red punto a punto para Tunnel0 |

---

## ⚙️ Parámetros de la VPN

| Parámetro | Valor |
|:---------:|-------|
| Versión IKE | IKEv2 |
| Tipo de VPN | Site-to-Site basada en enrutamiento |
| Interfaz virtual | Tunnel0 / VTI |
| Cifrado IKEv2 | AES-CBC-256 |
| Integridad IKEv2 | SHA256 |
| Grupo Diffie-Hellman | Grupo 5 |
| Autenticación | Pre-Shared Key |
| Clave compartida | VPN12345 |
| Transform-set IPSec | esp-aes 256 esp-sha256-hmac |
| Perfil IPSec | PROFILE-IKEV2 |
| Ruta R1 → LAN-B | 10.7.25.96/27 vía Tunnel0 |
| Ruta R2 → LAN-A | 10.7.25.64/27 vía Tunnel0 |

---

## 🔍 Funcionamiento

En esta implementación, R1 y R2 crean una **interfaz virtual Tunnel0** que funciona como un enlace lógico punto a punto entre ambos routers. La protección del tráfico se aplica mediante un **perfil IPSec** (`PROFILE-IKEV2`) asociado directamente al túnel con `tunnel protection ipsec profile`.

El **enrutamiento** se encarga de enviar el tráfico de cada LAN hacia Tunnel0 mediante rutas estáticas. Por esta razón, no se utiliza una ACL de tráfico interesante ni un crypto map aplicado a la interfaz WAN como en una VPN basada en políticas — el tráfico se protege automáticamente al atravesar el túnel IPSec.

---

## 🚀 Scripts de Configuración

### ISP
```cisco
hostname ISP
interface Ethernet0/0
 description ENLACE_A_R1
 ip address 10.7.25.2 255.255.255.252
 no shutdown
interface Ethernet0/1
 description ENLACE_A_R2
 ip address 10.7.25.6 255.255.255.252
 no shutdown
```

### R1 — Configuración Principal
```cisco
hostname R1
interface Ethernet0/0
 description WAN_HACIA_ISP
 ip address 10.7.25.1 255.255.255.252
 no shutdown
interface Ethernet0/1
 description LAN_A
 ip address 10.7.25.65 255.255.255.224
 no shutdown
ip route 0.0.0.0 0.0.0.0 10.7.25.2

crypto ikev2 proposal IKEV2-PROP
 encryption aes-cbc-256
 integrity sha256
 group 5

crypto ikev2 policy IKEV2-POLICY
 proposal IKEV2-PROP

crypto ikev2 keyring IKEV2-KEYRING
 peer R2
  address 10.7.25.5
  pre-shared-key local VPN12345
  pre-shared-key remote VPN12345

crypto ikev2 profile IKEV2-PROFILE
 match identity remote address 10.7.25.5 255.255.255.255
 identity local address 10.7.25.1
 authentication remote pre-share
 authentication local pre-share
 keyring local IKEV2-KEYRING

crypto ipsec transform-set TS-IKEV2 esp-aes 256 esp-sha256-hmac
 mode tunnel

crypto ipsec profile PROFILE-IKEV2
 set transform-set TS-IKEV2
 set pfs group5
 set ikev2-profile IKEV2-PROFILE

interface Tunnel0
 description VTI_IPSEC_IKEV2_HACIA_R2
 ip address 10.7.25.137 255.255.255.252
 tunnel source Ethernet0/0
 tunnel destination 10.7.25.5
 tunnel mode ipsec ipv4
 tunnel protection ipsec profile PROFILE-IKEV2
 no shutdown

ip route 10.7.25.96 255.255.255.224 Tunnel0
```

### R2 — Configuración Principal
```cisco
hostname R2
interface Ethernet0/0
 description WAN_HACIA_ISP
 ip address 10.7.25.5 255.255.255.252
 no shutdown
interface Ethernet0/1
 description LAN_B
 ip address 10.7.25.97 255.255.255.224
 no shutdown
ip route 0.0.0.0 0.0.0.0 10.7.25.6

crypto ikev2 proposal IKEV2-PROP
 encryption aes-cbc-256
 integrity sha256
 group 5

crypto ikev2 policy IKEV2-POLICY
 proposal IKEV2-PROP

crypto ikev2 keyring IKEV2-KEYRING
 peer R1
  address 10.7.25.1
  pre-shared-key local VPN12345
  pre-shared-key remote VPN12345

crypto ikev2 profile IKEV2-PROFILE
 match identity remote address 10.7.25.1 255.255.255.255
 identity local address 10.7.25.5
 authentication remote pre-share
 authentication local pre-share
 keyring local IKEV2-KEYRING

crypto ipsec transform-set TS-IKEV2 esp-aes 256 esp-sha256-hmac
 mode tunnel

crypto ipsec profile PROFILE-IKEV2
 set transform-set TS-IKEV2
 set pfs group5
 set ikev2-profile IKEV2-PROFILE

interface Tunnel0
 description VTI_IPSEC_IKEV2_HACIA_R1
 ip address 10.7.25.138 255.255.255.252
 tunnel source Ethernet0/0
 tunnel destination 10.7.25.1
 tunnel mode ipsec ipv4
 tunnel protection ipsec profile PROFILE-IKEV2
 no shutdown

ip route 10.7.25.64 255.255.255.224 Tunnel0
```

### Configuración de VPCs
```bash
# VPC1
ip 10.7.25.66 255.255.255.224 10.7.25.65

# VPC2
ip 10.7.25.98 255.255.255.224 10.7.25.97
```

---

## ✅ Verificación del Túnel

```cisco
show ip interface brief
show running-config interface tunnel0
show running-config | section crypto
show ip route 10.7.25.96
show ip route 10.7.25.64
show interface tunnel0
show crypto ikev2 sa
show crypto ipsec sa
show crypto session
```

| Comando | Estado esperado |
|:-------:|------------------|
| `show interface tunnel0` | up/up |
| `show crypto ikev2 sa` | READY |
| `show crypto ipsec sa` | encaps/decaps activos |
| `show crypto session` | UP-ACTIVE, perfil IKEV2-PROFILE asociado a Tunnel0 |

> 💡 **Nota sobre traceroute:** El mensaje "Destination port unreachable" al final del trace es normal en VPCS — usa paquetes UDP, y al llegar al host final este responde que el puerto no está disponible. Esto también confirma que el destino fue alcanzado.

---

## 📊 Resultados

| Prueba | Resultado |
|:------:|:---------:|
| Tunnel0 en R1 y R2 | ✅ up/up |
| Configuración IKEv2/IPSec (proposal, policy, keyring, profile) | ✅ Presente en ambos peers |
| Rutas estáticas hacia LAN remota vía Tunnel0 | ✅ Confirmadas en R1 y R2 |
| IKEv2 SA | ✅ READY |
| IPSec SA | ✅ Encaps/decaps activos |
| Sesión crypto | ✅ UP-ACTIVE |
| Ping VPC1 → VPC2 | ✅ Exitoso |
| Ping VPC2 → VPC1 | ✅ Exitoso |

---

## 📁 Archivos del Repositorio

| Archivo | Descripción |
|:-------:|-------------|
| [`SaelGerman_2025-0725_Script_VPN_IPSec-IKEv2-Site-to-Site-basada-en-enrutamientoP2.pdf`](SaelGerman_2025-0725_Script_VPN_IPSec-IKEv2-Site-to-Site-basada-en-enrutamientoP2.pdf) | Scripts de configuración Cisco IOS |
| [`SaelGerman_2025-0725_VPN-IPSec-IKEv2-Site-to-Site-basada-en-enrutamiento_P2.pdf`](SaelGerman_2025-0725_VPN-IPSec-IKEv2-Site-to-Site-basada-en-enrutamiento_P2.pdf) | Documentación técnica completa |

---

## 🖼️ Capturas de Pantalla

- 📸 [Figura 1 — Topología VPN IKEv2 route-based](SaelGerman_2025-0725_Capturas_IKEV2_RouteBased_GitHub/01_topologia_ikev2_route_based.png)
- 📸 [Figura 2 — Interfaces del ISP](SaelGerman_2025-0725_Capturas_IKEV2_RouteBased_GitHub/02_isp_show_ip_interface_brief.png)
- 📸 [Figura 3 — Interfaces de R1](SaelGerman_2025-0725_Capturas_IKEV2_RouteBased_GitHub/03_r1_show_ip_interface_brief.png)
- 📸 [Figura 4 — Configuración de Tunnel0 en R1](SaelGerman_2025-0725_Capturas_IKEV2_RouteBased_GitHub/04_r1_show_running_config_interface_tunnel0.png)
- 📸 [Figura 5 — Configuración crypto IKEv2/IPSec en R1](SaelGerman_2025-0725_Capturas_IKEV2_RouteBased_GitHub/05_r1_show_running_config_section_crypto.png)
- 📸 [Figura 6 — Ruta de R1 hacia LAN remota por Tunnel0](SaelGerman_2025-0725_Capturas_IKEV2_RouteBased_GitHub/06_r1_show_ip_route_lan_remota.png)
- 📸 [Figura 7 — Interfaces de R2](SaelGerman_2025-0725_Capturas_IKEV2_RouteBased_GitHub/07_r2_show_ip_interface_brief.png)
- 📸 [Figura 8 — Configuración de Tunnel0 en R2](SaelGerman_2025-0725_Capturas_IKEV2_RouteBased_GitHub/08_r2_show_running_config_interface_tunnel0.png)
- 📸 [Figura 9 — Configuración crypto IKEv2/IPSec en R2](SaelGerman_2025-0725_Capturas_IKEV2_RouteBased_GitHub/09_r2_show_running_config_section_crypto.png)
- 📸 [Figura 10 — Ruta de R2 hacia LAN remota por Tunnel0](SaelGerman_2025-0725_Capturas_IKEV2_RouteBased_GitHub/10_r2_show_ip_route_lan_remota.png)
- 📸 [Figura 11 — Estado de Tunnel0 en R1](SaelGerman_2025-0725_Capturas_IKEV2_RouteBased_GitHub/11_r1_show_interface_tunnel0.png)
- 📸 [Figura 12 — Estado de Tunnel0 en R2](SaelGerman_2025-0725_Capturas_IKEV2_RouteBased_GitHub/12_r2_show_interface_tunnel0.png)
- 📸 [Figura 13 — Prueba desde VPC1 hacia VPC2](SaelGerman_2025-0725_Capturas_IKEV2_RouteBased_GitHub/13_vpc1_show_ip_ping_trace_to_vpc2.png)
- 📸 [Figura 14 — Prueba desde VPC2 hacia VPC1](SaelGerman_2025-0725_Capturas_IKEV2_RouteBased_GitHub/14_vpc2_show_ip_ping_trace_to_vpc1.png)
- 📸 [Figura 15 — Verificación IKEv2 SA en R1](SaelGerman_2025-0725_Capturas_IKEV2_RouteBased_GitHub/15_r1_show_crypto_ikev2_sa.png)
- 📸 [Figura 16 — Verificación IKEv2 SA en R2](SaelGerman_2025-0725_Capturas_IKEV2_RouteBased_GitHub/16_r2_show_crypto_ikev2_sa.png)
- 📸 [Figura 17 — Verificación IPSec SA en R1](SaelGerman_2025-0725_Capturas_IKEV2_RouteBased_GitHub/17_r1_show_crypto_ipsec_sa.png)
- 📸 [Figura 18 — Verificación IPSec SA en R2](SaelGerman_2025-0725_Capturas_IKEV2_RouteBased_GitHub/18_r2_show_crypto_ipsec_sa.png)
- 📸 [Figura 19 — Sesión crypto activa en R1](SaelGerman_2025-0725_Capturas_IKEV2_RouteBased_GitHub/19_r1_show_crypto_session.png)
- 📸 [Figura 20 — Sesión crypto activa en R2](SaelGerman_2025-0725_Capturas_IKEV2_RouteBased_GitHub/20_r2_show_crypto_session.png)

---

## 📎 Recursos

📄 **Documentación Técnica:** [Ver Informe PDF](SaelGerman_2025-0725_VPN-IPSec-IKEv2-Site-to-Site-basada-en-enrutamiento_P2.pdf)  
▶️ **Video Demostración:** [Ver en YouTube](https://youtu.be/Vol05Ka-xvM)

---

## 📚 Referencias

1. Cisco Systems. *Configuring Site-to-Site IPSec VPN using IKEv2 with VTI (Route-Based)*. Documentación oficial Cisco IOS.
2. Reconocimiento especial: Troubleshooting, base del script y documentación apoyado en Inteligencia Artificial.

---

<div align="center">

*Este laboratorio fue desarrollado exclusivamente con fines académicos y educativos.*

</div>
