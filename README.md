# P3 – Seguridad de Redes | Infraestructura 1

Implementación de una infraestructura segmentada y protegida mediante **FortiGate 7.0.3**, **Cisco IOSvL2**, **VLANs**, una **DMZ** y políticas de control de acceso entre usuarios, servidores e Internet.

> **Asignatura:** Seguridad de Redes  
> **Proyecto:** P3  
> **Infraestructura:** 1  
> **Matrícula:** 2025-1331

---

## 🎥 Video de demostración

> Agregar aquí el enlace del video de Infraestructura 1.

---

## 🧭 Objetivo

Diseñar e implementar una infraestructura de red con segmentación lógica y controles de seguridad que permita:

- Separar usuarios restringidos y privilegiados mediante VLANs.
- Alojar servidores críticos dentro de una DMZ.
- Impedir fugas de tráfico desde la DMZ hacia las LAN.
- Restringir el acceso de usuarios de VLAN10 al servidor de inventario.
- Permitir acceso SSH a servidores únicamente desde VLAN20.
- Restringir el acceso general a Internet desde la DMZ, permitiendo únicamente DNS y endpoints necesarios para actualizaciones Debian.
- Aplicar medidas básicas de hardening en el switch de acceso.

---

## 🏗️ Topología

La infraestructura utiliza:

- 1 FortiGate
- 1 switch Cisco IOSvL2
- 2 VLAN de usuarios
- 3 servidores en DMZ
- 1 cliente Windows de pruebas
- GNS3 integrado con Proxmox

> El diagrama de topología se agregará en la carpeta `diagrams/`.

---

## 🌐 Plan de direccionamiento

| Segmento | Red | Gateway | Uso |
|---|---|---|---|
| VLAN10 | `10.31.10.0/25` | `10.31.10.1` | Usuarios restringidos |
| VLAN20 | `10.31.20.0/25` | `10.31.20.1` | Usuarios privilegiados |
| DMZ | `10.31.30.0/28` | `10.31.30.1` | Servidores |

### Servidores DMZ

| Servidor | Dirección IP | Función |
|---|---:|---|
| INFRA1-WEB-CAJA | `10.31.30.2` | Servidor web de Caja |
| INFRA1-WEB-INVENTARIO | `10.31.30.3` | Servidor web de Inventario |
| INFRA1-DB01 | `10.31.30.4` | Servidor de base de datos MariaDB |

---

## 🔥 FortiGate

### Interfaces principales

- **port1 / WAN-MGMT:** acceso de administración y salida WAN.
- **VLAN10-USERS:** VLAN ID 10 — `10.31.10.1/25`.
- **VLAN20-ADMIN:** VLAN ID 20 — `10.31.20.1/25`.
- **P3-INFRA1-DMZ / port3:** `10.31.30.1/28`.

### DHCP

FortiGate entrega direccionamiento dinámico para las dos VLAN de usuarios:

- VLAN10: `10.31.10.20 - 10.31.10.120`
- VLAN20: `10.31.20.20 - 10.31.20.120`

---

## 🛡️ Políticas de firewall

### DMZ → Internet

| Política | Acción | Propósito |
|---|---|---|
| `DMZ-ALLOW-DNS` | ACCEPT | Permitir DNS hacia Cloudflare |
| `DMZ-ALLOW-DEBIAN-UPDATES` | ACCEPT | Permitir actualizaciones desde repositorios Debian autorizados |
| `DMZ-DENY-INTERNET` | DENY | Bloquear el resto del acceso a Internet |

### DMZ → LAN

| Política | Acción |
|---|---|
| `DMZ-DENY-VLAN10` | DENY |
| `DMZ-DENY-VLAN20` | DENY |

Estas reglas evitan que los servidores de la DMZ puedan iniciar tráfico hacia las redes internas.

### VLAN10 → DMZ

| Política | Resultado |
|---|---|
| `VLAN10-ALLOW-CAJA` | Permite HTTP/HTTPS hacia WEB-CAJA |
| `VLAN10-DENY-INVENTARIO` | Bloquea HTTP/HTTPS hacia WEB-INVENTARIO |
| `VLAN10-DENY-SSH` | Bloquea SSH hacia los servidores |

### VLAN20 → DMZ

| Política | Resultado |
|---|---|
| `VLAN20-ALLOW-SSH` | Permite SSH hacia los tres servidores de la DMZ |

---

## 🔀 Switch Cisco – SW-INFRA1

### VLANs

| VLAN | Nombre | Puertos |
|---:|---|---|
| 10 | `USERS-RESTRICTED` | Gi0/1 |
| 20 | `USERS-PRIVILEGED` | Gi0/2, Gi0/3 |
| 999 | `UNUSED_PORTS` | Gi1/0 - Gi1/3 |

### Trunk

- **Gi0/0**
- Encapsulación: **802.1Q**
- VLAN permitidas: **10,20**

### Hardening aplicado

- PortFast en puertos de acceso.
- BPDU Guard en puertos de acceso.
- Port-Security con máximo de 1 MAC.
- Modo de violación: `restrict`.
- Puertos no utilizados asignados a VLAN999.
- Puertos no utilizados administrativamente apagados.

---

## 🧪 Pruebas realizadas

### VLAN10 – USERS-RESTRICTED

Cliente de pruebas:

- IP obtenida por DHCP: `10.31.10.21/25`

Resultados:

- ✅ Acceso HTTP a `10.31.30.2` — WEB-CAJA
- ❌ Acceso HTTP a `10.31.30.3` — WEB-INVENTARIO
- ❌ Acceso SSH a servidores DMZ
- ✅ FortiGate registra los bloqueos como **policy violation**

### VLAN20 – USERS-PRIVILEGED

Cliente de pruebas:

- IP obtenida por DHCP: `10.31.20.21/25`

Resultados:

- ✅ SSH a `10.31.30.2`
- ✅ SSH a `10.31.30.3`
- ✅ SSH a `10.31.30.4`
- ✅ Sesión SSH real establecida con WEB-CAJA
- ✅ FortiGate registra el tráfico mediante `VLAN20-ALLOW-SSH`

### Restricción de Internet desde la DMZ

Resultados:

- ✅ Resolución DNS autorizada.
- ✅ `apt update` funciona contra los repositorios Debian autorizados.
- ❌ Acceso general a Internet bloqueado.
- ❌ Tráfico no autorizado registrado por `DMZ-DENY-INTERNET`.

---

## 🗂️ Estructura del repositorio

```text
P3-Seguridad-Redes-2025-1331-Infra1/
├── README.md
├── docs/
├── diagrams/
├── screenshots/
│   ├── fortigate/
│   ├── switch/
│   ├── vlan10/
│   ├── vlan20/
│   ├── dmz/
│   └── pruebas/
├── configs/
└── scripts/
```

---

## 📁 Configuraciones

La carpeta `configs/` contendrá, entre otros archivos:

- `SW-INFRA1-running-config.txt`

La configuración del FortiGate se documenta principalmente mediante capturas de la GUI y evidencias de las políticas y logs.

---

## ✅ Resultado

La Infraestructura 1 implementa segmentación mediante VLANs, aislamiento de servidores en DMZ, restricciones de acceso por origen y servicio, control de salida a Internet, registros de violaciones de política y hardening básico del switch.

Las pruebas realizadas confirman que las políticas de seguridad se aplican correctamente y que cada segmento posee únicamente los accesos definidos para su función.
