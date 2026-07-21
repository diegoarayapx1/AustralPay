> ⚠️ **AustralPay y la consultora DiegoAraya son entidades FICTICIAS, creadas exclusivamente con fines de laboratorio y portafolio. Ningún dato, dirección, dispositivo o infraestructura descrito aquí corresponde a una red real.**

# Diseño de red — AustralPay
### Bloque 1 · documento de diseño (topología, direccionamiento, enrutamiento, redundancia, seguridad)
### Actualizado tras Bloque 8 (Verificación y entrega)

> 🧪 Todo lo descrito es ficticio, con fines de laboratorio. Las direcciones "públicas" usan rangos de documentación (RFC 5737), no espacio real ruteable.

> 📌 **Documento vivo.** El cuerpo refleja el diseño *actual*. Los cambios respecto a la idea inicial se registran en el **Historial de decisiones (changelog)** al final. Ver ese changelog para entender la evolución.

> 🔧 **Estado de implementación.** El cuerpo describe el diseño completo, y **todo él está implementado**. El **Bloque 8** (2026-07-15/16) lo validó end-to-end —matriz de conectividad **59/59** con predicción escrita antes de cada prueba, failover de ISP y de HSRP medidos con ping continuo— y corrigió las **11 divergencias** que encontró entre lo que este documento declaraba y lo que los equipos hacían. Lo marcado `[B8]` nació de esas correcciones. El estado congelado de los 8 equipos vive en `configs/`. Ver sección 9.

---

## 1. Resumen del diseño

Red corporativa de dos sitios para AustralPay: sitio matriz **STGO** (Providencia) y **sede regional de producción VALPO** (Valparaíso), unidos por VPN **GRE-over-IPSec** sobre internet. Diseño jerárquico clásico (borde / distribución-núcleo / acceso).

**Decisiones rectoras:**
- **IGP único: OSPF** en todo el interior. Nada de EIGRP ni redistribución mutua interna — un solo protocolo es más limpio y mantenible.
- **BGP solo en el borde.** **Ambas sedes con multihoming**, compartiendo los mismos dos ISP de tránsito (ver sección 5.2 y changelog Bloque 5) para alta disponibilidad — VALPO es sede de producción regional, no un DR frío, así que necesita su propia salida resiliente. `AS 65010` para AustralPay. **La ruta por defecto se aprende por BGP desde ambos upstreams** `[B8]`: ninguna sede depende de un solo ISP para salir a internet (sección 5.3).
- **VPN GRE-over-IPSec site-to-site** entre bordes, anclada a **loopbacks públicos anunciados por BGP** (no a interfaces físicas — ver sección 4.5 y changelog Bloque 5); OSPF área 0 corre sobre el túnel GRE (el crypto map puro no transporta el multicast de OSPF; GRE da la interfaz ruteable, IPSec el cifrado, en modo **tunnel**).
- **Redundancia por capa en ambas sedes:** HSRP (gateway), EtherChannel (enlaces), RPVST+ (capa 2).
- **Segmentación por política, no solo por VLAN.** El tráfico intra-sitio se filtra con **lista blanca (default-deny)** en las SVI de distribución; el inter-sitio con ACL en el túnel (secciones 7.1 y 7.2).
- **AAA local con separación de roles** (admin / operador) en los 8 equipos — defendible a esta escala; ver sección 7.3 y la lista de producción (sección 11).
- **IPv4 puro.** Direccionamiento jerárquico por sitio.

> **Rol de VALPO y DR:** VALPO es una sede regional de producción (atiende usuarios y tráfico propio). La capacidad de **disaster recovery** surge como *subproducto* de tener dos sedes de producción con separación geográfica y VPN — no como una sede dedicada a DR. La arquitectura da continuidad geográfica sin infraestructura de DR dedicada.

> **Resiliencia de salida: la promesa, medida (Bloque 8).** Hasta el B7 esta promesa **era falsa**. Los dos bordes tenían **una** ruta estática por defecto hacia `ISP-1` y **un** solo statement de NAT atado a su interfaz: la caída de `ISP-1` dejaba la sede sin internet. El túnel sí sobrevivía —está anclado a un loopback aprendido por ambos ISP (sección 4.7)—, así que **el multihoming protegía la VPN, no el negocio**. Con la default aprendida por BGP de ambos upstreams y el NAT siguiendo al ruteo (secciones 5.2-5.4), la promesa se cumple **y está medida**: **129 s de corte**, y después internet **sostenido** por `ISP-2` (171 paquetes consecutivos a `ttl=252`). La afirmación honesta no es *"sin interrupción"*: es **"ninguna sede queda sin salida"**, con un hoyo medido de 129 s. Por qué son 129 s y no 5, ver sección 11 (BFD). Ver changelog B8-#4.

---

## 2. Inventario de dispositivos

| Sitio | Equipo | Rol | Funciones |
|---|---|---|---|
| STGO | `BR-STGO` | Router de borde (CSR1000v) | eBGP ×2 (multihoming), NAT, extremo VPN GRE-over-IPSec, filtro inter-sitio, **servidor NTP de la red** `[B8]` (sección 7.5) |
| STGO | `DLS1-STGO` | Switch multicapa | Distribución/núcleo, SVI, HSRP, OSPF, RPVST+, ACL de política intra-sitio, DHCP `[B7]` |
| STGO | `DLS2-STGO` | Switch multicapa | Distribución/núcleo, SVI, HSRP, OSPF, RPVST+, ACL de política intra-sitio, DHCP `[B7]` |
| STGO | `ALS1-STGO` | Switch de acceso (L2 puro) | VLANs, port-security, PortFast/BPDUGuard, protected ports `[B7]`, DHCP snooping `[B7]` |
| VALPO | `BR-VALPO` | Router de borde (CSR1000v) | eBGP ×2 (multihoming), NAT, extremo VPN GRE-over-IPSec, filtro inter-sitio |
| VALPO | `MLS1-VALPO` | Switch multicapa | Distribución, SVI, HSRP, OSPF, RPVST+, ACL de política intra-sitio, DHCP `[B7]` |
| VALPO | `MLS2-VALPO` | Switch multicapa | Distribución, SVI, HSRP, OSPF, RPVST+ (redundancia de gateway), ACL de política intra-sitio, DHCP `[B7]` |
| VALPO | `ALS1-VALPO` | Switch de acceso (L2 puro) | VLANs, port-security, protected ports `[B7]`, DHCP snooping `[B7]` |
| Tránsito (fuera del AS de AustralPay) | `ISP-1` | Router ISP (vIOS) | eBGP AS 65001, doble rol: ISP-A hacia STGO / ISP-C hacia VALPO |
| Tránsito (fuera del AS de AustralPay) | `ISP-2` | Router ISP (vIOS) | eBGP AS 65002, doble rol: ISP-B hacia STGO / ISP-D hacia VALPO |

> Los **8 equipos de AustralPay** llevan plano de gestión con SSHv2, usuarios nombrados y separación de roles (sección 7.3).
>
> `MLS2-VALPO` se agrega para dar **redundancia de gateway interna (HSRP)** en VALPO, coherente con su rol de sede de producción. VALPO pasa de "colapsado de un solo switch" a un par de distribución chico, tipo mini-STGO.
>
> `ISP-1`/`ISP-2` no son equipos de AustralPay — representan el sustrato de "internet" que conecta ambas sedes; ver sección 5.2 para el porqué de esta simplificación de dos AS compartidos en vez de cuatro AS independientes.
>
> **Nota de plataforma (Bloque 6):** la imagen **vIOS-L2 trae `ip routing` activo por defecto**, lo que anula silenciosamente el `ip default-gateway` de un switch de acceso. Ambos ALS llevan `no ip routing` explícito para operar como L2 puro, que es su rol. Ver changelog.

---

## 3. Plan de VLANs

| Sitio | VLAN | Nombre | Subred | Gateway | Direccionamiento |
|---|---|---|---|---|---|
| STGO | 10 | Servidores-App | `10.1.10.0/24` | `10.1.10.1` (HSRP VIP) | Estático |
| STGO | **15** `[B7]` | **Servidores-DB** | `10.1.15.0/24` | `10.1.15.1` (HSRP VIP) | Estático |
| STGO | 20 | Operaciones | `10.1.20.0/24` | `10.1.20.1` (HSRP VIP) | **DHCP** `[B7]` |
| STGO | 30 | Gestión | `10.1.30.0/24` | `10.1.30.1` (HSRP VIP) | Estático |
| STGO | **50** `[B7]` | **Invitados** | `10.1.50.0/24` | `10.1.50.1` (HSRP VIP) | **DHCP** `[B7]` |
| STGO | 99 | Tránsito OSPF (DLS1↔DLS2) | `10.255.99.0/30` | — | Estático |
| STGO | **888** `[B8]` | **PARKING-NO-USADO** | — (sin SVI) | — | — (VLAN en `shutdown`) |
| STGO | 999 | Nativa (sin uso) | — | — | — |
| VALPO | 40 | Oficina regional | `10.2.40.0/24` | `10.2.40.1` (HSRP VIP) | **DHCP** `[B7]` |
| VALPO | 41 | Gestión | `10.2.41.0/24` | `10.2.41.1` (HSRP VIP) | Estático |
| VALPO | **55** `[B7]` | **Invitados** | `10.2.55.0/24` | `10.2.55.1` (HSRP VIP) | **DHCP** `[B7]` |
| VALPO | 99 | Tránsito OSPF (MLS1↔MLS2) | `10.255.99.4/30` | — | Estático |
| VALPO | **888** `[B8]` | **PARKING-NO-USADO** | — (sin SVI) | — | — (VLAN en `shutdown`) |
| VALPO | 999 | Nativa (sin uso) | — | — | — |

**Direcciones de gestión de los switches de acceso** (agregadas en el Bloque 6): `ALS1-STGO` → `10.1.30.10`, `ALS1-VALPO` → `10.2.41.10`. Ambos con `ip default-gateway` al VIP de HSRP de su VLAN de gestión (son L2 puros, sin tabla de ruteo propia).

> **Separación App/DB (Bloque 6, ejecuta B7):** `austral-app` y `austral-db` compartían VLAN 10 — su tráfico era **L2 puro, sin pasar por ninguna SVI**, así que **ninguna ACL podía filtrarlo**: quedaba permitido por física, no por política. Con `austral-db` en VLAN 15 propia, el salto app→db cruza una SVI y la política de la sección 7.2 puede aplicarse. Ver changelog.
>
> **VLAN de Invitados (Bloque 6, ejecuta B7):** segmento para dispositivos **transitorios y no confiables**, con acceso **solo a internet** y aislamiento total del interior (sección 7.2). No hay AP en el laboratorio; la configuración switch-side es idéntica con o sin punto de acceso (un AP se conecta por un puerto y el switch no distingue si del otro lado hay radio). El nombre `Invitados` —en vez de "Wifi"— evita afirmar una capacidad wireless que no se probó.
>
> **VLAN 888 — `PARKING-NO-USADO` (Bloque 8):** destino de los puertos no usados (sección 7.4). Sin SVI, sin ruteo, y la VLAN misma en `shutdown`. Existe en los 4 switches L3 (`DLS1`, `DLS2`, `MLS1`, `MLS2`); los dos ALS no la necesitan porque no tienen puertos libres. **Tres números, tres propósitos, ninguno confundible: `99` tránsito · `888` parqueo · `999` nativa.** Se descartó `998`: es un **typo de distancia de 999**, y ese typo aterriza el puerto parqueado en la **nativa** — el fallo exacto que el número venía a evitar.
>
> Los servidores del SOC/NOC (`austral-app` en VLAN 10, `austral-db` en VLAN 15) viven en **STGO**; las estaciones en **VLAN 20**. La VLAN 30/41 de gestión es la que después monitorea Zabbix por SNMP, es la única VLAN con tránsito inter-sitio permitido (sección 7.1), y es la única con acceso a las líneas VTY (sección 7.3).

---

## 4. Plan de direccionamiento IPv4

### 4.1 Segmentación por sitio
- **STGO:** `10.1.0.0/16`
- **VALPO:** `10.2.0.0/16`
- **Infraestructura (P2P, loopbacks, túnel):** `10.255.0.0/16`
- **Direccionamiento público de laboratorio (RFC 5737):** `203.0.113.0/24`, `198.51.100.0/24`, `192.0.2.0/24`

### 4.2 Enlaces de infraestructura

| Enlace | Subred | Área OSPF |
|---|---|---|
| Túnel GRE-over-IPSec `BR-STGO`↔`BR-VALPO` | `10.255.0.0/30` | Área 0 (backbone) |
| `DLS1-STGO`↔`BR-STGO` | `10.255.1.0/30` (DLS1 `.1` · BR-STGO `.2`) | Área 1 |
| `DLS2-STGO`↔`BR-STGO` | `10.255.1.4/30` (DLS2 `.5` · BR-STGO `.6`) | Área 1 |
| VLAN 99 — tránsito OSPF `DLS1`↔`DLS2` (sobre peer-link Po1, p2p) | `10.255.99.0/30` | Área 1 |
| Peer link `DLS1`↔`DLS2` (L2 trunk, transporta VLANs 10/15/20/30/50/99) | — (troncal, sin IP propia) | Área 1 (SVIs) |
| `MLS1-VALPO`↔`BR-VALPO` | `10.255.2.0/30` (MLS1 `.1` · BR-VALPO `.2`) | Área 2 |
| `MLS2-VALPO`↔`BR-VALPO` | `10.255.2.4/30` (MLS2 `.5` · BR-VALPO `.6`) | Área 2 |
| VLAN 99 — tránsito OSPF `MLS1`↔`MLS2` (sobre peer-link Po1, p2p) | `10.255.99.4/30` | Área 2 |
| Peer link `MLS1`↔`MLS2` VALPO (L2 trunk, transporta VLANs 40/41/55/99) | — (troncal, sin IP propia) | Área 2 (SVIs) |

> **VLAN 99 de tránsito (ambas sedes)** y **bordes dual-homed (ambas sedes)**: sin cambios respecto al Bloque 4 — ver ese changelog. Ejecutado en el Bloque 5: `ip ospf network point-to-point` aplicado también en `Gi1/0` de `DLS1`/`DLS2` (quedaba pendiente desde el Bloque 3), dejando las adyacencias borde↔distribución simétricas en ambos sitios (`FULL/-`, sin elección DR/BDR).

### 4.3 Router-id de OSPF `[B8]`

| Equipo | Router-id | ¿Existe como interfaz? |
|---|---|---|
| `BR-STGO` | `10.255.255.1` | **No** |
| `BR-VALPO` | `10.255.255.2` | **No** |
| `DLS1-STGO` | `10.1.1.1` | **No** |
| `DLS2-STGO` | `10.1.1.2` | **No** |
| `MLS1-VALPO` | `10.2.1.1` | **No** |
| `MLS2-VALPO` | `10.2.1.2` | **No** |

**Esquema:** `10.255.255.x` bordes · `10.1.1.x` distribución STGO · `10.2.1.x` distribución VALPO. Fijado a mano con `router-id` bajo `router ospf 1` en los 6 equipos que corren OSPF.

> ⚠️ **Ninguna de estas seis direcciones corresponde a una interfaz, y no hay que crearlas.** El router-id de OSPF es un **identificador de 32 bits**: se escribe con formato de IP porque históricamente se derivaba de una interfaz, pero no necesita existir en ninguna parte ni ser alcanzable. Fijarlo explícitamente lo hace **estable e independiente de qué interfaces estén arriba**, que es todo lo que se busca.

> **Corrección del Bloque 8 (3ª divergencia — la más peligrosa de las once).** Hasta el B7 este documento declaraba que `BR-STGO`/`BR-VALPO` tenían un **`Loopback0` con `10.255.255.1`/`.2`**. Esa interfaz **no existe**. Peor: en esos dos equipos el nombre `Loopback0` está **tomado** por el loopback público que ancla la VPN (`192.0.2.100`/`.101`, sección 4.7).
>
> El riesgo no era cosmético. En el **Bloque 6 los dos bordes se reconstruyeron desde cero** tras una corrupción de disco. Una reconstrucción guiada por la tabla vieja habría configurado `interface Loopback0` / `ip address 10.255.255.2` — **pisándole al equipo el anclaje del túnel**. La tabla no estaba incompleta: estaba **en el punto exacto donde equivocarse tumba la VPN**, en el documento que se usa cuando el equipo ya no está.
>
> **Y los cuatro router-id de distribución nunca se habían documentado:** el mismo valor invisible, cuatro equipos más.

> **De dónde vienen los `10.255.255.x`:** se planificaron para peering **iBGP** entre bordes. **iBGP fue eliminado del diseño en el Bloque 6** (sección 5.2 y changelog). Sobrevivieron como router-id, que es el único trabajo que hacen hoy.

### 4.4 Enlaces públicos (ISP) — rangos de documentación RFC 5737

| Enlace | Subred | Direcciones |
|---|---|---|
| `BR-STGO`↔`ISP-1` (rol ISP-A) | `203.0.113.0/30` | ISP-1 `.1` · BR-STGO `.2` |
| `BR-STGO`↔`ISP-2` (rol ISP-B) | `203.0.113.4/30` | ISP-2 `.5` · BR-STGO `.6` |
| `BR-VALPO`↔`ISP-1` (rol ISP-C) | `198.51.100.0/30` | ISP-1 `.1` · BR-VALPO `.2` |
| `BR-VALPO`↔`ISP-2` (rol ISP-D) | `198.51.100.4/30` | ISP-2 `.5` · BR-VALPO `.6` |
| `ISP-1`↔`ISP-2` (peering/tránsito, sustrato de "internet") | `192.0.2.0/30` | ISP-1 `.1` · ISP-2 `.2` |

> **Decisión defendible:** se usan rangos RFC 5737 (`203.0.113.0/24`, `198.51.100.0/24`, `192.0.2.0/24`), reservados para documentación, en vez de IPs públicas reales inventadas. Es la práctica correcta y evita "quemar" espacio ruteable de terceros.
>
> Ver sección 5.2 y changelog del Bloque 5 para el porqué de colapsar los 4 AS ISP originalmente planificados (ISP-A/B/C/D, AS 65001-65004) en **2 AS compartidos** (`ISP-1`=65001 cubre los roles A y C, `ISP-2`=65002 cubre B y D), peereados entre sí para formar el sustrato de "internet" que interconecta ambos sitios.

### 4.5 Loopbacks públicos — anclaje de la VPN

`Loopback0` de `BR-STGO` (`192.0.2.100/32`) y de `BR-VALPO` (`192.0.2.101/32`); son el **único prefijo que cada borde anuncia por BGP** (sección 5.2). El inventario completo de las cuatro interfaces `Loopback` de la red está en la **sección 4.7**; esta sección conserva el *porqué*.

> **Por qué un loopback, no una interfaz física (Decisión Bloque 5):** el túnel GRE-over-IPSec usa `tunnel source`/`tunnel destination` apuntando a estos loopbacks, no a las interfaces hacia los ISP. Así, la resiliencia del multihoming protege también a la VPN — si un ISP cae, BGP reconverge y el túnel sigue resolviendo el camino por el otro, sin intervención manual. Es el único par de prefijos que cada borde anuncia por `network` en BGP; el resto del espacio interno (RFC 1918) nunca se anuncia.
>
> **Caveat de producción:** en una red real, un ISP filtraría cualquier anuncio más específico que /24 — se anunciaría un bloque /24 propio (PI space) con el loopback adentro, no un /32 aislado. Se usa el /32 directo en el lab porque el concepto a demostrar (anclaje resiliente vía BGP) es el mismo.

### 4.6 DHCP `[B7]`

**Criterio de asignación:** infraestructura y servidores **estáticos** (necesitan direcciones predecibles); estaciones e invitados por **DHCP** (numerosos y/o transitorios).

| VLAN | Pool | Lease | Servidores |
|---|---|---|---|
| 20 — Operaciones | `10.1.20.0/24` | **7 días** | DLS1 + DLS2 (split-scope) |
| 50 — Invitados STGO | `10.1.50.0/24` | **2 horas** | DLS1 + DLS2 (split-scope) |
| 40 — Oficina VALPO | `10.2.40.0/24` | **7 días** | MLS1 + MLS2 (split-scope) |
| 55 — Invitados VALPO | `10.2.55.0/24` | **2 horas** | MLS1 + MLS2 (split-scope) |

**Split-scope:** **`DLS1`/`MLS1` sirven la mitad baja (`.11–.132`)** y **`DLS2`/`MLS2` la mitad alta (`.133–.254`)**, uniforme en las 4 VLANs; cada uno excluye el rango del otro con `ip dhcp excluded-address` (comando **global**, no del pool). Se excluyen siempre `.1` (VIP de HSRP) y `.2`/`.3` (SVIs de los switches L3).

> **Por qué no alinear la mitad baja con el activo de HSRP:** no compra nada. La elección de servidor DHCP la hace el cliente por el **primer OFFER que llega** — no tiene relación con quién es el gateway. Alinearlo agregaría una regla que hay que recordar a cambio de cero beneficio. Uniforme se audita de un vistazo.

> **El split-scope quema una dirección por DORA `[B8]`.** El servidor que **pierde** la carrera del primer OFFER igual reservó la que ofreció: se vio `MLS2` pasar de 0 a 2 leases tras dos `dhcp -r` que sirvió `MLS1`. Inofensivo a esta escala (122 direcciones útiles por mitad), pero es **el costo "sin estado compartido" hecho visible** — y nadie mira el pool del servidor que no contestó.
>
> **La mitad alta sí ha servido clientes** (`10.2.55.133`/`.134`, Bloque 7): el reparto es **no determinista**, no está sin ejercitar. Que en el B8 `MLS1` ganara las dos carreras es una observación sobre el timing del laboratorio, no sobre el diseño.

**Parámetros comunes a los 4 pools:**

| Campo | Valor |
|---|---|
| `default-router` | **El VIP de HSRP** (`.1`), nunca la SVI física |
| `dns-server` | **`9.9.9.9`** (Quad9) |
| `domain-name` | **No se entrega** |

> **`default-router` = VIP, no SVI:** entregar `.2` o `.3` clava al cliente a un switch concreto y **anula el HSRP para ese cliente** — cae el switch, cae el cliente, aunque el VIP siga vivo. Es el error que invalida el trabajo del Bloque 3 sin que nada se vea roto.

> **Por qué Quad9 y no `1.1.1.1` u `8.8.8.8` (Decisión Bloque 7):** los tres cuestan lo mismo — un campo. Pero `1.1.1.1` y `8.8.8.8` resuelven **cualquier cosa que les pidas, incluido el C2 del malware**. Quad9 rechaza dominios maliciosos conocidos usando feeds de threat intel: si un host pide un dominio de C2, devuelve NXDOMAIN y la conexión nunca se abre. Es el control más barato con mejor retorno de la pila — corta C2, phishing y descarga de payloads sin firewall, sin proxy y sin agente, atacando la cadena en su eslabón más temprano. Se descartó el **DNS diferenciado** (resolver interno para corporativas, público para invitados): se cae al primer contraargumento — *"¿por qué filtras los dominios maliciosos de los invitados pero no los de tu VLAN de operaciones?"* — porque le pondría el mejor control a la red menos crítica. Ver sección 11: en producción, resolver interno con filtrado (Umbrella o similar) **más bloqueo de DoH en el borde**, porque un navegador con DNS-over-HTTPS evade el `dns-server` del DHCP por completo.

> **Sin `domain-name`:** los equipos tienen `ip domain-name australpay.lab`, pero no existe resolución interna que lo use — sería un campo decorativo. Y en los pools de invitados además filtraría el nombre de dominio interno a equipos que no se administran.

**Lease diferenciado:** los invitados llevan lease **corto** (el dispositivo se va y la IP vuelve al pool); las estaciones lease **largo** (si el servidor DHCP cae, siguen funcionando — es lo que hace que el impacto de una caída de DHCP sea diferido).

> **Por qué split-scope y no un servidor único (Decisión Bloque 6):** **IOS no tiene DHCP failover** (Windows Server e ISC sí lo tienen: un protocolo real de sincronización de leases). En un switch Cisco, split-scope es la única redundancia posible. Sus costos se aceptan conscientemente: sin estado compartido (diagnóstico en dos lugares), capacidad a la mitad durante una falla, y asignación no determinista.
>
> **Por qué DHCP no merece la misma redundancia que HSRP:** una caída de DHCP **no es una caída de red** — los clientes con lease vigente siguen operando; solo se afectan los que piden IP nueva. HSRP cae y el tráfico muere al instante. El impacto de DHCP es diferido y menor.
>
> **Equivalente de producción:** las empresas corren DHCP en servidores dedicados (Windows con failover, Infoblox/IPAM) y los switches solo **relayean** con `ip helper-address`. Split-scope en el switch es la respuesta correcta *cuando el switch es el servidor*, que es este caso.

### 4.7 Inventario de loopbacks `[B8]`

Las **únicas cuatro** interfaces `Loopback` de la red. Las seis direcciones de router-id (sección 4.3) **no** están acá: no son interfaces.

| Interfaz | Equipo | Dirección | Trabajo | OSPF |
|---|---|---|---|---|
| `Loopback0` | `BR-STGO` | `192.0.2.100/32` | **Público.** Ancla la VPN (`tunnel source`) · único prefijo anunciado por BGP | No (vive en BGP) |
| `Loopback0` | `BR-VALPO` | `192.0.2.101/32` | Ídem | No (vive en BGP) |
| **`Loopback30`** `[B8]` | `BR-STGO` | `10.1.30.254/32` | **Gestión** · **servidor NTP de toda la red** (sección 7.5) | Área 1 |
| **`Loopback41`** `[B8]` | `BR-VALPO` | `10.2.41.254/32` | **Gestión** · `ntp source` de `BR-VALPO` | Área 2 |

> **Por qué los loopbacks de gestión (B8-#3b).** `BR-VALPO` era **el único de los 8 equipos sin una dirección dentro del rango que el filtro inter-sitio deja cruzar**: sus direcciones internas viven en `10.255.2.x`, y el permit de la sección 7.1 solo deja pasar `10.1.30.0/24 ↔ 10.2.41.0/24`. Desde donde viven los administradores —STGO— el borde de VALPO era **inalcanzable**. Un equipo inadministrable desde el único lugar con administradores no es un control: es un hueco.
>
> **Por qué un loopback y no ensanchar el filtro:** `10.255.2.2` vive en `Gi3`. Si ese uplink cae, **la dirección desaparece** y pierdes la administración del borde justo cuando la necesitas, aunque el equipo esté perfecto y siga alcanzable por `Gi4`. **Es la decisión del Bloque 5 —*"el túnel se ancla a un loopback, no a la interfaz física de un ISP"*— aplicada al plano de gestión.** Y el loopback **no obliga a tocar el filtro**: `10.2.41.254` ya cae dentro del permit existente. La promesa *"Gestión ↔ Gestión"* pasa a ser verdad sin excepciones. **Verificado:** al hacer SSH al loopback, la línea 20 del filtro incrementó y **la 40 no** — si hubiera marcado la 40, el paquete habría caído en "lo no anticipado" y el diseño no habría funcionado como se pensó.

> **`Loopback30` no es el par simétrico de `Loopback41`: es el reloj.** Los **7 clientes NTP apuntan a `10.1.30.254`** (sección 7.5). Si la fuente de tiempo viviera en `10.255.1.2` (`Gi1` de `BR-STGO`), la caída de un uplink de distribución le borraría la hora a los 8 equipos — **justo durante la falla, que es cuando el Bloque 8 demostró que más se necesita correlacionar**. El mismo argumento del Bloque 5, tercera capa: la dirección de un servicio crítico no vive en una interfaz que puede caerse.

> **Caveat declarado — proxy-ARP.** Los loopbacks de gestión **se solapan con la subred de su propia SVI** (`10.1.30.254` está dentro de `10.1.30.0/24`). Desde dentro de la VLAN de gestión, los hosts los alcanzan por **proxy-ARP** (confirmado por `ttl=254`), que se rompería si alguien lo deshabilita. No es grave: desde V41 el borde ya es alcanzable en `10.255.2.2`. Se declara porque una dependencia de proxy-ARP que nadie escribió es una sorpresa esperando fecha.

> **El hueco que ningún loopback cierra:** administrar un borde **a través del túnel que ese mismo borde termina** es circular — si está sano no necesitas entrar con urgencia; **si está roto, el túnel está caído y ninguna ACL te salva**. La respuesta real es acceso **out-of-band**: sección 11.

---

## 5. Diseño de enrutamiento

### 5.1 OSPF (interior)

Diseño multiárea con los routers de borde como **ABR**:

| Área | Contenido |
|---|---|
| **Área 0** (backbone) | Túnel GRE-over-IPSec entre `BR-STGO` y `BR-VALPO` |
| **Área 1** | Interior de STGO (SVIs de distribución, enlace borde↔distribución) |
| **Área 2** | Interior de VALPO (SVIs, enlace borde↔distribución) |

El multiárea aquí **no es decorativo**: cada sitio es su propia área y los bordes son ABR reales porque tienen una pata en el backbone (túnel) y otra en el área del sitio — confirmado en el Bloque 5 y re-verificado tras la reconstrucción del Bloque 6 (3 adyacencias `FULL` por borde: 2 hacia distribución + 1 hacia el otro borde sobre `Tunnel0`).

Las SVIs de usuario se anuncian con `network` pero **no forman adyacencia** (`passive-interface default` + VLAN 99 dedicada al control-plane) — por eso una ACL en una SVI de usuario **no toca OSPF**. Sí toca HSRP: ver sección 7.2.

Ambos bordes inyectan una **ruta por defecto** hacia el interior (`default-information originate`, ver 5.3). Esa default **se aprende por eBGP desde ambos ISP** `[B8]`; **ya no hay ruta estática que la respalde** — ver 5.3 y changelog B8-#4.

### 5.2 BGP (borde)

| AS | Entidad |
|---|---|
| `65010` | AustralPay (`BR-STGO`, `BR-VALPO`) — un solo AS para toda la empresa |
| `65001` | `ISP-1` — doble rol: ISP-A (hacia STGO) e ISP-C (hacia VALPO) |
| `65002` | `ISP-2` — doble rol: ISP-B (hacia STGO) e ISP-D (hacia VALPO) |

> **Cambio respecto al diseño original (ver changelog Bloque 5):** se planificaban 4 AS distintos (65001-65004), uno por rol de ISP. En la ejecución se colapsaron a 2 AS, cada uno sirviendo a ambos sitios — decisión tomada bajo el filtro función > seguridad > simplicidad: la diversidad de camino real (requisito de multihoming genuino) se mantiene intacta con 2 AS, y la resiliencia *per-sitio* tampoco se pierde. Lo único que se resigna es el aislamiento de fallos *entre sitios*, una propiedad que en la práctica el cliente tampoco controla (los ISP disponibles en una región son los que son).

- **eBGP:** `BR-STGO`↔`ISP-1`, `BR-STGO`↔`ISP-2`, `BR-VALPO`↔`ISP-1`, `BR-VALPO`↔`ISP-2`.
- **`allowas-in 1`** en los 4 vecinos de los bordes: necesario porque ambos sitios comparten AS 65010 y los mismos dos ISP de tránsito — sin esto, la prevención de loop de BGP descarta el anuncio de un sitio al llegarle de vuelta con su propio AS ya en el path. Ver changelog para el detalle (elegido sobre separar AustralPay en dos AS, que habría convertido la relación entre bordes de iBGP a eBGP).
- **iBGP: NO se implementa (Decisión Bloque 6).** BGP anuncia **solo 2 prefijos** (los loopbacks públicos /32 de la sección 4.5). Cada borde **ya aprende el loopback del otro por eBGP** vía los ISP (gracias a `allowas-in`); el transporte interno RFC 1918 va por **OSPF sobre GRE**; y la salida a internet es por `default-information originate`, no por tabla BGP. Una sesión iBGP entre bordes cargaría exactamente los mismos 2 prefijos que ya se ven por eBGP: **redundante**. Tampoco podría servir para alcanzar el destino del túnel — el túnel se ancla en esos loopbacks, así que no puede aprenderlos por encima de sí mismo. Ver changelog.
- **Manipulación de atributos (ambas sedes, Bloque 5):** `ISP-1` como upstream primario en ambas sedes. **Local Preference** 200 (ISP-1) vs 100 (ISP-2) controla el tráfico **saliente**. **AS-path prepend** (`65010 65010`, doble) hacia ISP-2 influye el tráfico **entrante**, desincentivando ese camino sin bloquearlo. Verificado con `show ip bgp` (ISP-1 marcado `best`) y `traceroute` real.
- **Default por BGP** `[B8]`: los dos ISP fabrican `0.0.0.0/0` con `neighbor <ip> default-originate` hacia los **4 vecinos de AustralPay**. **No entre ellos**: un `default-originate` en el peering `ISP-1`↔`ISP-2` sería un **loop de default**. Ver 5.3.
- **Filtro de salida** `[B8]`: prefix-list **`ANUNCIO-PROPIO`** (el loopback público propio, y nada más) con `match ip address prefix-list` en los route-maps `PASS-OUT` y `PREPEND-ISP2-OUT` de los 4 vecinos. El `deny` implícito del route-map hace el resto.

> **Las tres formas de anunciar una default, y por qué solo una sirve acá `[B8]`:**
>
> | Forma | Qué exige | Sirve para un ISP de tránsito |
> |---|---|---|
> | `network 0.0.0.0` | **Tener** la default en la RIB | ❌ |
> | `default-information originate` (BGP) | Redistribuir una default **existente** | ❌ |
> | **`neighbor <ip> default-originate`** | Nada — **la fabrica**, por vecino | ✅ |
>
> Un ISP de tránsito **no tiene** ruta por defecto: *él es* el destino por defecto. Las dos primeras anuncian una default que ya tienes; la tercera la **fabrica**. **Frase de entrevista:** *"`network` y `default-information originate` anuncian una default que ya tienes; `neighbor default-originate` la fabrica. Un ISP de tránsito usa la tercera, porque él es el destino por defecto — no tiene uno."*

> **El tránsito accidental (B8-#6) — el hallazgo existía desde el Bloque 5.** `PASS-OUT` era `permit 10` **sin `match`**, o sea *"anuncia todo lo que sepas"*. Cada borde le anunciaba a cada ISP **7 prefijos**: el suyo más los **6 aprendidos del otro ISP**, re-anunciados con `65010` al frente del AS-path — que en BGP significa literalmente *"puedes alcanzar esto a través de mí"*. **Incluida `0.0.0.0/0`**: AustralPay ofreciéndole una ruta por defecto a un AS de tránsito.
>
> **Un cliente multihomed que no filtra su salida se anuncia como tránsito entre sus dos upstreams**, y es la causa raíz de una familia entera de incidentes reales de BGP: una red chica anuncia lo que aprendió, un ISP grande lo cree, y se traga tráfico que jamás podría cursar. El leak era **invisible mientras no hubo nada que filtrar** — nació con `allowas-in` en el B5 y nadie miró la salida.
>
> **La corrección son 3 líneas por borde**, y el efecto es de 7 prefijos a **1** en las cuatro sesiones. Detectado con `show ip bgp neighbors <ip> advertised-routes` — **el comando que nadie corre**: todo el mundo mira lo que aprende, casi nadie lo que anuncia.

### 5.3 Salida a internet (por qué NO redistribución mutua BGP↔OSPF)

Los bordes inyectan una **ruta por defecto en OSPF** (`default-information originate`) hacia el interior. Esa default **la aprenden por eBGP de ambos ISP** (`neighbor default-originate`, sección 5.2) y **la LocalPref decide cuál instalan** `[B8]`. **No se redistribuye la tabla BGP completa a OSPF** — meter la tabla de internet en el IGP nunca se hace en producción. Con NAT en el borde, el interior sale enmascarado; no hay necesidad de anunciar redes internas al ISP.

**Sin `always`, deliberadamente:** `default-information originate` **retira su LSA** si el borde se queda sin default en la RIB. Es lo correcto: si el borde realmente no tiene salida, el interior debe **enterarse** en vez de seguir mandándole tráfico a un agujero.

> **Cambio del Bloque 8 — de estática a BGP (B8-#4, el hallazgo más grande del bloque).** Hasta el B7 la default venía de **una ruta estática hacia `ISP-1`** en cada borde. Ante la caída de `ISP-1`: la conectada desaparece → el next-hop no resuelve → **la estática se cae de la RIB** → `default-information originate` **retira su LSA** → el interior pierde la default. **Internet muere** (el túnel no: sección 6.4). Es la misma clase de falla que la sección 7.0: el documento promete una propiedad **global**, la config la entrega en un **subconjunto**, y nadie lo nota porque lo configurado se ve perfecto.
>
> **Se eligió la default por BGP sobre una estática flotante a `ISP-2` (AD 10), por tres razones:**
> 1. **Convierte el trabajo del Bloque 5 en portante.** La LocalPref 200/100 gobernaba **2 prefijos /32** que casi no llevaban tráfico. Ahora decide por dónde sale **todo el internet de la sede** — que es literalmente para lo que existe el atributo. *"Manipulé LocalPref"* pasa de anécdota a **política de negocio implementada**.
> 2. **Cubre un modo de falla que la flotante no cubre.** BGP detecta por **teardown de sesión**, no solo por link-down. *"ISP muerto con el cable arriba"* es real, es el que agujerea en silencio, y **el laboratorio lo reprodujo sin querer**: el `shutdown` de un ISP **no bajaba el link del borde** (artefacto de EVE-NG, sección 11.2), así que la falla se detectó por expiración del hold timer — exactamente el escenario que una flotante atada a link-down **nunca habría visto**.
> 3. **Es lo que hace una empresa multihomed de verdad.** Clavar una estática al primario es precisamente lo que el multihoming venía a evitar.
>
> **El orden de operaciones es la parte que no se improvisa:** (1) `default-originate` en los ISP → **impacto cero**, porque la distancia administrativa deja la default de eBGP (20) fuera de la RIB mientras exista la estática (1) — y IOS lo dice en voz alta: `RIB-failure(17)` / *"Higher admin distance"*, o sea *"la aprendí, la elegí como mejor camino de mi protocolo, y no la instalé"*; (2) NAT con route-map → **impacto cero**, el ruteo sigue mandando todo por ISP-1; (3) **retirar la estática** ← el único paso con riesgo, y llega cuando 1 y 2 están verificados. **Al revés te quedas sin salida.**
>
> **El paso 3, medido:** **cero paquetes perdidos**, y las LSA del interior con su timestamp **intacto** (`13:50:19 ago` en DLS1, `1d04h ago` en MLS1). Se cambió el protocolo que sostiene la salida a internet de dos sedes **y el IGP no registró ni un evento**.
>
> **Nota de lectura:** un prefijo en `RIB-failure` **se sigue anunciando** — BGP anuncia desde su tabla, no desde la RIB. La AD decide **quién entra a la RIB**, no quién es mejor dentro de su propio protocolo.

> **El `Tag` como instrumento `[B8]`:** la default trae **el AS de origen tatuado**. `show ip route 0.0.0.0` dice `Tag 65001` o `Tag 65002` — **por qué ISP salió el tráfico, en un solo `show`, sin traceroute**. Y el `ttl` del ping lo confirma desde el otro lado: `253` vía ISP-1, **`252` vía ISP-2** (un salto más, el peering ISP-2→ISP-1).

> **Punto de entrevista:** esto es lo *contrario* a lo que pedía la pauta del ramo (redistribución mutua). Aquí se hace lo correcto para un diseño real, y saber *por qué* no redistribuir demuestra criterio, no desconocimiento.

### 5.4 NAT

PAT (overload) en cada borde, **un statement por ISP, basado en route-map** `[B8]`:

| Borde | Statement | Traduce a |
|---|---|---|
| `BR-STGO` | `ip nat inside source route-map NAT-ISP1 interface Gi2 overload` | `203.0.113.2` (ISP-1) |
| `BR-STGO` | `ip nat inside source route-map NAT-ISP2 interface Gi3 overload` | `203.0.113.6` (ISP-2) |
| `BR-VALPO` | `NAT-ISP1` → `Gi1` · `NAT-ISP2` → `Gi2` | `198.51.100.2` · `198.51.100.6` |

Cada route-map lleva **dos `match`**: `match ip address NAT-EXCLUDE-VPN` (**qué** se traduce) y `match interface GiX` (**solo si el ruteo decidió que sale por ahí**).

La ACL `NAT-EXCLUDE-VPN` **excluye** el tráfico hacia el otro sitio (`10.1.0.0/16`↔`10.2.0.0/16`), para que el tráfico entre sitios viaje cifrado por el túnel con IP interna real y no salga natteado — verificado con tráfico real en el Bloque 5, y **aislado con una sola variable en el Bloque 8**: desde `V41A`, hacia internet → **10 traducciones**; hacia `10.1.30.11` → **cero**. Mismo host, mismo borde, misma config de NAT; la única diferencia es el destino, o sea `NAT-EXCLUDE-VPN`. No queda otra explicación posible.

> **Por qué route-map y no `interface Gi2 overload` a secas (B8-#4).** En la forma basada en lista, la interfaz **no es una condición: es el resultado**. `ip nat inside source list X interface Gi2 overload` significa *"traduce lo que matchee X, **siempre** a la IP de Gi2"* — no pregunta por dónde sale el paquete, solo de dónde saca la IP global.
>
> **Con multihoming eso falla peor que no natear:** si el ruteo manda el paquete por `Gi3`, el statement lo traduce igual a la IP de `Gi2` → **un prefijo sin camino de vuelta**. El tráfico sale, nadie vuelve, y **todos los `show` se ven bien**. Con route-map la interfaz pasa a ser **condición**: el NAT deja de tener opinión propia y **sigue a la tabla de ruteo**, que es lo que debe hacer.
>
> **Frase de entrevista:** *"En NAT basado en lista, la interfaz es de dónde sacas la IP global. En NAT basado en route-map, es una condición del match. Con multihoming necesitas la segunda: si el NAT no sigue al ruteo, traduce a un prefijo muerto y el retorno nunca llega."*
>
> **Caveat declarado:** al conmutar de ISP, las traducciones **ya existentes** quedan atadas a la IP global vieja hasta expirar. Afecta a TCP; con ICMP no es observable — ver 11.1.

Como la exclusión se expresa a nivel de `/16` por sitio, las VLANs agregadas en el Bloque 7 (15, 50, 55) quedan cubiertas automáticamente: salen a internet por PAT y nunca se natean hacia el otro sitio, **sin cambios en los bordes**.

> **Nota de diagnóstico (Bloque 6):** un `ping` originado **desde la consola del propio borde** hacia una red interna sale por una interfaz `ip nat inside` y es candidato a traducción — caso borde de IOS que descarta el paquete o rompe su retorno. El tráfico de **tránsito** no sufre esto. Al diagnosticar, usar un host real como origen, no la consola del router NAT.

---

## 6. Redundancia

### 6.1 HSRP

Activo/standby por VLAN con **reparto de carga** (se alterna el activo por VLAN), en **ambas sedes**.

**STGO:**

| VLAN | VIP | `DLS1` | `DLS2` | Activo |
|---|---|---|---|---|
| 10 — Servidores-App | `10.1.10.1` | `.2` | `.3` | DLS1 |
| **15 — Servidores-DB** `[B7]` | `10.1.15.1` | `.2` | `.3` | **DLS1** |
| 20 — Operaciones | `10.1.20.1` | `.2` | `.3` | DLS2 |
| 30 — Gestión | `10.1.30.1` | `.2` | `.3` | DLS1 |
| **50 — Invitados** `[B7]` | `10.1.50.1` | `.2` | `.3` | **DLS2** |

**VALPO:**

| VLAN | VIP | `MLS1` | `MLS2` | Activo |
|---|---|---|---|---|
| 40 — Oficina | `10.2.40.1` | `.2` | `.3` | MLS1 |
| 41 — Gestión | `10.2.41.1` | `.2` | `.3` | MLS2 |
| **55 — Invitados** `[B7]` | `10.2.55.1` | `.2` | `.3` | **MLS2** |

> **Por qué el reparto difiere entre sedes `[B8]`.** En STGO el activo de `DLS1` agrupa **V10, V15 y V30**, y en VALPO el de `MLS2` agrupa **V41 y V55**: **Gestión queda con los servidores en STGO y con los invitados en VALPO**. No es una asimetría accidental ni una regla distinta: **en STGO el reparto lo condiciona una adyacencia de aplicación** —V10 y V15 son la conversación app→db y se mantienen en el mismo switch (sección 8)—, y las VLAN restantes caen del otro lado. **VALPO no tiene servidores**, así que no hay adyacencia que respetar y el reparto responde **solo al balance de carga**, que es la regla general enunciada arriba. La forma del bloque es idéntica; lo que cambia es que en una sede hay una restricción adicional y en la otra no. Ver el diagrama `ARQ-003`.

**Convención común a las 8 VLANs (cerrada en el Bloque 7):**

| Parámetro | Valor | Por qué |
|---|---|---|
| Versión | **HSRPv2** | Soporta 4096 grupos (v1: 256), timers en ms, autenticación MD5 e IPv6. v1 no tiene ninguna ventaja sobre v2: un diseño nuevo no tiene razón para arrancar en el estándar viejo. |
| Número de grupo | **= número de VLAN** | La MAC virtual codifica el grupo (`0000.0c9f.f0XX`): ver `f028` en una tabla MAC dice "VLAN 40" al instante. Con grupo `1` diría `f001` y no significaría nada. Diagnóstico gratis. |
| Prioridad | **105** (activo) / **100** (standby, implícito) | La prioridad es **relativa al decremento**, no absoluta: 105−10 = 95 < 100 cruza el umbral. Con 150 el activo quedaría en 140 y **el track no conmutaría nunca**. |
| Tracking | `track 1 interface <uplink> line-protocol` + `standby X track 1 decrement 10` | Un objeto por switch, compartido por todos sus grupos. Enhanced Object Tracking; el `standby track <interfaz>` directo no existe en esta plataforma. |
| Preempt | `preempt delay minimum 30` | Ver abajo. |

> **Por qué `preempt delay minimum 30` (Decisión Bloque 7):** sin él, un switch que **reinicia** levanta sus SVI, preemptea en ~1 s y se vuelve gateway activo **con la tabla de ruteo vacía** — agujero negro de 30-40 s hasta que OSPF converge desde cero. De paso amortigua los flaps del uplink al recuperarse (observados en el lab: `Up→Down→Up` en 2 s). El costo es una recuperación más lenta, que es un evento raro. Se aplica también en los grupos donde el switch es standby y el delay nunca dispara: config simétrica en los cuatro switches, sin excepciones que recordar.

> **Alcance real del tracking (Bloque 7, corregido en el Bloque 8):** el track **no evita un agujero negro, evita un camino subóptimo** — y el B8 midió que **tampoco evita el hairpin**. Lo que lo justifica es **una sola** de sus dos razones declaradas: ante una **segunda falla** (peer-link) sí habría agujero negro. En un diseño con peer-link solo L2, el track sería la diferencia entre servicio y caída. **El track vale, por una de sus dos razones.**

> **La razón que se cayó, y por qué (4ª divergencia, Bloque 8).** Este documento afirmaba que el track *"mantiene el tráfico directo por el switch que sí tiene uplink"*. **Es falso, y la culpa es de STP.**
>
> Tras el failover, el `traceroute` desde `S10A` sale efectivamente por DLS2 (`10.1.10.3`, el nuevo activo) — pero la tabla MAC de `ALS1-STGO` muestra la vMAC (`0000.0c9f.f00a`) aprendida **por `Po2`, hacia DLS1**. `DLS1` **sigue siendo root de V10**: el puerto de ALS1 hacia DLS2 está **`BLK`**. Camino real: `S10A → ALS1 → DLS1 → peer-link → DLS2 → BR-STGO`. **La trama entra por el switch que perdió su uplink, cruza el peer-link, y se rutea en el otro.**
>
> **Con track o sin track cruza el peer-link igual.** Sin track, DLS1 seguiría activo y rutearía por la adyacencia OSPF de VLAN 99 hacia DLS2: **un salto de peer-link en ambos casos**. Solo cambia si va **bridgeado** (V10) o **ruteado** (V99).
>
> **No es un error de configuración: es un límite del diseño.** Alinear root con activo optimiza el **estado normal** —y eso el B8 lo midió, no lo supuso: los contadores de las ACL incrementaron **solo en el switch activo** de cada VLAN—, pero **en estado degradado la alineación se rompe sola**, porque solo uno de los dos mecanismos se movió: HSRP conmuta, STP no. La solución sería que STP conmutara también (root secundario): **~30 s de reconvergencia para ahorrar un salto de peer-link. No vale la pena — y saber decir eso es el entregable.**

> **Por qué V15 va en DLS1 (junto a V10) y no alternada:** el reparto por VLAN busca equilibrio, pero **V10 y V15 son la conversación app→db**. Si quedaran en switches distintos, ese tráfico haría *hairpin* (ALS1 → DLS1 → peer-link → DLS2 → ALS1), porque el puerto de ALS1 hacia el switch no-root estaría bloqueado por STP para esa VLAN. Juntarlas mantiene el ruteo local. El equilibrio se conserva **por tipo de tráfico**: DLS1 lleva servidores + gestión, DLS2 lleva usuarios + invitados.

### 6.2 EtherChannel
- **Acceso↔distribución**: troncales L2 agregadas con **LACP**.
- **Peer link** entre los switches de distribución (STGO y VALPO): troncal L2 con LACP (permite el span de VLANs y los hellos de HSRP).
- **Borde↔distribución**: enlace **L3** dual-homed (dos /30 independientes por sitio), con `ip ospf network point-to-point` en ambos extremos.

### 6.3 RPVST+ (alineado con HSRP)

Root bridge planificado **por VLAN** y alineado con el activo de HSRP, para que el forwarding L2 y L3 coincidan y no haya *hairpinning* **en estado normal** — medido en el Bloque 8. **En estado degradado la alineación se rompe sola** (HSRP conmuta, STP no) y el hairpin ocurre igual: ver la corrección de la sección 6.1.

**STGO:**

| VLAN | Root primario | Root secundario |
|---|---|---|
| 10 | DLS1 | DLS2 |
| **15** `[B7]` | **DLS1** | DLS2 |
| 20 | DLS2 | DLS1 |
| 30 | DLS1 | DLS2 |
| **50** `[B7]` | **DLS2** | DLS1 |

**VALPO:**

| VLAN | Root primario | Root secundario |
|---|---|---|
| 40 | MLS1 | MLS2 |
| 41 | MLS2 | MLS1 |
| **55** `[B7]` | **MLS2** | MLS1 |

`PortFast` + `BPDUGuard` en puertos de acceso.

> **Nota sobre el conteo de instancias:** RPVST+ corre una instancia por VLAN. Con las VLANs del Bloque 7, STGO pasa de 4 a 6 instancias y VALPO de 3 a 4 — muy por debajo de donde MST empezaría a justificarse. La decisión de RPVST+ sobre MST (Bloque 2) se mantiene sin cambios.

### 6.4 Resiliencia de la VPN (WAN, Bloque 5)

El túnel GRE-over-IPSec ancla su origen y destino a loopbacks públicos anunciados por BGP a ambos upstreams (secciones 4.5 y 4.7) — la caída de `ISP-1` o `ISP-2` no tumba el túnel, BGP reconverge automáticamente hacia el ISP disponible sin intervención manual.

**Verificado en el Bloque 8** `[B8]`, con `ISP-1` caído (su interfaz hacia `BR-STGO` en `shutdown`) y ping continuo corriendo: **el túnel sobrevivió**. El corte de **129 s** que se midió fue de **salida a internet** (sección 5.3), no del túnel, y el control lo confirma desde el otro lado: `V40A` mantuvo **383/383** paquetes durante las dos pruebas de ISP — la falla no se propagó a VALPO. **Esta sección es de las pocas afirmaciones que el bloque de validación confirmó en vez de desmentir**: el multihoming sí protegía la VPN. Lo que no protegía era el negocio (sección 5.3).

> **Un dato contra la predicción:** se predijo que el túnel recuperaría **sin re-formar** su adyacencia OSPF. **Se reformó las dos veces** (`19:32:15`, `20:38:12`). **Cuánto tardó** en re-formarse no se pudo medir: no había reloj común entre los equipos. Ese hallazgo es el que motivó la sección 7.5 — con hora común, ese mismo log ya sirve. Ver 11.2.

---

## 7. Seguridad

### 7.0 Endurecimiento de capa 2 (base, Bloque 2/4)

- **VLAN nativa dedicada (999) sin uso** en todas las troncales → mitiga **double tagging**.
- **DTP deshabilitado en las troncales *y* en los puertos no usados** `[B8]` (`switchport mode access` + `switchport nonegotiate`); troncales configuradas manualmente. La política de puerto no usado está en 7.4.
- **Poda de VLANs** en las troncales (solo se permiten las VLANs necesarias).
- **`vtp mode transparent` en los 6 switches** `[B8]` → las VLANs viven en el `running-config`, no solo en `vlan.dat`: **el archivo reconstruye el equipo**. Un switch en VTP server además adopta el primer dominio que le llegue por una troncal y puede perder VLANs.
- **NAT con exclusión de tráfico VPN** (sección 5.4): el tráfico inter-sitio nunca se natea.

> **Corrección del Bloque 8 (1ª divergencia).** *"DTP deshabilitado"* **era falso**. El `switchport nonegotiate` se aplicó a las troncales —donde se pensó el control— y **nunca tocó lo que nadie configuró**: los **12 puertos libres** de `DLS1`/`DLS2`/`MLS1`/`MLS2` estaban en `dynamic auto` · `Negotiation: On` · `Trunking VLANs Enabled: ALL`. Un puerto en `dynamic auto` no inicia negociación pero **acepta** convertirse en troncal si el otro lado la pide (*switch spoofing*). Corregido en el B8 → sección 7.4. **Ahora la frase es verdad: troncales y puertos no usados.**
>
> **Es el patrón que este bloque encontró once veces:** el documento declara una propiedad **global**, la ejecución la aplica donde se **pensó**, y lo que nadie tocó se queda con su **default**. Nadie lo nota porque **lo configurado se ve perfecto**.

> **`vtp mode transparent` (9ª divergencia, Bloque 8).** `ALS1-VALPO` era el único de los 6 en **VTP server** — el default de la plataforma. Sus VLANs vivían solo en `vlan.dat`, así que **su `running-config` no reconstruía el equipo**: el archivo de entrega no listaba **ni una VLAN**. Es una divergencia distinta a las demás: no rompía nada en operación, **rompía la recuperación**. Y este proyecto ya reconstruyó dos equipos desde archivos (Bloque 6).

### 7.1 Filtro de seguridad inter-sitio (Bloque 5)

Únicamente el tráfico **Gestión-VALPO (VLAN 41) ↔ Gestión-STGO (VLAN 30)** puede cruzar entre sitios; todo el resto de tráfico inter-sitio queda bloqueado por defecto.

- **Mecanismo:** ACL extendida (`INTERSITE-FILTER-IN`) aplicada `in` sobre `Tunnel0` en ambos bordes — el único punto de tránsito posible entre área 1 y área 2 (vía área 0), coherente con el principio ya aplicado a la VLAN 99 y `passive-interface default` (filtrar en el punto de convergencia único, no duplicar en cada extremo).
- **Regla no negociable:** el primer `permit` cubre explícitamente OSPF (`permit ospf any any`) — de lo contrario el `deny` final tumbaría los propios hellos de área 0 entre bordes.
- **Naturaleza del control:** ACL extendida clásica, **stateless**. La confidencialidad ya la provee IPSec; esta ACL es de **autorización** (quién habla con quién), no de cifrado.
- **Cobertura gratuita de las VLANs del B7:** el `deny` de esta ACL está escrito a nivel de `/16` por sitio, así que Invitados (V50/V55) y Servidores-DB (V15) quedan bloqueados entre sedes **sin cambios en los bordes**.
- **`40 deny ip any any log` al final de la lista** `[B8]`: el `deny` de la línea 30 captura **lo que el diseño anticipó** (`10.2.0.0/16 → 10.1.0.0/16`); la línea 40 captura **lo que no anticipó**. El `deny` implícito de IOS **no cuenta y no loguea**.
- **Verificado con tráfico real:** Gestión-VALPO→VIP Gestión-STGO 5/5 (permit); Oficina-VALPO→VIP Gestión-STGO timeout (deny, logueado).
- **Consecuencia aceptada (Bloque 6, matizada en el B8):** un ping directo entre las IPs internas del túnel (`10.255.0.1`↔`10.255.0.2`) **no matchea** ningún permit → cae en **la línea 40** y falla, **ahora dejando rastro**. Se mantiene la decisión de **no agregar una excepción**: el diagnóstico de salud del túnel pasa por la **adyacencia OSPF en FULL** (que no sube si el túnel no transporta tráfico real), no por ping directo. Lo que cambió no es la política: es que el drop **es visible**. Verificado en el B8 con las dos cosas lado a lado: adyacencias `FULL` y `ping 10.255.0.1` **0/5** — el chequeo de salud correcto y el incorrecto, en la misma captura.

> **Por qué la línea 40, y por qué `log` acá sí (B8-#2).** Antes del Bloque 8, ese ping **desaparecía sin dejar ni un contador ni una línea de syslog en ningún equipo**. Es al revés de lo que quieres: **el tráfico raro es justo el que tiene señal**.
>
> El criterio de la sección 7.2 —*"un log vale cuando su volumen esperado es cero"*— se cumple acá **mejor que en ninguna otra lista**: el túnel es un **canal autenticado por IPSec**, solo llega lo que el otro borde cifró. No hay internet abierto detrás, así que el riesgo de flood que impide loguear en las listas expuestas **no existe en esta**.
>
> **Se pagó sola en 15 minutos y atrapó tres flujos legítimos el mismo día:** (1) el SSH de `ALS1-STGO` al loopback de gestión de `BR-VALPO` (sección 4.7) — sin la línea 40, drop silencioso y tú depurando el SSH en el equipo equivocado; (2) los unreachables del filtro espejo; (3) **16 consultas DNS por broadcast** desde `BR-STGO` hacia los ISP y hacia la VPN (sección 7.3). **Los tres eran invisibles.**
>
> **El mecanismo que explica `timed out` en vez de `refused`**, confirmado con IP, puerto y código en los dos logs: `BR-VALPO` deniega el SSH → genera un unreachable *type 3 code 13* **sourceado de `10.255.0.2`** → cruza el túnel → **`BR-STGO` lo deniega**, porque `10.255.0.2 ∉ 10.2.0.0/16`. **La notificación del deny muere en el mismo control que produjo el deny.**

### 7.2 Política de segmentación intra-sitio `[B7]`

**El hallazgo (Bloque 6):** hasta este punto el proyecto filtraba el tráfico **inter-sitio** —el que cruza cifrado por un túnel, el camino *más* protegido— y dejaba **completamente abierto el intra-sitio**, donde vive la mayoría del tráfico real. En una fintech, cualquier estación de trabajo alcanzaba la base de datos sin nada que lo impidiera. Es el patrón clásico: perímetro protegido, interior plano.

**Postura: lista blanca (default-deny).** Con lista negra solo se bloquea lo que se pensó; con lista blanca, **lo que se olvidó queda bloqueado**. Falla cerrado.

#### Matriz de política — STGO

✅ permitido · ❌ denegado · ↩️ solo tráfico de respuesta (`echo-reply`)

| Origen ↓ / Destino → | V10 App | V15 DB | V20 Oper | V30 Gest | V50 Inv | Internet |
|---|---|---|---|---|---|---|
| **V10 App** | — | ✅ | ↩️ | ↩️ | ❌ | ✅ |
| **V15 DB** | ↩️ | — | ❌ | ↩️ | ❌ | ❌ |
| **V20 Oper** | ✅ | ❌ | — | ↩️ | ❌ | ✅ |
| **V30 Gest** | ✅ | ✅ | ✅ | — | ✅ | ✅ |
| **V50 Inv** | ❌ | ❌ | ❌ | ↩️ | — | ✅ |

#### Matriz de política — VALPO

| Origen ↓ / Destino → | V40 Oficina | V41 Gest | V55 Inv | Internet |
|---|---|---|---|---|
| **V40 Oficina** | — | ↩️ | ❌ | ✅ |
| **V41 Gest** | ✅ | — | ✅ | ✅ |
| **V55 Inv** | ❌ | ↩️ | — | ✅ |

**En palabras:** las estaciones hablan con la aplicación, **nunca** con la base de datos. La app es la única que toca la DB. La DB **no inicia nada** — solo responde. Gestión administra todo. Invitados solo internet, y solo pueden **responder** al admin.

> **Corrección Bloque 7 — `V20 → V30` pasó de ❌ a ↩️.** La celda original era la **única sin su ↩️ en toda la red** (los pares V30↔V10, V30↔V15, V30↔V50, V41↔V40 y V41↔V55 sí lo tenían) y contradecía el texto de esta misma sección: con `V20 → V30 ❌`, *"Gestión administra todo"* era falso — gestión no podía administrar las estaciones, que es la VLAN que más se administra en cualquier red real. Se detectó ejecutando la matriz en el lab: `S30A → S20A` daba **timeout** en vez de `code 13`, señal de que el paquete salía y llegaba pero **la respuesta** moría en `POL-V20-OPER`. Ver changelog.

#### Ubicación y estructura

- **`in` en la SVI de origen**, una ACL por VLAN → cada lista se lee como *"qué puede hacer esta VLAN"* y mapea 1:1 con una **fila** de la matriz, dejando la config auditable contra este documento.
- **En la SVI de AMBOS switches L3** (DLS1 *y* DLS2; MLS1 *y* MLS2). Con HSRP cualquiera puede ser el activo: si la ACL va solo en el activo, **la política se evapora durante el failover**.
- **ACL `out` adicional en la SVI de V15 (DB)** — lista blanca explícita de quién puede alcanzarla (V10 App, V30 Gestión; deny el resto). El esquema basado en origen **falla abierto**: una VLAN nueva sin ACL quedaría permitida hacia todo. El activo de mayor valor lleva doble candado: restringir orígenes **y** proteger el activo.
- **V30 y V41 (Gestión) NO llevan ACL** `[Decisión B7]`. Su fila de la matriz es todo-permitido: la lista sería literalmente `permit ip any any`. **La ausencia de ACL *es* esa política.** Aplicar un `permit ip any any` explícito sería ceremonia que consume CPU por cero enforcement. Se documenta para que la ausencia no se lea como omisión.

#### Nomenclatura y logging

Listas nombradas, patrón `POL-V<vlan>-<rol>`: `POL-V10-APP`, `POL-V15-DB`, `POL-V15-DB-ACCESO` (out), `POL-V20-OPER`, `POL-V50-INV` en STGO; `POL-V40-OFICINA`, `POL-V55-INV` en VALPO. **Siete listas, cada una en los dos switches de su par** `[B8]`.

> **Corrección del Bloque 8 (11ª divergencia): eran nueve, son siete.** El número **nueve** calzaba exacto con las **8 filas de la matriz más la lista `out` de V15** — o sea, era correcto **antes** de la decisión *"V30 y V41 no llevan ACL"*, tomada en el mismo bloque y en **esta misma sección**. 9 − 2 = 7, y nadie recontó. Es el patrón del bloque en su versión más barata: **el documento declara un número, una decisión posterior lo invalida, y el número viejo sobrevive porque contar de nuevo no se le ocurre a nadie.** A un entrevistador con la config al lado, sí.

**`log` solo en tres líneas, todas con volumen normal esperado cero** `[Decisión B7]`:

| Lista | Línea | Qué captura |
|---|---|---|
| `POL-V20-OPER` / `POL-V50-INV` | `deny ip any 10.1.15.0 0.0.0.255 log` (**antes** del deny genérico) | **Quién intentó llegar a la DB**, con IP de origen |
| `POL-V15-DB` (in) | `deny ip any any log` | **La DB iniciando algo** — indicador clásico de exfiltración o C2 |
| `POL-V15-DB-ACCESO` (out) | `deny ip any any log` | **Cable trampa**: hoy no puede dispararse (todo lo que podría ya muere en su propio ingreso). Grita el día que alguien cree una VLAN sin ACL |

> **Por qué solo esas tres:** un `log` vale cuando su volumen esperado es **cero** — ahí cada evento es señal. En las demás listas el volumen sería ruido sin valor, y cada paquete denegado se puntea a CPU. En producción, con rate-limit.
>
> **Los eventos de ACL son agregados, no uno por paquete:** IOS emite el primer paquete al instante y sumariza el resto en una ventana de 5 minutos (`1 packet` … `4 packets`). Es rate-limiting deliberado — sin él, un escaneo inundaría el syslog y el `log` sería un vector de DoS contra el propio colector. Una correlación que cuente eventos en vez de leer el campo `N packets` **subcontará por un orden de magnitud**.

#### Reglas obligatorias en las listas

- **Permit de HSRP donde la lista termine en `deny any`.** Los hellos del switch vecino **entran por la SVI** y una ACL entrante los filtra. La red corre **HSRPv2** (sección 6.1) → multicast **`224.0.0.102`**, UDP 1985. Sin `permit udp any host 224.0.0.102 eq 1985` antes del deny, cada switch deja de ver al otro y **ambos se creen Activos** (*split-brain*). Afecta a **V15**, la única lista que termina en `deny any` (las demás cierran con `permit ip any any` para la salida a internet, y el HSRP pasa de rebote). Es el mismo bug que el `permit ospf any any` de la sección 7.1 ya evita.
  > **Verificado en el Bloque 7 desde los dos lados:** `POL-V15-DB` línea 10 acumuló **570 matches** (el permit explícito haciendo su trabajo) y `POL-V50-INV` línea 50 (`permit ip any any`) acumuló ~500 (los mismos hellos entrando de rebote). Los dos contadores prueban las dos mitades de la regla.
- **OSPF no necesita permit en ninguna lista.** Todas las SVI de usuario son pasivas (`passive-interface default`, con solo VLAN 99 y los uplinks exceptuados) — no hay hellos de OSPF entrando por ellas.
- **`deny ip any 10.0.0.0 0.255.255.255` antes del `permit ip any any` final.** Sin ese deny intermedio, el `permit ip any any` que habilita la salida a internet dejaría pasar **todo el tráfico interno** y la matriz se caería entera.
  > **Efecto esperado, no es un bug:** el VIP de cada VLAN está dentro de `10.0.0.0/8`, así que **ningún host de V10, V20, V40, V50 o V55 puede pinguear su propio gateway**. El tráfico *a través* del gateway hacia internet sí funciona (lo captura el `permit ip any any`). En V15 es aún más estricto: su lista termina en `deny any`, así que la DB no puede pinguear nada. `ping <gateway>` deja de servir como chequeo de salud en 6 de las 8 VLANs.
  >
  > **Y el reverso, que este documento no decía (5ª divergencia, Bloque 8): el gateway tampoco puede pinguear a sus propios hosts.** El echo-reply del host entra por la SVI, **no matchea el permit de echo-reply** —que apunta solo a Gestión— y cae en `deny ip any 10.0.0.0/8`. Aplica a los 4 switches L3 y a 6 de las 8 VLANs.
  >
  > **`ping <host>` desde el gateway es el primer comando de diagnóstico de cualquier NOC**, y esta lista blanca se lo lleva. El costo estaba documentado a medias: la mitad que se descubre sola —el usuario reclama— estaba escrita; **la que descubre el operador a las 3 AM, no**. Es un costo operativo real de la lista blanca y va declarado en los dos sentidos.
- **Permit de DHCP en las VLANs servidas** (V20/V40/V50/V55): el DISCOVER broadcast (`255.255.255.255`) pasa por el `permit ip any any` final, pero la **renovación es unicast a la IP del servidor** (la SVI del switch, dentro de `10.0.0.0/8`) y sería bloqueada por el deny de Invitados. Requiere permit explícito a las SVIs de ambos switches L3.
- **Retorno con `echo-reply`, no `established`.** Solo se implementa el retorno ICMP, que es lo único que el laboratorio puede validar (VPCS no hace TCP). Ver sección 11 — en producción cada camino de retorno necesita además `established`.

#### Limitaciones conocidas

- **El tráfico intra-VLAN es invisible para estas ACL.** Dos hosts en la misma VLAN se hablan por L2 puro sin pasar por la SVI. Por eso `austral-db` se separó a VLAN 15 (sección 3), y por eso los invitados necesitan aislamiento a nivel de puerto (sección 7.4).
- **Granularidad host, no puerto.** La política discrimina por IP de destino (app vs. db), no por puerto TCP. Es lo correcto para lo que el laboratorio puede validar con ping; ver sección 11.

### 7.3 Plano de gestión y AAA (Bloque 6)

Aplicado a los **8 equipos** de AustralPay.

**Acceso:**
- **Solo SSHv2** (`transport input ssh`, sin telnet), llave RSA 2048 — **y los servidores HTTP/HTTPS apagados** (`no ip http server` / `no ip http secure-server`) en los 8 `[B8]`. Ver abajo.
- **`access-class MGMT-VTY-ACCESS in`** en `line vty 0 15`; `deny any log` registra los intentos. **Asimétrica** `[B8]`:

| Sede | Quién alcanza sus VTY |
|---|---|
| **STGO** (4 equipos) | `10.1.30.0/24` — Gestión-STGO |
| **VALPO** (4 equipos) | `10.2.41.0/24` — Gestión-VALPO · **y `10.1.30.0/24`** `[B8]` |

- **`exec-timeout 10 0`** — **es el valor por defecto de IOS; no está configurado** `[B8]`. Ver abajo.
- **`no ip domain lookup`** en los 8 `[B8]`. Ver abajo.
- **Sincronización horaria (NTP):** sección 7.5.

> **`ip http server` estaba escuchando en los 8 (8ª divergencia, Bloque 8).** Esta sección declaraba *"Acceso: solo SSHv2"* y era **falso**: `ip http server` **y** `ip http secure-server` venían activos por defecto, con `ip http authentication local` —o sea **las mismas credenciales de admin**— y **sin `ip http access-class`**. `MGMT-VTY-ACCESS` no los cubría: **una ACL de VTY solo aplica a las VTY**. Resultado real: dos servicios de administración, con las credenciales buenas, alcanzables desde cualquier VLAN con ruta al equipo. Apagados en el B8 y verificado con `show ip sockets` / `show tcp brief all`: **ni 80 ni 443 escuchando**.
>
> **La promesa era de exclusividad —*"solo SSH"*— y se había verificado mirando lo que estaba configurado.** Una promesa de exclusividad no se verifica mirando lo que pusiste: se verifica mirando **qué está escuchando**.

> **`exec-timeout 10 0`: la promesa se cumplía por accidente (10ª divergencia, Bloque 8).** No está configurado en ningún equipo — **es el default de IOS**, y `show running-config` no muestra defaults. Verificado con `show running-config all | include exec-timeout` en los 8. Se declara así: **que el valor sea el correcto no lo convierte en una decisión**. Un control que se cumple por default no está protegido — nadie lo defiende el día que alguien lo cambia, porque nadie sabía que estaba ahí.

> **VTY asimétrica: por qué STGO alcanza VALPO y no al revés (B8-#3).** La sección 7.2 dice *"Gestión administra todo"*, y era verdad **solo dentro de su propia sede**: la ACL de VTY se escribió por sede y **nadie preguntó para qué existía el permit inter-sitio** del Bloque 5. Resultado: los 4 equipos de VALPO solo se administraban desde `10.2.41.0/24` — **y en VALPO no hay staff de TI** (el `00-CONTEXTO` pone operaciones y personal en la matriz). **Un equipo que solo se administra desde una sede sin administradores no es un control: es un equipo inadministrable.**
>
> **Se eligió asimetría sobre simetría:** la función de TI vive en STGO, y **la dirección que se abre es la que ya está más protegida** — Gestión-VALPO, la sede con invitados y menos control físico, **sigue sin llegar a las VTY de la matriz**. Precedente directo: el Bloque 6 ya rechazó forzar simetría en el rol operador.
>
> **Se descartó el permit por host** (`10.1.30.11` en vez del `/24`): la sección 7.2 dice que **el tráfico intra-VLAN es invisible para las ACL**. Un atacante ya dentro de V30 comparte dominio L2 con `S30A` — le toma la MAC o la IP y **el permit por host no lo ve**. La frontera de confianza real es *"estar en la VLAN de gestión"*, y eso es lo que el `/24` expresa. **El permit por host sería documentación disfrazada de control.**
>
> **El costo, dicho sin maquillaje:** comprometer `10.1.30.0/24` pasa de dar **4 equipos a dar 8**.

> **`no ip domain lookup`: el typo que sale a internet (B8-#9).** `ip domain-lookup` viene **activo por defecto** y no hay `ip name-server`. Cuando IOS no reconoce un comando en modo exec, **asume que es un hostname** al que quieres hacer telnet e intenta resolverlo **por todas las interfaces**: `BR-STGO` emitía consultas DNS a `255.255.255.255:53`, cada 4 s, **16 veces**. Salían por `Gi2` y `Gi3` — o sea **hacia los ISP** — y hacia la VPN. Es el *"se me colgó la consola 30 segundos"* que todo el mundo conoce y nadie sabe por qué pasa. En una red real es **un typo de la consola de tu router de borde viajando a internet en claro**, y a veces el typo es una contraseña mal pegada. Sin la línea 40 de la sección 7.1, era invisible.
>
> **No toca `ip domain name australpay.lab`** — son cosas distintas: uno **nombra al equipo** (y sostiene la llave RSA), el otro **busca nombres ajenos**.

**Identidades y roles** (niveles de privilegio, sin `aaa new-model`):

| Usuario | Nivel | Alcance |
|---|---|---|
| `admin` | 15 | Administración completa. |
| `operador` | 14 | Ver sección de perfiles abajo. |

Todas las credenciales con **hash tipo 8 (SHA256)** vía `algorithm-type sha256`.

**Perfiles del rol operador — deliberadamente distintos según el rol del equipo:**

| Equipo | Alcance del operador |
|---|---|
| **ALS (acceso)** | Asigna VLANs a puertos (`switchport access vlan`). **Sin** `shutdown`/`no shutdown`, `ip address`, routing, `username`, `crypto`. Cadena de privilegio: `exec → configure → interface → switchport access vlan`. |
| **DLS/MLS (distribución)** | Solo-lectura con cuenta nombrada (trazable). Ve HSRP (`show standby`, nivel 1 de fábrica). Sin acceso a modo config. Gestión de pools DHCP `[B7]`. |
| **BR (borde)** | Solo-lectura total. No hay VLAN/DHCP/HSRP que administrar; todo lo que tiene (BGP, NAT, crypto) es terreno del admin. |

> **Por qué no se le da `shutdown`/`no shutdown`:** en el modelo de niveles el privilegio es **por comando, no por interfaz** — darle recuperación de puertos err-disabled le daría también apagar el uplink que sostiene OSPF. Se optó por negárselo: una violación de port-security en un servidor es un **evento de seguridad**, y que la recuperación **escale al admin** es lo correcto. Ver sección 11 (TACACS+ resolvería esto por argumento o por grupo de equipos).
>
> **El rol no se fuerza simétrico:** en distribución no hay puertos de acceso de usuario, así que darle `switchport access vlan` sería privilegio decorativo.
>
> **Plus por defecto:** `show running-config` es nivel 15 de fábrica → el operador nunca lo ve, sin configurarlo. Ahí vive el hash del `enable secret`.

**Endurecimiento del acceso administrativo:**

| Control | Estado |
|---|---|
| `login block-for 120 attempts 3 within 60` | 8 equipos — bloqueo global tras 3 fallos en 60 s |
| `login delay 1` | 8 equipos — frena scripts de fuerza bruta |
| `login on-failure log` / `on-success log` | 8 equipos — rastro de auditoría local |
| `service password-encryption` | 8 equipos — ofuscación tipo 7 de cualquier `password` residual |
| `security passwords min-length 9` | **Solo bordes** — no soportado en vIOS-L2 |

> **Trade-off aceptado:** `login block-for` bloquea a **todos**, incluido el admin legítimo — es un DoS auto-infligido posible. Se puede exceptuar una IP de gestión con `login quiet-mode access-class`; a esta escala se aceptó sin excepción.
>
> **Límite de plataforma:** `security passwords min-length` no existe en la imagen vIOS-L2. La alternativa de Cisco (`aaa common-criteria policy`) exige `aaa new-model` **y** no evalúa usuarios creados con `secret 8`/`secret 9` — que es el hash usado aquí. Se aplica donde la plataforma lo soporta; en los switches queda como regla operativa. Ver changelog.
>
> **Diferencia de sintaxis:** el CSR1000v (IOS-XE 17.3) requiere `ip domain name` (con espacio); el vIOS-L2 acepta `ip domain-name` (con guión). Un error aquí impide generar la llave RSA y deja el SSH inoperativo en silencio. **IOS-XE cambió toda la familia `ip domain` de guión a espacio** `[B8]`: también `no ip domain lookup` en los bordes vs `no ip domain-lookup` en los switches.
>
> **Y el comando de verificación hereda el bug** `[B8]`: `show run | include ip domain-lookup` **no encuentra la forma con espacio** y reporta ausente algo que está. Se verifica con `include ip domain`. **Un comando de verificación que no puede encontrar lo que busca es peor que no verificar: te da un falso negativo con cara de dato.**

### 7.4 Endurecimiento de puertos de acceso

| Tipo de puerto | Port-security | Aislamiento |
|---|---|---|
| **Servidores (V10, V15)** `[B7]` | MAC sticky, `maximum 1`, violación **`shutdown`** | — |
| **Estaciones / Gestión (V20, V30, V40, V41)** | MAC sticky, `maximum 2`, violación `restrict` | — |
| **Invitados (V50, V55)** `[B7]` | `maximum 1`, violación `restrict`, **sin sticky** | **`switchport protected`** |
| **No usados** `[B8]` | — (el puerto está apagado) | `switchport mode access` + `nonegotiate` + `access vlan 888` + **`shutdown` de puerto y de VLAN** |

- `PortFast` + `BPDUGuard` en todos los puertos de acceso.
- **DHCP snooping** `[B7]` en ambos ALS, sobre las VLANs servidas por DHCP: uplinks (Port-channel) marcados `trust`, puertos de acceso *untrusted* por defecto. **Requiere `no ip dhcp snooping information option`** — sin eso, el switch inserta Option 82 con `giaddr=0` (bridgea, no relayea) y el servidor DHCP de IOS descarta los paquetes por defecto, **rompiendo el DHCP entero sin error visible**.

> **Violación `shutdown` en puertos de servidor (Decisión Bloque 6):** respuesta **diferenciada** — un servidor tiene MAC fija y es alto valor, así que un cambio de MAC ahí es un evento sospechoso que exige revisión humana. Los puertos de usuario quedan en `restrict` (dropea y loguea). La recuperación del err-disable es tarea del **admin**, no del operador (sección 7.3).
>
> **Sin sticky en invitados:** un puerto de invitados existe para que roten dispositivos distintos; MAC sticky sería contraproducente. `maximum 1` impide que enchufen un switch/hub y multipliquen invitados.
>
> **Puertos protegidos:** una ACL de SVI **no ve el tráfico intra-VLAN**, así que sin esto un invitado comprometido atacaría libremente a los demás invitados. `switchport protected` bloquea unicast, multicast y broadcast entre puertos protegidos del mismo switch, dejando pasar el tráfico hacia el uplink (no-protegido). Es el equivalente switch-side del *client isolation* de un AP. **Limitación:** solo aísla dentro del mismo switch — ver sección 11 (Private VLAN).

> **Política de puerto no usado (B8-#1).** Aplicada a los **12 puertos libres** de `DLS1`/`DLS2`/`MLS1`/`MLS2` (`Gi1/1`–`Gi1/3` en cada uno). Los dos ALS están al **100% de ocupación** —`ALS1-STGO` tiene 10 puertos y usa 10— así que **cumplen la política por ausencia de superficie**. Se declara en vez de fabricar puertos para demostrarla: *"los agregué para mostrar que sé apagarlos"* no es una respuesta de entrevista.
>
> **La cadena que lo hace grave sale del propio diseño:** puerto en `dynamic auto` → trunk negociado (*switch spoofing*) → `Trunking VLANs Enabled: ALL` significa que no aterriza en "una VLAN": aterriza en **todas** → el atacante etiqueta como **VLAN 30** y cae en la única VLAN **sin ACL de política** (7.2) y la única **autorizada a las VTY** de los 8 equipos (7.3). No entra —le faltan credenciales—, pero **el control de 7.3 deja de existir**: esa ACL asume que estar en `10.1.30.0/24` es difícil. **La nativa 999 no cubre este ataque:** mitiga *double tagging*, que es el otro.
>
> **Cuatro candados, no uno:** `switchport mode access` (mata la negociación — **este es el control**) · `nonegotiate` (deja de **emitir** DTP; es lo que hace verdadera la frase de 7.0) · `access vlan 888` (si se levanta, aterriza aislado) · `shutdown` de puerto **y de la VLAN** (el segundo mata el riesgo **el día que alguien haga `no shut` sin pensar**, que es el día en que esto pasa).
>
> **Se cerró la clase, no la instancia:** un bloque de validación es donde se cierran clases. Arreglar solo los puertos que miraste **no es validar, es parchar**.
>
> **Los 12 puertos parqueados son el inventario disponible**, no puertos quemados: `no shut` + `switchport access vlan X` y quedan operativos. La política no los pierde — **vuelve deliberado el acto de habilitar uno**.

> **Nota de lectura de las configs — `maximum 1` y `violation shutdown` no aparecen en el archivo.** Son el **valor por defecto** de IOS, y `show running-config` no muestra defaults: en los puertos de servidor solo se ve la línea de `sticky`. **La decisión existe y está argumentada** (changelog B6: respuesta diferenciada porque un servidor tiene MAC fija y es de alto valor) y su comportamiento **se verificó** en el B7 con un err-disable real. Es lo **contrario** del caso `exec-timeout` de la sección 7.3: allá el default cumplió por casualidad una promesa que nadie tomó; **acá el default coincide con una decisión tomada**. Teclearlos para que se vean sería la misma ceremonia que esta arquitectura ya rechaza en 7.2 al no escribir `permit ip any any` en V30. **Una decisión que coincide con el default sigue siendo una decisión — pero hay que decir dónde vive, porque en el archivo no se ve.**

### 7.5 Sincronización horaria (NTP) `[B8]`

`BR-STGO` es el reloj de la red (`ntp master 3`, estrato 3) y los otros **7 equipos** son sus clientes, apuntando a **`10.1.30.254`** — su loopback de gestión (sección 4.7).

| Parámetro | Valor | Por qué |
|---|---|---|
| Servidor | `BR-STGO`, `ntp master 3` | Se eligió sobre `ISP-1` — ver abajo |
| Dirección del servidor | **`Loopback30` `10.1.30.254`**, no una interfaz física | Un reloj que vive en una interfaz **desaparece con esa interfaz** |
| Autenticación | **MD5** — `ntp authentication-key 1` + `ntp trusted-key 1`, la misma en los 8 | No es ceremonia — ver abajo |
| **`ntp source`** | **Obligatorio** en los 7 clientes: `Vlan30` (STGO) · `Vlan41` (VALPO) · **`Loopback41`** en `BR-VALPO` | Es el punto entero — ver abajo |
| Zona horaria | **UTC**, sin `clock timezone` | Chile cambia a horario de verano: la hora entre 23:00 y 00:00 **ocurre dos veces** en otoño. UTC en la infraestructura; hora local solo en la pantalla del analista |

**Por qué NTP está en el diseño de una red y no en el del SOC:** un SIEM que recibe logs de 10 equipos **sin reloj común no puede ordenar dos eventos de la misma cadena de ataque**. Un timeline forense con relojes a la deriva **no es evidencia**. Es prerequisito, no infraestructura de fondo — y es lo que los proyectos de NOC (Zabbix) y SOC (Wazuh) consumen directamente de esta red.

> **El hallazgo que lo motivó (B8-#7).** Durante el failover de ISP **no se pudo correlacionar** el syslog de `ISP-1` (`19:19:54`) con el de `BR-STGO` (`hace 00:16:56`): cada nodo corría su propio reloj. **El timeline se reconstruyó con contadores `MsgRcvd`, no con horas.** Y una de las preguntas del bloque —cuánto tardó el túnel en re-formar su adyacencia OSPF— **quedó sin respuesta por eso mismo** (sección 11.2).

> **Se eligió `BR-STGO` sobre `ISP-1` como fuente.** El NTP existe para correlacionar logs, y **el momento en que más necesitas correlacionar es durante una falla**. Un reloj que depende de un ISP **se pierde justo cuando lo necesitas** — este mismo bloque midió **129 s** en que `BR-STGO` no tenía a `ISP-1` (sección 5.3). Además, el colector de Zabbix y el manager de Wazuh viven en STGO: la fuente de tiempo queda **en la misma sede que el SIEM**.
>
> **Estratos:** 0 = reloj físico (GPS/cesio) · 1 = servidor pegado a él · +1 por salto. `ntp master 3` = *"declárate estrato 3 usando tu propio reloj"*. Es **lo mismo que hace un servidor NTP de producción**, solo que él saca la hora de un pool público o un GPS: **la jerarquía es idéntica, cambia el origen**.

> **`ntp source` no es un detalle de sintaxis: es el punto entero.** Sin él, cada cliente sourcea de la interfaz por la que sale, y **el propio diseño lo mata**:
> - `MLS1-VALPO` sourcearía de `10.255.2.1` (su uplink), que **no matchea `10.2.41.0/24`** → lo mata el filtro inter-sitio (7.1).
> - `BR-VALPO` sourcearía de `10.255.0.2` (el túnel) → cae en la **línea 40**.
>
> Es la sección 7.3 en versión chica: *"el control no mira quién eres, mira desde dónde vienes"* — **y un router que no elige su origen, elige mal**.

> **La autenticación no es ceremonia.** Sin ella, cualquiera en la VLAN de gestión **se hace pasar por el servidor y corre el reloj de los 8 equipos**. Es un ataque **anti-forense**: mueves el tiempo y los logs dejan de correlacionar, **sin borrar una sola línea**. La clave va con hash **tipo 7 (ofuscación reversible)** porque **IOS no ofrece nada mejor para NTP** — se declara la limitación en vez de presentarla como cifrado, y por eso está redactada en las configs publicadas.
>
> **`BR-STGO` no lleva `ntp authenticate` y los otros 7 sí — y no es un olvido.** `ntp authenticate` gobierna las asociaciones que el equipo forma **como cliente**, y el master no forma ninguna. Lleva `ntp authentication-key` y `ntp trusted-key`, que es lo que necesita para **firmar lo que responde**. Se declara para que la asimetría entre las 8 configs no se lea como descuido.

> **Lo que este punto validó de gracia:** `BR-VALPO` sincroniza a **2,5 ms a través del túnel GRE-over-IPSec con MD5**, `poll 128` (NTP solo sube el poll cuando el peer es estable). **El permit `10.2.41.0 → 10.1.30.0` del Bloque 5 pasó de transportar pings de verificación a transportar el reloj de la sede**: un contador que crece solo cada 64 s, para siempre. Las secciones 4.7 y 7.5 validándose mutuamente con tráfico real y permanente.
>
> **`reach` es un registro de corrimiento en octal** — los últimos 8 polls: `377` = `11111111` = 8 de 8; `177` = perdió el más viejo; `1` = perdió 7 seguidos. Es el health check de NTP y casi nadie lo sabe leer. Acá delató **el entorno, no la red**: ver 11.1 y 11.2.
>
> **El asterisco, un par antes/después gratis:** `*Jul 15 20:26:00` — ese `*` está en **cada log de este proyecto hasta el B8** y significa *"mi reloj no está sincronizado, no confíes en esta hora"*. Al sincronizar, **desaparece**.

> **Limitación declarada:** `ntp master` es un reloj **inventado** — hora **relativa** correcta, hora **absoluta** falsa. La referencia `127.127.1.1` (`.LOCL.`) **es** la prueba. Para correlación forense relativa basta; para forense real, no: ver sección 11 (fuente externa trazable). Y los 6 vIOS-L2 **sincronizan pero no con la precisión que un SIEM necesita**: ver 11.1.

---

## 8. Decisiones de diseño defendibles (resumen para entrevista)

- **Un solo IGP (OSPF)** en vez de apilar EIGRP + OSPF + redistribución → mantenibilidad; lo múltiple solo existe en migraciones.
- **GRE-over-IPSec** en vez de IPSec puro → el crypto map no transporta el multicast de OSPF; GRE da la interfaz ruteable, IPSec el cifrado.
- **`default-information originate`** en vez de redistribuir BGP a OSPF → nunca se mete la tabla de internet en el IGP.
- **Dos sedes de producción independientes y resilientes** (ambas multihomed + HSRP) → la expansión regional lo justifica; el DR es subproducto de la separación geográfica.
- **Un solo router de borde por sede** (con doble ISP) → el borde es un SPOF aceptado; la resiliencia está en los ISP y la VPN.
- **RFC 5737** para las direcciones "públicas" → práctica correcta de laboratorio.
- **STP root alineado con HSRP activo** por VLAN → evita hairpinning **en estado normal** (medido en el B8); y **V10/V15 en el mismo switch** porque son la conversación app→db. **En estado degradado la alineación se rompe sola** —HSRP conmuta, STP no— y el hairpin ocurre igual; arreglarlo cuesta ~30 s de reconvergencia STP para ahorrar un salto de peer-link, y no vale la pena. **Conocer el límite del propio diseño se defiende mejor que la decisión** (sección 6.1).
- **VLAN nativa dedicada + DTP off + poda** → endurecimiento de capa 2.
- **ACL de NAT que excluye el tráfico de la VPN** → el tráfico inter-sitio viaja cifrado, no natteado.
- **VLAN de tránsito dedicada para OSPF entre switches de distribución** → mantiene `passive-interface default` sin excepciones en VLANs de datos.
- **Borde dual-homed a ambos switches de distribución** → el tracking de HSRP en cada switch vigila su propio uplink físico.
- **Dos ISP compartidos por ambas sedes** → multihoming genuino sin operar una mini-internet; refleja la concentración real de oferta de ISP en Chile.
- **VPN anclada a loopback público anunciado por BGP** → la resiliencia del multihoming protege también al túnel.
- **`allowas-in` en vez de fragmentar AustralPay en dos AS** → resuelve el loop de BGP sin reabrir la identidad de AS único del Bloque 1.
- **Filtro inter-sitio acotado a Gestión↔Gestión, en el único punto de tránsito (Tunnel0)** → autorización granular sin duplicar controles.
- **iBGP omitido y documentado** → con solo 2 prefijos anunciados, ambos ya aprendidos por eBGP, y transporte interno por OSPF-over-GRE, una sesión iBGP no cargaría nada nuevo. Reconocer que un protocolo planificado no aporta es criterio, no desconocimiento.
- **AAA local con roles, en vez de TACACS+** → autenticación fuerte, SSHv2, ACL de VTY y separación de roles sin agregar un servidor que se vuelve SPOF. Saber *cuándo* centralizar vale más que tenerlo montado.
- **Rol operador definido por lo que se le niega** → un operador que no puede crear usuarios, tocar AAA/crypto ni apagar interfaces no puede auto-escalar ni romper el plano de routing.
- **Lista blanca (default-deny) intra-sitio** → lo que se olvidó queda bloqueado; falla cerrado.
- **Separación app/db en VLANs distintas** → sin ella, el salto app→db es L2 puro y ninguna ACL puede verlo: quedaba permitido por física, no por política.
- **La DB no inicia conexiones salientes** → una base de datos abriendo conexiones hacia afuera es un indicador clásico de exfiltración; prohibirlo por política es también una regla de detección.
- **ACL `out` adicional en la SVI de la DB** → el esquema basado en origen falla abierto ante una VLAN nueva sin ACL; el activo crítico lleva doble candado.
- **ICMP NO se bloquea internamente** → romper PMTUD en una red con GRE-over-IPSec (MTU ~1400) provoca black holes de TCP a cambio de un valor de seguridad casi nulo. Se bloquea **por política de origen, no por protocolo**.
- **VLAN de Invitados aislada, no solo creada** → el entregable es la política; una VLAN de invitados sin ACL que llega a la base de datos es peor que no tenerla.
- **DHCP split-scope, no servidor único** → IOS no tiene DHCP failover; split-scope es la única redundancia posible en el switch. Y una caída de DHCP no es una caída de red: su impacto es diferido.
- **Default aprendida por BGP, no estática al primario** `[B8]` → clavar una estática al ISP-1 es **exactamente lo que el multihoming venía a evitar**; y convierte la LocalPref del Bloque 5 de anécdota en política de negocio: pasa de gobernar 2 prefijos /32 a decidir por dónde sale **todo el internet de la sede**.
- **Filtro de salida BGP (`ANUNCIO-PROPIO`)** `[B8]` → un cliente multihomed que no filtra su salida **se convierte en tránsito accidental entre sus upstreams**, y le ofrece una ruta por defecto a un AS de tránsito. Detectado con `show ip bgp neighbors advertised-routes` — **el comando que nadie corre**.
- **NAT con route-map: la interfaz como condición, no como resultado** `[B8]` → si el NAT no sigue al ruteo, traduce a un prefijo muerto y el retorno nunca llega. **Falla peor que no natear:** sale, nadie vuelve, y todos los `show` se ven bien.
- **NTP autenticado como prerequisito del SOC, no como infraestructura de fondo** `[B8]` → sin reloj común, dos eventos de la misma cadena en equipos distintos **no se pueden ordenar**. Y se autentica porque **mover el reloj borra evidencia sin tocar un log**.
- **Política de puerto no usado (VLAN 888)** `[B8]` → el endurecimiento cubría los puertos **usados**; **el silencio era el hueco**. Un puerto en `dynamic auto` con `Trunking VLANs: ALL` aterriza en la única VLAN sin ACL y con acceso a las VTY.
- **`deny any any log` explícito en el filtro inter-sitio** `[B8]` → el deny implícito **no cuenta y no loguea**. En un canal autenticado por IPSec el volumen esperado es cero, así que **cada evento es señal**: atrapó tres flujos legítimos el primer día.
- **Acceso de gestión asimétrico** `[B8]` → un equipo que solo se administra desde una sede **sin administradores** no es un control: es un equipo inadministrable. Y la dirección que se abre es **la que ya está más protegida**.
- **Loopback de gestión en los bordes** `[B8]` → el argumento del Bloque 5 (*"el túnel se ancla a un loopback, no a la interfaz física de un ISP"*) aplicado al plano de gestión: **una dirección que vive en un uplink desaparece justo cuando la necesitas**.

---

## 9. Qué sigue (siguientes bloques)

- **Bloque 2 — Capa 2 STGO:** VLANs, troncales, RPVST+, EtherChannel, port-security. ✅
- **Bloque 3 — Capa 3 STGO:** SVIs, InterVLAN, HSRP, OSPF área 1. ✅
- **Bloque 4 — Sitio VALPO:** capa 2 y 3 + OSPF área 2 + HSRP (VLANs 40/41). ✅
- **Bloque 5 — Borde y WAN:** eBGP multihoming (ambas sedes), NAT, VPN GRE-over-IPSec, filtro inter-sitio, manipulación de atributos BGP. ✅
- **Bloque 6 — Plano de gestión y AAA local:** SSHv2 + usuarios + roles + endurecimiento en los 8 equipos; decisión de iBGP; reconstrucción de bordes; diseño de segmentación y servicios del B7. ✅
- **Bloque 7 — Segmentación y servicios:** VLANs 15/50/55 (SVI + HSRP + EOT + RPVST+ + OSPF), política intra-sitio (sección 7.2), DHCP split-scope (sección 4.6), protected ports + DHCP snooping + port-security de servidor (sección 7.4), endurecimiento de consola, y los pendientes del checklist del Bloque 6. ✅
- **Bloque 8 — Verificación y entrega:** matriz de conectividad end-to-end **59/59** (permit *y* deny, con la predicción escrita **antes** de cada prueba), failover real medido con ping continuo (**ISP: 129 s** · **HSRP: 5 s**), **11 divergencias** corregidas entre lo declarado y lo real, 8 configs congeladas y sanitizadas, diagrama final, documento de entrega. ✅

> **Cierre.** El proyecto de red termina en el Bloque 8. Lo que sigue son los proyectos que corren **sobre** esta red: **NOC (Zabbix)** y **SOC (Wazuh)**, que consumen su plano de gestión (7.3), su reloj común (7.5) y sus 12 puertos parqueados como inventario disponible (7.4). Ver `00-CONTEXTO`, sección 6.
>
> **Lo que este bloque dejó dicho sobre el resto:** un `show` prueba que un componente hace lo que dice; **una matriz de conectividad prueba que el diseño hace lo que prometió**. Las 11 divergencias no aparecieron porque el trabajo anterior estuviera mal hecho: aparecieron porque **nadie las había buscado**. Es la diferencia entre configurar y entregar.

---

## 10. Historial de decisiones (changelog)

Registro de cambios respecto a la idea inicial del diseño. Cada fila = una decisión tomada durante la ejecución, con su bloque y su porqué.

| Fecha | Bloque | Cambio | Por qué |
|---|---|---|---|
| 2026-07-11 | Pre-B3 | VPN pasa de **IPSec puro (crypto map)** a **GRE-over-IPSec** | El crypto map solo cifra unicast que matchea la ACL; no levanta interfaz ruteable ni transporta el multicast de OSPF (224.0.0.5). GRE da la interfaz `Tunnel0` con IP del /30 en área 0; IPSec la protege. |
| 2026-07-11 | Pre-B3 | VALPO redefinido de **DR frío** a **sede regional de producción** | La narrativa real es expansión: empresa que crece y abre sede regional, no un sitio de respaldo pasivo. |
| 2026-07-11 | Pre-B3 | VALPO pasa a **multihoming**; `BR-VALPO` queda simétrico a `BR-STGO` | Una sede de producción que atiende tráfico propio necesita salida a internet resiliente. |
| 2026-07-11 | Pre-B3 | Se agrega `MLS2-VALPO` + **HSRP en VALPO** (VLANs 40/41), RPVST+ alineado | HSRP requiere dos dispositivos L3; VALPO tenía uno solo. |
| 2026-07-11 | B2 | Nota de ejecución: el vIOS-L2 tiene **4 puertos por módulo (Gi0/0–Gi0/3)** | Restricción de la imagen; obliga a planificar puertos. |
| 2026-07-12 | B3 | Se agrega **VLAN 99 de tránsito** (`10.255.99.0/30`) sobre el peer-link Po1, dedicada a la adyacencia OSPF `DLS1`↔`DLS2` | Vecinar sobre las SVIs de usuario habría expuesto VLANs de datos a `passive-interface` activo y generado adyacencias redundantes. |
| 2026-07-12 | B3 | Enlaces `BR-STGO`↔distribución documentados como **dos /30 separados** (dual-homing) | Cada DLS necesita su propio uplink físico para que el tracking de HSRP (EOT) vigile un enlace propio. |
| 2026-07-12 | B3 | Nota de ejecución: HSRP requiere **Enhanced Object Tracking**, no `standby track <interfaz>` directo | Restricción de la plataforma; no cambia el diseño, solo la sintaxis. |
| 2026-07-12 | B4 | Se agrega **VLAN 99 de tránsito en VALPO** (`10.255.99.4/30`), réplica del patrón de STGO | Mismo argumento que en STGO: aísla el control-plane OSPF de las SVIs de usuario. |
| 2026-07-12 | B4 | `BR-VALPO` pasa a **dual-homing** (dos /30 separados), simétrico a `BR-STGO` | El enlace único era una asimetría heredada de cuando VALPO era un switch colapsado. |
| 2026-07-13 | B5 | **Colapso de 4 AS de ISP (65001-65004) a 2 AS compartidos**, peereados entre sí | Bajo función > seguridad > simplicidad: 2 AS mantienen diversidad de camino real y resiliencia per-sitio; 4 AS solo agregaban aislamiento de fallos *entre* sitios sin capacidad nueva. Refleja la concentración real de oferta de ISP en Chile. |
| 2026-07-13 | B5 | Túnel anclado a **loopback público anunciado por BGP** (`192.0.2.100/32`, `192.0.2.101/32`) en vez de a la IP física de un ISP | Anclar a una IP física habría dejado la VPN dependiente de un solo ISP, contradiciendo el multihoming recién construido. |
| 2026-07-13 | B5 | Se agrega `allowas-in 1` en los 4 vecinos eBGP | Al compartir AS 65010 y los mismos ISP entre sitios, la prevención de loop de BGP descartaba el anuncio de un sitio al volver con el propio AS en el path. Preferido sobre fragmentar AustralPay en dos AS. |
| 2026-07-13 | B5 | Se agrega **filtro inter-sitio** (`INTERSITE-FILTER-IN`) en `Tunnel0`: solo Gestión↔Gestión cruza | Requisito de segmentación no contemplado en el diseño original. Aplicado en el único punto de tránsito posible. |
| 2026-07-13 | B5 | `ip ospf network point-to-point` en `Gi1/0` de `DLS1`/`DLS2` (pendiente desde B3) | Sin esto la adyacencia con `BR-STGO` subía como `FULL/BDR` en vez de `FULL/-` — funcional pero asimétrico respecto a VALPO. |
| 2026-07-13 | B5 | `default-information originate` + ruta estática con **next-hop IP** hacia el ISP primario | Sin ruta por defecto el interior no podía rutear hacia "internet" y el NAT nunca actuaba. Next-hop IP evita el lookup recursivo de IOS en enlaces Ethernet multiacceso. |
| 2026-07-13 | B5 | Manipulación de atributos BGP: `ISP-1` primario (LocalPref 200 vs 100, AS-path prepend ×2 hacia ISP-2) | BGP elegía el mejor camino con el criterio default, sin política explícita del negocio. |
| **2026-07-14** | **B6** | **iBGP entre bordes ELIMINADO del diseño** (estaba planificado desde el B1, pendiente de activación al cierre del B5). Los loopbacks `10.255.255.1/.2` quedan solo como router-id de OSPF | BGP anuncia **solo 2 prefijos** (los loopbacks públicos que anclan la VPN) y **cada borde ya aprende el del otro por eBGP** vía los ISP con `allowas-in`; el transporte interno va por OSPF-over-GRE y el egress por `default-information originate`. Una sesión iBGP cargaría exactamente los mismos 2 prefijos: **redundante**. Tampoco podría alcanzar el destino del túnel (el túnel se ancla en esos loopbacks: no puede aprenderlos por encima de sí mismo). Se documenta la omisión en vez de configurar protocolo muerto. |
| **2026-07-14** | **B6** | **Plano de gestión y AAA local** en los 8 equipos: SSHv2 + llave RSA 2048, `admin` (priv 15) / `operador` (priv 14) con hash tipo 8, `access-class MGMT-VTY-ACCESS` limitando VTY a la VLAN de gestión, y endurecimiento anti-fuerza bruta | El diseño (sección 7) prometía "SSH restringido a la VLAN de gestión con ACL en las VTY" pero nunca se había implementado. AAA local es defendible a esta escala: da autenticación fuerte y separación de roles sin agregar un servidor TACACS+ que se vuelve SPOF. Ver sección 11 para los disparadores de centralización. |
| **2026-07-14** | **B6** | **Rol operador SIN `shutdown`/`no shutdown`**; la recuperación de puertos err-disabled escala al admin | En el modelo de niveles de privilegio el permiso es **por comando, no por interfaz** — no existe "puede hacer `no shut` en el puerto del servidor pero no en el uplink". Darle recuperación le daría también apagar el uplink que sostiene OSPF. Además, una violación de port-security en un servidor es un **evento de seguridad**: que escale es lo correcto. TACACS+ (por argumento o grupo de equipos) lo resolvería — es un disparador concreto de centralización. |
| **2026-07-14** | **B6** | **Rol operador con alcance distinto por tipo de equipo** (ALS: VLANs · DLS/MLS: solo-lectura + HSRP · BR: solo-lectura total) | Se rechazó forzar simetría: en distribución y borde no hay puertos de acceso de usuario, así que `switchport access vlan` sería privilegio decorativo. El rol se define por lo que el equipo realmente tiene para operar. |
| **2026-07-14** | **B6** | `security passwords min-length 9` **solo en los bordes** (CSR1000v); en los switches queda como regla operativa documentada | No existe en la imagen vIOS-L2. La alternativa de Cisco (`aaa common-criteria policy`) exige `aaa new-model` **y** no evalúa usuarios creados con `secret 8`/`secret 9` — el hash usado aquí. Se prefiere un control que funciona a uno que aparenta funcionar. |
| **2026-07-14** | **B6** | Nota de ejecución: **`ip domain name` (con espacio) en CSR1000v**; el vIOS-L2 acepta la forma con guión | IOS-XE 17.3 rechaza `ip domain-name`. El fallo encadenó: sin dominio, `crypto key generate rsa` abortó y el SSH quedó configurado pero **inoperativo, sin error visible**. |
| **2026-07-14** | **B6** | **`no ip routing` en ambos switches de acceso** (`ALS1-STGO`, `ALS1-VALPO`) | La imagen **vIOS-L2 trae `ip routing` activo por defecto** (verificado: los informes solo lo configuran en DLS/MLS; el B2 lo lista como pendiente para el B3). Con `ip routing` activo, IOS **ignora el `ip default-gateway`** y el switch queda sin ruta por defecto — se destapó al primer tráfico fuera de subred de todo el proyecto (SSH del operador hacia el borde). Un switch de acceso ruteando es además una desviación de su rol. |
| **2026-07-14** | **B6** | **Retirada la ruta por defecto duplicada** (`ip route 0.0.0.0 0.0.0.0 GigabitEthernetX`) de ambos bordes | Quedó de la Prueba 3 fallida del B5: se agregó la versión correcta con next-hop IP pero **nunca se retiró la vieja** basada en interfaz. Misma distancia administrativa → IOS las trataba como equivalentes. Corregir no es solo agregar lo correcto, es retirar lo incorrecto. |
| **2026-07-14** | **B6** | **El ping directo entre IPs internas del túnel queda bloqueado — aceptado sin excepción** | No matchea el `permit ospf` ni el de Gestión de `INTERSITE-FILTER-IN` → cae en el deny implícito. No es una regresión: el ping exitoso del B5 fue en el Punto 3, **antes** de aplicar la ACL en el Punto 5, y no se volvió a correr. El diagnóstico de túnel pasa a ser la adyacencia OSPF en FULL (que no sube sin tráfico real), preservando el filtro tal como fue diseñado. |
| **2026-07-14** | **B6** | **Incidente y reconstrucción:** `BR-STGO` y `BR-VALPO` cayeron a prompt `grub>` por corrupción de disco; reconstruidos desde los `show running-config` capturados en los session logs | Un CSR1000v nuevo de prueba arrancó normal → corrupción de esas dos instancias, no de la imagen maestra. **Hipótesis:** presión de memoria del host (27/32 GB) causando apagados no-graceful. Una primera reconstrucción hecha desde el diseño + memoria difería del estado real en 3 puntos (4 route-maps y no 2; deny específico y no genérico en el filtro inter-sitio; ruta por defecto duplicada) → **el log ganó sobre la inferencia**. |
| **2026-07-14** | **B6** | **Se agrega VLAN 15 (Servidores-DB, `10.1.15.0/24`)**, separando `austral-db` de `austral-app` `[B7]` | Compartiendo VLAN 10, el tráfico app→db era **L2 puro y ninguna ACL podía filtrarlo**: quedaba permitido por física, no por política. Con VLAN propia, ese salto cruza una SVI y la política de la sección 7.2 puede aplicarse. HSRP/root en DLS1 junto a V10 para evitar hairpinning de esa conversación. |
| **2026-07-14** | **B6** | **Se agregan VLANs de Invitados: 50 en STGO (`10.1.50.0/24`) y 55 en VALPO (`10.2.55.0/24`)** `[B7]` | El DHCP necesitaba su caso de uso real (dispositivos transitorios y no confiables) y una fintech no puede tener dispositivos no confiables ruteando libre hacia servidores. Se descartó el nombre "Wifi": no hay AP en el lab y la config switch-side es idéntica con o sin él — lo artificial sería afirmar que se probó wireless. |
| **2026-07-14** | **B6** | **Se agrega política de segmentación intra-sitio** (lista blanca, sección 7.2) `[B7]` | **Hallazgo:** el proyecto filtraba el tráfico inter-sitio (el más protegido, ya cifrado) y dejaba el intra-sitio completamente abierto — cualquier estación alcanzaba la base de datos. Patrón clásico de perímetro protegido / interior plano. Alcance: Invitados aislados + Servidores protegidos, con la matriz completa documentada aunque solo se implementen las filas que importan. |
| **2026-07-14** | **B6** | Política de retorno con **`echo-reply` solamente** (sin `established`) | Los ACL son stateless: una política unidireccional exige permitir el retorno explícitamente. Solo se implementa el retorno ICMP, que es lo único validable con VPCS. `established` para TCP pasa a la lista de producción (sección 11). |
| **2026-07-14** | **B6** | **ACL `out` adicional en la SVI de V15 (DB)**, sobre el esquema general basado en origen | El esquema `in` en la SVI de origen **falla abierto**: una VLAN nueva sin ACL queda permitida hacia todo. El activo de mayor valor lleva una lista blanca explícita de quién puede alcanzarlo, que sobrevive al olvido. |
| **2026-07-14** | **B6** | **Regla obligatoria: permit de HSRP en toda ACL de SVI que termine en `deny any`** | Detectado **en diseño, antes de configurar**: la ACL de V15 habría descartado los hellos de HSRP del switch vecino (multicast **`224.0.0.102`**, UDP 1985) que entran por esa misma SVI → **split-brain** en la VLAN de la base de datos, con síntoma intermitente. Es el mismo bug que el `permit ospf any any` del túnel ya evita. |
| **2026-07-14** | **B6** | **ICMP NO se bloquea en la infraestructura interna** | Se evaluó y descartó: el modelo de amenaza (Ping of Death, Smurf) está obsoleto o ya mitigado por defecto (`no ip directed-broadcast`), y un flood volumétrico consume el ancho de banda antes de llegar a la ACL. En cambio **rompe PMTUD**, crítico en esta red por el MTU reducido del GRE-over-IPSec (~1400 vs 1500) → black holes de TCP. Se bloquea por **política de origen**, no por protocolo. CoPP (limitar en tasa hacia la CPU) es la respuesta correcta al plano de control → sección 11. |
| **2026-07-14** | **B6** | **DHCP: split-scope en ambos switches L3, no servidor único** `[B7]`; leases diferenciados (invitados 2 h, estaciones 7 días) | **IOS no tiene DHCP failover** — split-scope es la única redundancia posible en el switch. Se aceptan sus costos (sin estado compartido, capacidad a la mitad durante una falla, asignación no determinista). Lease largo en estaciones para que una caída del servidor tenga impacto diferido; corto en invitados para devolver IPs al pool. |
| **2026-07-14** | **B6** | **Se agregan `switchport protected` (invitados) y DHCP snooping** `[B7]` | Una ACL de SVI **no ve el tráfico intra-VLAN**: sin puertos protegidos, un invitado comprometido ataca libremente a los demás invitados. Y desplegar DHCP sin protegerlo del servidor pirata es dejar sin cerradura la puerta recién instalada. Requiere `no ip dhcp snooping information option` (Option 82 con `giaddr=0` rompe el DHCP entero). |
| **2026-07-15** | **B7** | **§7.2 — celda `V20 → V30` corregida de ❌ a ↩️** | Era la **única celda sin su ↩️ en toda la red** y contradecía el texto de su propia sección (*"Gestión administra todo"*): con ❌, gestión no podía administrar las estaciones, que es la VLAN que más se administra en una red real. Detectado ejecutando la matriz: `S30A → S20A` daba **timeout** en vez de `code 13` — el paquete salía y llegaba, pero la **respuesta** moría en `POL-V20-OPER`. La matriz se había transcrito fielmente a las ACL, incluido su error. |
| **2026-07-15** | **B7** | **§6.1 — HSRP normalizado en las 8 VLANs: v2 · grupo = VLAN · 105/100 · `preempt delay minimum 30`** | El diseño no fijaba versión ni convención de grupo, y la ejecución acumuló drift: V10/V20/V30/V40/V41 en **v1** y las nuevas en v2; VALPO con grupos `1`/`2` mientras STGO usaba grupo = VLAN. Se normalizó a v2 (4096 grupos, timers en ms, MD5, IPv6 — v1 no tiene ninguna ventaja) y grupo = VLAN (la MAC virtual codifica la VLAN: `f028` = 40). **La prioridad 150 de las VLANs nuevas se bajó a 105**: con `decrement 10` quedaba en 140 > 100 y **el track no podía conmutar** — redundancia decorativa. |
| **2026-07-15** | **B7** | **§4.6 — `dns-server 9.9.9.9` (Quad9) en los 4 pools; `DLS1`/`MLS1` sirven la mitad baja** | El diseño no fijaba DNS ni cuál switch sirve qué mitad. Quad9 sobre `1.1.1.1`/`8.8.8.8` porque **rechaza dominios maliciosos conocidos** (corta C2/phishing antes del primer paquete) al mismo costo: un campo. Se descartó el DNS diferenciado — le pondría el mejor control a la red menos crítica. Mitad baja uniforme en `DLS1`/`MLS1`: alinearla con el activo de HSRP no compra nada (el cliente elige por primer OFFER, no por gateway). |
| **2026-07-15** | **B7** | **§7.2 — V30 y V41 (Gestión) sin ACL** | Su fila de la matriz es todo-permitido: la lista sería `permit ip any any`. La **ausencia de ACL *es* esa política**; aplicarla sería ceremonia que consume CPU por cero enforcement. Se documenta para que la ausencia no se lea como omisión. |
| **2026-07-15** | **B7** | **§7.2 — `log` en tres líneas de deny (acceso a la DB, en ambas direcciones)** | `deny ip any 10.1.15.0/24 log` en `POL-V20-OPER`/`POL-V50-INV` captura **quién intentó llegar a la DB**; `deny ip any any log` en `POL-V15-DB` (in) captura **la DB iniciando algo** — indicador clásico de exfiltración o C2. Las dos mitades de la misma pregunta, ambas con volumen esperado **cero**, que es el criterio que hace un log señal en vez de ruido. |
| **2026-07-15** | **B7** | Nota de ejecución: **RPVST+ solo estaba en 3 de los 6 switches** | El informe del B2 declaraba *"RPVST+ en los tres switches"*; el **root sí se había planificado**, pero el **modo** solo llegó a `ALS1-STGO`. `DLS1-STGO`, `DLS2-STGO` y `ALS1-VALPO` corrían PVST clásico. Nunca se notó porque PVST+ y RPVST+ interoperan (fallback a 802.1D) — se perdía la convergencia sub-segundo, justo el argumento con que el B2 descartó 802.1D. Un informe declara la intención; solo un `show` declara el estado. |
| **2026-07-15** | **B7** | Nota de ejecución: **una SVI recién creada nace en `shutdown`** en esta plataforma | Las tres SVI nuevas quedaron en HSRP `Init/unknown/unknown`. `Init` no significa *"no encuentro al vecino"* sino *"la interfaz no está arriba"*. Sexta sorpresa de plataforma del proyecto. |
| **2026-07-15** | **B7** | Nota de ejecución: **hostnames `MLS1`/`MLS2` corregidos a `MLS1-VALPO`/`MLS2-VALPO`** | Eran los únicos 2 de 8 equipos sin sufijo de sede; el inventario (sección 2) ya decía el nombre correcto. Se corrigió la config al diseño, no el diseño a la desviación. La llave RSA sobrevivió al renombre (SSH verificado). |
| **2026-07-15** | **B8** | **§7.0 / §7.4 — Política de puerto no usado: 12 puertos a VLAN 888 (`PARKING-NO-USADO`), `switchport mode access` + `nonegotiate` + `shutdown` de puerto y de VLAN** | §7.0 declaraba *"DTP deshabilitado"* y era **falso en 12 puertos**: los libres de `DLS1`/`DLS2`/`MLS1`/`MLS2` estaban en `dynamic auto` · `Negotiation: On` · `Trunking VLANs Enabled: ALL`. El `nonegotiate` se aplicó a las **troncales** —donde se pensó el control— y nunca tocó lo que nadie configuró. Un trunk negociado ahí aterriza en **todas** las VLANs, incluida V30/V41: la única sin ACL de política (§7.2) y la única autorizada a las VTY (§7.3). **888 y no 998** porque 998 es un **typo de distancia de 999**, y ese typo aterriza el puerto parqueado en la nativa — el fallo exacto que el número venía a evitar. Los dos ALS cumplen por ocupación al 100%: **se declara en vez de fabricar puertos para demostrarlo**. Se cerró la **clase**, no la instancia. |
| **2026-07-15** | **B8** | **§7.1 — `40 deny ip any any log` en `INTERSITE-FILTER-IN`, ambos bordes** | El deny implícito **no cuenta y no loguea**: un ping entre las IPs internas del túnel desapareció sin dejar un contador ni una línea de syslog **en ningún equipo**. El deny específico captura lo que el diseño **anticipó**; la línea 40, **lo que no anticipó**. El `log` no tiene acá el costo que tiene en otras listas: el túnel es un **canal autenticado por IPSec** —solo llega lo que el otro borde cifró—, así que el volumen esperado es **cero** y cada evento es señal. **Se pagó sola en 15 minutos: atrapó tres flujos legítimos el mismo día.** |
| **2026-07-15** | **B8** | **§7.3 — VTY asimétrica: los 4 equipos de VALPO aceptan además `10.1.30.0/24` (secuencia 15); STGO sin tocar** | *"Gestión administra todo"* (§7.2) era verdad **solo dentro de cada sede** — la ACL se escribió por sede y nadie preguntó para qué existía el permit inter-sitio del B5. **En VALPO no hay staff de TI** (`00-CONTEXTO`): 4 equipos administrables solo desde una sede sin administradores **no son un control, son equipos inadministrables**. Se eligió asimetría sobre simetría: la dirección que se abre es **la que ya está más protegida**, y Gestión-VALPO sigue sin llegar a las VTY de la matriz (precedente: el B6 ya rechazó forzar simetría en el rol operador). Se descartó el permit por host — el tráfico intra-VLAN es invisible para las ACL, así que la frontera de confianza real es *"estar en la VLAN de gestión"*. **Costo declarado:** comprometer `10.1.30.0/24` pasa de dar 4 equipos a dar 8. |
| **2026-07-15** | **B8** | **§4.3 reescrita + §4.7 nueva — Router-id explícito **sin interfaz** en los 6 equipos; loopbacks de gestión `Loopback30 10.1.30.254/32` (BR-STGO, área 1) y `Loopback41 10.2.41.254/32` (BR-VALPO, área 2), en OSPF** | **3ª divergencia, la más peligrosa:** §4.3 declaraba un `Loopback0 = 10.255.255.x` que **no existe**, y en esos equipos ese nombre está tomado por el **loopback público que ancla la VPN**. En el B6 los dos bordes se reconstruyeron desde cero: una reconstrucción guiada por esa tabla **le habría pisado el anclaje del túnel al equipo**. Los 4 router-id de distribución (`10.1.1.x`/`10.2.1.x`) nunca se habían documentado. **Los loopbacks de gestión (B8-#3b):** `BR-VALPO` era el único de los 8 sin dirección dentro del rango que el filtro deja cruzar → inadministrable desde donde viven los administradores. **Es la decisión del B5 —"el túnel se ancla a un loopback, no a la interfaz de un ISP"— aplicada al plano de gestión:** `10.255.2.2` vive en `Gi3` y desaparece si ese uplink cae, justo cuando lo necesitas. No obliga a tocar el filtro: `10.2.41.254` ya está dentro del permit. `Loopback30` resultó no ser el par simétrico sino **la dirección del servidor NTP de toda la red** (§7.5). **Caveat:** se solapan con la SVI → desde dentro de V41 responden por proxy-ARP (`ttl=254`). |
| **2026-07-15/16** | **B8** | **§1 / §5.1 / §5.2 / §5.3 / §5.4 — Ruta por defecto aprendida por BGP (`neighbor default-originate` en ambos ISP) + NAT con route-map; estática retirada** | **El hallazgo más grande del bloque.** §1 prometía salida resiliente en ambas sedes **y era falso**: una estática a `ISP-1` y el NAT atado a su interfaz → su caída dejaba la sede **sin internet** (el túnel sí sobrevivía, anclado a loopback). **El multihoming protegía la VPN, no el negocio.** Se eligió BGP sobre estática flotante porque (1) **convierte la LocalPref del B5 en política real** —de 2 prefijos /32 a todo el internet de la sede—, (2) detecta por **teardown de sesión**, no solo por link-down: *"ISP muerto con el cable arriba"*, que **el lab reprodujo sin querer**, y (3) es lo que hace una empresa multihomed de verdad. **El NAT con route-map convierte la interfaz de resultado en condición:** sin eso, un paquete que sale por Gi3 se traduce a la IP de Gi2 — sale, nadie vuelve, y todos los `show` se ven bien. **Orden de operaciones:** default-originate → NAT → retirar la estática; los dos primeros con impacto cero por AD, el tercero es el único con riesgo. **Medido:** 129 s de corte y luego internet **sostenido** por ISP-2 (171 pkt a `ttl=252`); y la migración misma con **cero paquetes perdidos y sin un evento en el IGP**. |
| **2026-07-16** | **B8** | Nota de ejecución: **rutas de retorno agregadas en el sustrato de ISP** (cada ISP anuncia sus /30 de cliente) | **Arregla el instrumento de medición, no el diseño.** Tras el primer failover la tabla decía `Tag 65002` y el NAT `NAT-ISP2 refcount 1` —todo verde— **y el usuario sin internet**: vía ISP-2 el PAT traduce a `203.0.113.6`, y **`ISP-1` no tenía ruta de vuelta a `203.0.113.4/30`**. El hueco estaba latente desde el B5 porque **todo el tráfico salía por ISP-1**. Los ISP son **andamiaje del lab, no la red que se entrega**; y sin instrumento no hay medición. **Es la teoría del bloque cobrándose:** tabla + `refcount` probaban que el failover de ruteo y NAT funcionó — **no que el usuario tuviera internet**. |
| **2026-07-16** | **B8** | **§5.2 — Prefix-list `ANUNCIO-PROPIO` en los route-maps `PASS-OUT` y `PREPEND-ISP2-OUT` de los 4 vecinos** | Sin filtro de salida, cada borde anunciaba **7 prefijos** (el suyo + 6 aprendidos del otro ISP, **incluida `0.0.0.0/0`**) con `65010` al frente del AS-path: **AustralPay ofreciéndose como tránsito entre sus dos upstreams**, y ofreciéndole una default a un AS de tránsito. **Existía desde el B5** (`PASS-OUT` era `permit 10` sin `match`) y era invisible porque no había nada que filtrar. **De 7 a 1 en las 4 sesiones.** Es la causa raíz de una familia entera de incidentes reales de BGP, detectada con el comando que nadie corre: `show ip bgp neighbors advertised-routes`. |
| **2026-07-16** | **B8** | **§7.5 nueva — NTP autenticado (MD5), `BR-STGO` estrato 3 sobre `Loopback30`, UTC sin `clock timezone`, `ntp source` obligatorio; los 8 equipos** | **Sin reloj común no hay correlación:** durante el failover no se pudo cruzar el syslog de `ISP-1` con el de `BR-STGO` — **el timeline se reconstruyó con contadores `MsgRcvd`, no con horas**. Es prerequisito del SOC, no infraestructura de fondo. Se eligió el borde sobre un ISP porque **el momento en que más necesitas correlacionar es durante una falla**, y este bloque midió 129 s sin ISP-1. **`ntp source` no es sintaxis:** sin él `MLS1` sourcea de `10.255.2.1` y lo mata el filtro inter-sitio, y `BR-VALPO` sourcearía del túnel y caería en la línea 40. La autenticación es **anti-forense**: mover el reloj borra evidencia sin tocar un log. **Limitación declarada (§11.1):** los 6 vIOS-L2 sincronizan con offset de 1.000-4.750 ms y drift en el tope de 500 ppm; los 2 CSR, a **2,5 ms** con el mismo servidor y camino. La config es correcta; la plataforma no la sostiene. |
| **2026-07-16** | **B8** | **§7.3 — `no ip domain lookup` y `no ip http server` / `no ip http secure-server` en los 8** | `ip domain-lookup` viene **activo por defecto** y no hay `name-server`: **cada typo en la consola generaba 16 consultas DNS por broadcast**, por todas las interfaces — **incluidos los ISP y la VPN**. En una red real es un typo de la consola del borde viajando a internet en claro, y a veces el typo es una contraseña. **Y §7.3 declaraba *"Acceso: solo SSHv2"* y era falso (8ª divergencia):** `ip http server` **e** `ip http secure-server` escuchando en los 8, con `ip http authentication local` —las credenciales de admin— y **sin `ip http access-class`**: `MGMT-VTY-ACCESS` solo aplica a las VTY. **Una promesa de exclusividad no se verifica mirando lo que configuraste, sino lo que está escuchando.** |
| **2026-07-16** | **B8** | **§7.2 — "Nueve listas" corregido a "Siete listas"** | 11ª divergencia. **Nueve** calzaba con las 8 filas de la matriz + la lista `out` de V15: era correcto **antes** de la decisión *"V30 y V41 sin ACL"*, tomada en el mismo bloque y en la misma sección. 9 − 2 = 7, y nadie recontó. **El documento declara un número, una decisión posterior lo invalida, y el número viejo sobrevive porque contar de nuevo no se le ocurre a nadie.** A un entrevistador con la config al lado, sí. |

**Notas de ejecución del Bloque 8** (no cambian el diseño; cambian **cómo se verifica**):

- **`ip domain lookup` con espacio en el CSR1000v** (IOS-XE); el vIOS-L2 acepta el guión. **Misma divergencia que `ip domain name` del B6**: IOS-XE cambió toda la familia `ip domain`. **Y `show run | include ip domain-lookup` hereda el bug**: no encuentra la forma con espacio → **reporta ausente algo que está**. Se verifica con `include ip domain`. **Un comando de verificación que no puede encontrar lo que busca es peor que no verificar: da un falso negativo con cara de dato.**
- **`show interfaces trunk | include Vlans allowed on`** devuelve **solo el encabezado**: `include` filtra línea por línea y la línea con los datos no contiene ese texto. Se usa `begin`. **Un `show` que no puede fallar no es una verificación** — su output vacío se lee como *"verificado"* si no lo miras dos veces.
- **`shutdown` dentro de `vlan 888` sí existe** en vIOS-L2 (`act/lshut`). Séptima sorpresa de plataforma **anticipada que no ocurrió**.
- **Los puertos salen `connected` sin nada enchufado** (artefacto de EVE-NG). Los contadores lo delatan: `53238 packets output, 0 packets input`. En hardware real dirían `notconnect`. Ver §11.2.
- **Las claves tipo 7 con el mismo salt producen el mismo hash.** Dos pares de equipos coincidieron: es la prueba de que la contraseña es **idéntica en los 8**. Si alguno tuviera un typo, no habría coincidido con nadie.

---

## 11. Recomendaciones para producción (fuera del alcance del laboratorio)

Controles que un diseño real incorporaría y que aquí **no se implementan**, con la razón explícita. Se documentan en vez de configurarse a medias o sin poder validarlos.

| Control | Por qué no aquí | Cuándo se justificaría |
|---|---|---|
| **TACACS+ (AAA centralizado)** | A 8 equipos, la gestión local es manejable y el servidor sería un SPOF nuevo que hay que operar en alta disponibilidad. | Cuando el número de equipos/administradores haga inmanejable la gestión caja por caja, o cuando se necesite **accounting central** (rastro de quién tecleó qué) y **autorización por comando** — que es además lo que permitiría dar al operador `shutdown`/`no shutdown` acotado por argumento o por grupo de equipos. |
| **CoPP (Control Plane Policing)** | Un CoPP mal clasificado dropea hellos de OSPF o keepalives de BGP y provoca la caída que venía a evitar. Y **no es validable acá**: probarlo exige generar un flood contra el plano de control, y VPCS solo hace ping. | Siempre en core empresarial y proveedores. Es la respuesta correcta a la preocupación por floods contra la CPU — limita en tasa en vez de romper el diagnóstico. |
| **Private VLAN** | A esta escala ambos invitados cuelgan del mismo switch, donde `switchport protected` resuelve el caso completo. Soporte dudoso en vIOS-L2. | Cuando los invitados se repartan en varios switches de acceso — protected ports solo aísla dentro del mismo switch. |
| **`established` en los retornos TCP** | VPCS no hace TCP: la línea existiría sin poder ejercitarse. | **Obligatorio en producción.** Con solo `echo-reply`, ninguna sesión TCP puede completar su retorno — la app no podría consultar la DB. |
| **ACL por puerto** (`permit tcp ... eq 443`) | No hay servicios escuchando en los VPCS: se vería el contador de la ACL incrementar, nunca el flujo completo. Evidencia débil. | En producción, donde los flujos se conocen y se pueden probar. |
| **Firewall stateful / ZBF** | Es feature de router; un switch L3 no lo hace. Los ACL stateless cubren la política a esta escala. | Cuando se necesite estado de sesión real (cierra el hueco de UDP) e inspección de aplicación. |
| **Permit puntual DB → repositorio interno** | La política actual prohíbe que la DB inicie cualquier cosa, lo que también le impide DNS, NTP y parches. | En producción se abriría un permit específico a un repositorio/NTP interno, no salida general. |
| **`security passwords min-length`** en switches | No soportado en vIOS-L2; la alternativa (`aaa common-criteria`) no evalúa hashes tipo 8/9. | En hardware real que soporte el comando. |
| **Anuncio BGP de un /24 propio** en vez de /32 | En el lab el concepto (anclaje resiliente vía BGP) es idéntico. | En producción un ISP filtra cualquier anuncio más específico que /24. |
| **`ip mtu` / `ip tcp adjust-mss` en `Tunnel0`** | No configurado — hoy el túnel depende 100% de PMTUD. | En producción, para no depender de que ICMP Tipo 3 Código 4 sobreviva todo el camino. |
| **DHCP en servidor dedicado con relay** | El switch como servidor es válido a esta escala y demuestra el concepto. | Lo estándar en empresa: Windows con failover o Infoblox/IPAM, con `ip helper-address` en las SVIs. |
| **Bloqueo de DNS-over-HTTPS en el borde** | Fuera del alcance: exige inspección o listas de resolvers DoH conocidos en el firewall de salida. | **Necesario para que `dns-server 9.9.9.9` sea un control y no una sugerencia.** Firefox y Chrome traen DoH por defecto y **evaden por completo** el resolver entregado por DHCP. Quad9 hoy filtra al que no se esfuerza, no al que se esfuerza — aunque la mayoría del malware *commodity* usa el resolver del sistema. |
| **`port-security aging type inactivity` en invitados** | `maximum 1` sin sticky ya cubre la rotación por link-down, que es el caso normal. | Cierra el hueco del **hub**: un invitado que conecte un switch propio y rote equipos detrás mantiene el link arriba, la MAC dinámica no se libera y el segundo dispositivo violaría el puerto. Con `aging 0` (default) las MAC seguras no expiran nunca. |
| **Resolver DNS interno con filtrado** (Umbrella o equivalente) | No existe infraestructura de DNS interno en el lab: sería un `dns-server` apuntando al vacío, sin checkpoint que lo valide. | En producción, resolución interna filtrada + logs de DNS al SIEM — la telemetría de mayor valor por byte que consume un SOC. |
| **BFD** `[B8]` | El lab **no puede medir la mejora con honestidad**: la unidad de medida es un ping a 1 pps —que acota a ~2 s, no mide— y las VM se congelan y falsean los timers (11.2). Configurarlo sin poder demostrar la diferencia sería decorativo. | **El argumento está medido en este bloque: 129 s (ISP) vs 5 s (uplink de DLS1)** entre dos fallas del mismo tipo. La diferencia es que en la segunda **el link que muere es local al equipo que reacciona** (`track 1 line-protocol`); en la primera hubo que **inferir la muerte del silencio** (hold timer). BFD da detección sub-segundo **independiente del link state** — que es justo el caso *"ISP muerto con el cable arriba"* que el lab reprodujo sin querer, y el que una estática flotante atada a link-down no habría cubierto nunca. |
| **NTP con fuente externa trazable** (GPS o pool público autenticado) `[B8]` | No hay internet real: `ntp master` es la única fuente posible en el laboratorio. | **Siempre.** `ntp master` es un reloj **inventado**: hora relativa correcta, **absoluta falsa** (`reference is 127.127.1.1` / `.LOCL.` **es** la prueba). Para correlación forense **relativa** basta; para forense real —o para cualquier cosa que se compare con el reloj de un tercero: un log de proveedor, una orden judicial, un timestamp de transacción— **no**. En una fintech con PCI-DSS eso deja de ser académico. |
| **Acceso out-of-band** (console server) para los bordes `[B8]` | En EVE-NG la consola **es** out-of-band: el lab no puede reproducir el problema que este control resuelve. | Administrar un borde **a través del túnel que ese mismo borde termina es circular**: si está sano no necesitas entrar con urgencia; **si está roto, el túnel está caído y ninguna ACL te salva**. Es el hueco que la sección 4.7 deja explícitamente abierto y que **ninguna mejora de ACL cierra**. |

### 11.1 Limitaciones de evidencia del laboratorio `[B7 · B8]`

Controles **implementados y correctos** cuya validación el lab no permite completar. Se declaran en vez de omitirse.

| Control | Qué falta probar | Por qué no se puede | Qué sí quedó probado |
|---|---|---|---|
| **Permit de renovación DHCP** (`permit udp any host <SVI> eq bootps`) | Una renovación **T1 real** (unicast del cliente al servidor al 50% del lease) | VPCS **no implementa T1**: `dhcp -r` rehace un DORA broadcast completo, no una renovación. | El **5-tuple por camino equivalente**: UDP:67 on-link a ambas SVI (`10.1.50.2` y `.3`) → **10 matches en cada una**. La ACL no inspecciona payload DHCP, así que el criterio de decisión es idéntico; solo difiere el contenido. |
| **DHCP snooping — servidor rogue** | Que snooping descarte **OFFER/ACK** (mensajes de servidor, puerto 68) llegando por un puerto untrusted | No hay con qué montar un servidor DHCP falso: VPCS no lo hace. `Packets Dropped From untrusted ports = 0` confirma que ese camino no se ejercitó. | Que snooping descarta **DHCP falsificado** desde un puerto de acceso: **15 drops** con la variable aislada (A-B-A: snooping ON → 0 matches en la ACL del DLS; OFF → 10; ON → sin subir). El drop lo produjo `Verification of hwaddr field is enabled`, no el filtro de puerto untrusted. |
| **Violación `restrict`** en puertos de usuario (`maximum 2`) | El err-disable no ocurre; el puerto dropea y cuenta. Exige un **tercer** dispositivo en el mismo puerto. | La topología tiene un PC por puerto de acceso. | La contraparte: **violación `shutdown` en puerto de servidor** (`maximum 1`), con `err-disable`, `Last Source Address 0050.7966.6815:10` y recuperación manual. |
| **Puerto de invitado aceptando un dispositivo nuevo** | La captura del intruso conectándose a `Gi2/0` sin violación | Descartado por costo (dos apagados de `ALS1-STGO`); no cierra un hueco lógico — con `Total: 0` y `maximum 1`, aceptar una MAC nueva es aritmética. | El **mecanismo**: mismo `shutdown`/`no shutdown` en dos puertos, la sticky de `Gi1/0` sobrevive y la dinámica de `Gi2/0` se libera. Variable aislada. |
| **NTP en vIOS-L2** `[B8]` | Que los 6 switches sostengan un orden **sub-segundo**, que es lo que un SIEM necesita para ordenar dos eventos de la misma cadena | **La plataforma, no la config.** Los 6 **sí sincronizan** (`stratum 4`, `loopfilter CTRL`), pero con offset de **1.000-4.750 ms** y `drift` clavado en **500 ppm — el tope del algoritmo**. Causa: **pérdida de polls** (`reach` oscilando entre `1` y `177`; hasta 7 consecutivos) por congelamiento de la VM (11.2). | Los **dos CSR1000v**, con el **mismo servidor y el mismo camino**, sincronizan a **2,5 ms** y 3 ppm — y `BR-VALPO` además **a través del túnel GRE-over-IPSec con MD5**. Misma config, otra imagen, **dos órdenes de magnitud**. La comparación es la prueba: **la config es correcta y funcionaría en hardware real.** |
| **Impacto del failover sobre sesiones NAT existentes** `[B8]` | Que una sesión de larga vida **sobreviva (o no)** al cambio de IP global cuando el ruteo conmuta de ISP — el caveat declarado en 5.4 | **VPCS no hace TCP.** Cada ping crea una traducción **nueva**: PAT usa el campo **Identifier** de la cabecera ICMP como discriminador y VPCS lo incrementa por paquete, así que **no hay flujos de larga vida** que queden atados a la IP global vieja. El caveat es real y **no observable con este instrumento**. | Que **el camino nuevo funciona end-to-end**: 171 paquetes consecutivos a `ttl=252` por ISP-2 tras el failover. Lo que falta medir no es el failover: es **su costo sobre lo que ya estaba abierto**. |

### 11.2 Entorno de laboratorio y preguntas abiertas `[B8]`

Hallazgos que **no son de la red**, y preguntas que **no tienen respuesta**. Se declaran para que ni lo uno se lea como lo otro, ni lo otro se rellene con inferencia.

**Un solo síntoma para seis incidentes.** El Bloque 8 acumuló anomalías que se estaban tratando por separado:

- **Dead timers de OSPF sin evento de red** — `20:49`, `20:55`, y a las `04:35` y `05:17` de **madrugada, sin nadie conectado**
- **`interface resets` y `lost carrier`** en enlaces que nunca se tocaron
- **Relojes NTP** con offset de 1-5 s y `reach` cayendo a `1` (**hasta 7 polls consecutivos perdidos**)
- **`track 1` con 15 transiciones** donde hubo 2
- **Dos VPCS reiniciándose solos**
- **Los dos CSR corrompidos a `grub>`** en el Bloque 6

**Todo es lo mismo: las VM se congelan bajo presión de CPU/memoria del host** (27/32 GB). Un equipo congelado **no manda hellos, no contesta polls y pierde ticks de reloj**. El **CSR1000v aguanta**; el **vIOS-L2 no**.

> **Por qué se declara y no se esconde:** seis síntomas independientes tratados uno por uno son **seis diagnósticos falsos**, y cada uno habría terminado en un cambio de configuración que no arreglaba nada. El diagnóstico correcto es que **ninguno era de la red** — y afirmar eso exige poder mostrar por qué. La prueba es la comparación de 11.1: dos imágenes distintas, mismo servidor NTP, mismo camino, **dos órdenes de magnitud de diferencia**.

**Artefactos de EVE-NG visibles en la evidencia:**

- **Puertos en `connected` sin nada enchufado** — la NIC de QEMU está contra un bridge que está arriba. Los contadores lo delatan: `53238 packets output, 0 packets input`. En hardware real dirían `notconnect`.
- **El `shutdown` de un ISP no baja el link del borde**, por lo mismo. **No es un defecto: es el modo de falla *"ISP muerto con el cable arriba"*** — el argumento de BFD al desnudo (sección 11), y el caso que una estática flotante atada a link-down nunca habría cubierto. **El lab lo reprodujo sin querer, y fue la mejor evidencia del bloque a favor de B8-#4.**

**Preguntas abiertas — declaradas sin respuesta inventada:**

| Pregunta | Estado |
|---|---|
| **UDP 2228** (los 6 switches) y **TCP 21111** (los 2 bordes) en escucha | **Sin identificar.** Candidato del primero: servidor de L2 traceroute de Cisco. **No se afirma sin verificar.** |
| **Por qué unas sesiones SSH se caen y otras no** | Los 8 tienen `exec-timeout 10 0` (7.3). **La causa es otra.** |
| **Cuánto tardó el túnel en re-formar su adyacencia OSPF** tras el failover de ISP | **No medible sin reloj común** — que es exactamente el hallazgo que motivó la sección 7.5. Con hora común, ese mismo log ya sirve. |

> **Una pregunta abierta declarada vale más que una respuesta inferida.** *"Candidato: L2 traceroute"* con la etiqueta de candidato es honesto; sin la etiqueta, es un dato falso que alguien va a citar.

### 11.3 Deltas conocidos: documento ↔ laboratorio `[B8]`

Diferencias **vivas** entre lo que este paquete de entrega describe y lo que hay en el laboratorio. No son limitaciones de evidencia (11.1) ni artefactos del entorno (11.2): son **cosas que no cuadran, encontradas por la propia validación y declaradas sin atenuante**.

| Delta | Detalle | Por qué se declara en vez de arreglarse |
|---|---|---|
| **El nodo `Intruso` sigue en el `.unl`** | MAC `0050.7966.6815`, desconectado tras la prueba de port-security del Bloque 7. **No aparece en el diagrama de entrega.** | El diagrama describe la red **entregada**; el `Intruso` es instrumental de una prueba, no infraestructura. Se declara la diferencia en vez de dejar que el revisor la encuentre solo. |
| **La documentación in-config de VALPO es más pobre que la de STGO** | `Gi1/0` de `MLS1`/`MLS2` no tiene `description` (los de `DLS1`/`DLS2` dicen `Uplink_a_BR-STGO`). Tampoco la tienen `Vlan40`, `Vlan41` ni `Vlan99` de MLS1/MLS2, mientras sus pares de STGO sí (`SVI_Servidores`, `SVI_Gestion`, `Transito_OSPF_DLS1-DLS2`). | **Asimetría de documentación, no de configuración:** ningún control depende de una `description`. Las 8 configs entregadas reflejan el **estado real** y no se maquillan. **Se declara el alcance completo —no solo el `Gi1/0`— porque declarar la mitad de un delta es peor que no declararlo.** |

> **Por qué esta sección existe separada de la 11.2.** La 11.2 dice *"estas anomalías no son culpa de mi red"*. Esta dice *"estas dos cosas no cuadran en mi entrega, y las encontré yo"*. **Son lo contrario**, y mezcladas se leen las dos como lo primero. Un registro de deltas revuelto con explicaciones del entorno **se lee como coartada**; solo, se lee como control de calidad.

---

> ⚠️ **AustralPay y DiegoAraya son ficticias, con fines exclusivos de laboratorio y portafolio. Ninguna configuración, dirección o dato aquí descrito corresponde a una red o cliente real.**
