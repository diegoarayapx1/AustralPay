> ⚠️ **AustralPay y la consultora DiegoAraya son entidades FICTICIAS, creadas exclusivamente con fines de laboratorio y portafolio. Ninguna configuración, dirección o dato aquí descrito corresponde a una red o cliente real.**

# Configuraciones — AustralPay

Configuraciones de los **8 equipos** de AustralPay, capturadas al cierre del **Bloque 8** (2026-07-15, ~23:35 UTC) y sanitizadas para publicación.

Este es el **estado congelado** que valida la matriz de conectividad y las pruebas de failover del informe del Bloque 8. Cualquier `show` de la evidencia corresponde a estos archivos.

---

## Inventario

| Archivo | Equipo | Rol | Imagen |
|---|---|---|---|
| `BR-STGO.txt` | `BR-STGO` | Borde STGO — eBGP ×2, NAT, VPN, filtro inter-sitio, **reloj NTP** | CSR1000v · IOS-XE 17.3 |
| `BR-VALPO.txt` | `BR-VALPO` | Borde VALPO — eBGP ×2, NAT, VPN, filtro inter-sitio | CSR1000v · IOS-XE 17.3 |
| `DLS1-STGO.txt` | `DLS1-STGO` | Distribución STGO — HSRP activo V10/V15/V30, DHCP mitad baja | vIOS-L2 · IOS 15.2 |
| `DLS2-STGO.txt` | `DLS2-STGO` | Distribución STGO — HSRP activo V20/V50, DHCP mitad alta | vIOS-L2 · IOS 15.2 |
| `ALS1-STGO.txt` | `ALS1-STGO` | Acceso STGO — port-security, protected ports, DHCP snooping | vIOS-L2 · IOS 15.2 |
| `MLS1-VALPO.txt` | `MLS1-VALPO` | Distribución VALPO — HSRP activo V40, DHCP mitad baja | vIOS-L2 · IOS 15.2 |
| `MLS2-VALPO.txt` | `MLS2-VALPO` | Distribución VALPO — HSRP activo V41/V55, DHCP mitad alta | vIOS-L2 · IOS 15.2 |
| `ALS1-VALPO.txt` | `ALS1-VALPO` | Acceso VALPO — port-security, protected ports, DHCP snooping | vIOS-L2 · IOS 15.2 |

> Los routers `ISP-1` e `ISP-2` **no se incluyen**: no son equipos de AustralPay. Representan el sustrato de "internet" del laboratorio.

---

## Sanitización

Se reemplazó por `<REDACTED>`:

| Elemento | Por qué |
|---|---|
| `enable secret 8` · `username ... secret 8` | Hash tipo 8 (SHA256). Es robusto, pero publicar hashes regala el trabajo de un ataque offline. |
| `crypto isakmp key` | Estaba **en texto claro** en la configuración. El más importante de la lista. |
| `ntp authentication-key ... md5 ... 7` | Tipo 7 es **ofuscación reversible**, no cifrado: se revierte en un segundo. IOS no ofrece un tipo más fuerte para NTP. |
| `crypto pki certificate chain` | Certificados autofirmados generados por IOS + CA de licenciamiento de Cisco. Identificadores de la instancia, sin valor documental. |
| `license udi ... sn` | Serial de la instancia CSR1000V. |
| Banners EULA de IOSv | Texto legal de Cisco incluido con la imagen, repetido 3× por switch. Se conserva el comando, no el cuerpo. |

**NO se tocó nada más.** Direccionamiento, ACLs, políticas, HSRP, RPVST+, OSPF, BGP, NAT, route-maps, prefix-lists, MACs sticky y descripciones están **intactos** — ahí vive el contenido del entregable. Las direcciones "públicas" usan rangos de documentación **RFC 5737** (`203.0.113.0/24`, `198.51.100.0/24`, `192.0.2.0/24`) y el interior es **RFC 1918**: no hay espacio ruteable real que ocultar, y ocultarlo destruiría el valor del documento.

---

## Artefactos de fábrica presentes

Elementos que **no configuró DiegoAraya** y que un revisor va a encontrar. Se declaran en vez de borrarlos del archivo: maquillar un `running-config` es lo contrario de entregarlo.

| Artefacto | Origen |
|---|---|
| `sl_def_acl` | **La genera IOS automáticamente** al configurar `login block-for` (§7.3). Es la ACL de *quiet-mode*: el artefacto concreto del trade-off ya documentado — durante el bloqueo, deniega telnet/http/ssh a **todos**, incluido el admin legítimo. |
| `preauth_ipv4_acl` · `CISCO-CWA-URL-REDIRECT-ACL` | Vienen con la imagen vIOS-L2 (plantillas de autenticación web/CWA). No están aplicadas a ninguna interfaz. |
| `crypto pki trustpoint TP-self-signed-*` · `SLA-TrustPoint` | Los genera IOS-XE solo. El servidor HTTPS que usaba el primero fue **deshabilitado en el Bloque 8**; el trustpoint queda huérfano. |
| `service call-home` / bloque `call-home` | Viene con Smart Licensing en el CSR1000v. |
| Banners EULA (`banner exec` / `incoming` / `login`) | Restricción de licencia de la imagen IOSv. |
| VLANs `1002-1005` (fddi/token-ring) | Reservadas por IOS, no eliminables. Visibles en `show vlan brief`, no en el `running-config`. |
| UDP **2228** (switches) · TCP **21111** (bordes) | Puertos en escucha que **no fueron identificados con certeza** durante el Bloque 8. Se declaran como pendiente en vez de atribuirles un servicio por inferencia. |

> Estas ACL de fábrica **no aparecen en los archivos de este directorio**: solo son visibles con `show ip access-lists`. Se documentan acá para que su aparición en la evidencia no se lea como configuración propia.

---

## Notas de reconstrucción

- **`exec-timeout 10 0`** no aparece en ningún archivo: **es el valor por defecto de IOS** y `show running-config` no muestra defaults. Se verificó con `show running-config all | include exec-timeout` en los 8 equipos. La promesa de §7.3 se cumple — por default, no por configuración explícita.
- **Los 6 switches están en `vtp mode transparent`**, así que sus VLANs viven en el `running-config` y estos archivos **sí reconstruyen el equipo**. `ALS1-VALPO` era la excepción (VTP server, VLANs solo en `vlan.dat`) y se corrigió en el Bloque 8.
- **`router-id` explícito** en los dos bordes (`10.255.255.1` / `.2`). Esas direcciones **no corresponden a ninguna interfaz**: no existe un `Loopback` con esa IP. `Loopback0` es el **loopback público** que ancla la VPN.

---

> ⚠️ **AustralPay y DiegoAraya son ficticias, con fines exclusivos de laboratorio y portafolio.**
