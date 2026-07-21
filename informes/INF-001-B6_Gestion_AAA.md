> ⚠️ **AustralPay y la consultora DiegoAraya son entidades FICTICIAS, creadas exclusivamente con fines de laboratorio y portafolio. Ningún dato, dispositivo o configuración descrito aquí corresponde a una red real.**

# Informe de Bloque — Redes AustralPay · Bloque 6: Plano de gestión y AAA local
Fecha: 2026-07-14

## Objetivo del bloque

Levantar el plano de gestión de los 8 equipos (SSHv2, usuarios nombrados, separación de roles, endurecimiento de acceso administrativo), resolver el iBGP que quedó pendiente del Bloque 5, y cerrar el diseño de segmentación y servicios que ejecuta el Bloque 7.

---

## Teoría cubierta

### AAA local vs. centralizado (TACACS+/RADIUS)
AAA son tres preguntas separables: Autenticación (¿quién eres?), Autorización (¿qué puedes hacer?), Accounting (¿qué hiciste?). **Local** significa que cada equipo es su propia autoridad, con las credenciales en el `running-config`. **Centralizado** significa que un servidor tiene las identidades y la política.

- **TACACS+** separa A/A/A, hace autorización **por comando** y accounting: es el correcto para administrar dispositivos. **RADIUS** es para acceso a la red (802.1X, VPN), no para admin de routers. Confundirlos es un error típico.
- **Lo que AAA local hace mal:** gestión O(N) —hay que tocar cada equipo para cambiar una clave—, sin fuente única de verdad (drift), sin auditoría central. Los tres son baratos con 8 equipos e inmanejables con 200.
- **Lo que NO pierde:** la seguridad del acceso es idéntica. Secretos hasheados fuertes, SSHv2, ACL de VTY y separación de roles son todos locales. AAA local no es "inseguro", es "no gestionado centralmente" — son ejes distintos.
- **El costo oculto de centralizar:** el servidor TACACS+ es un **SPOF nuevo** que hay que operar en alta disponibilidad. Si cae y el fallback local está mal configurado, quedas fuera de toda la red justo durante el incidente.

**Se eligió AAA local:** a esta escala da autenticación fuerte, SSHv2, ACL de VTY y separación de roles sin agregar un servidor que se vuelve SPOF. El disparador para centralizar es que el número de equipos y administradores haga que gestionar credenciales caja por caja —y la falta de rastro de auditoría central— pesen más que el costo de operar el servidor.

### Niveles de privilegio vs. Parser Views (RBAC)
- **Modelo 1 — Niveles (0–15):** hay 16 niveles, pero por defecto **solo 0/1/15 tienen contenido**. Los niveles 2–14 están **vacíos**: se llenan bajando comandos a mano (`privilege exec level 14 <comando>`). Un usuario en priv 14 sin comandos asignados se comporta como nivel 1. No necesita `aaa new-model`.
- **Modelo 2 — Parser Views:** requiere `aaa new-model`. Define vistas con listas de permiso explícitas (`parser view X` → `commands exec include ...`). El rol de solo-lectura sale trivial (`include all show`).
- **Cuándo cada uno:** el Modelo 1 hace **mal** el solo-lectura —el nivel 1 no ve `show running-config` y construirlo comando por comando es frágil—. El Modelo 2 lo hace natural, pero `aaa new-model` cambia el login por defecto y hay que setear `aaa authentication login default local` antes para no quedarse fuera.

**Se eligió el Modelo 1:** el requisito final fue admin (15) + operador (14) sin rol de solo-vista separado, y el Modelo 1 no exige `aaa new-model` — que además el vIOS-L2 podía no soportar. Los niveles fallan justamente donde no se los necesitó (el solo-lectura), y las views brillan justo ahí.

### La elevación de privilegio es por modo de comando, no global
Cada **modo** de IOS (EXEC, config global, config de interfaz) tiene su **propia tabla de privilegios**. Elevar un comando en un modo no eleva nada en el siguiente.

Para que el operador asigne VLANs hay que abrir la **cadena completa**: `privilege exec level 14 configure terminal` → `privilege configure level 14 interface` → `privilege interface level 14 switchport access vlan`.

Todo lo que **no** se toca queda en 15 **por herencia** — `shutdown`, `ip address`, `router ospf`, `router bgp`, `username`, `crypto key` nunca aparecen como opción, sin necesidad de negarlos. Y de regalo: `show running-config` es nivel 15 de fábrica, así que el operador nunca lo ve aunque no se haya negado. Bien, porque ahí vive el hash del `enable secret`.

### Endurecimiento de acceso administrativo: 4 mecanismos distintos
1. **`service password-encryption`** — cifra (tipo 7, **reversible**, descifrable en segundos) las contraseñas en texto plano del comando viejo `password`. **No mejora los `secret` tipo 8** (hash SHA256 de un solo sentido). Es higiene de bajo costo, no defensa real.
2. **`security passwords min-length N`** — política de longitud mínima. **No es retroactiva**: solo aplica a contraseñas creadas *después*.
3. **Límite de reintentos — son DOS cosas:** `ip ssh authentication-retries N` limita reintentos **dentro de una misma sesión** (anti-typo; un script simplemente abre sesión nueva). `login block-for X attempts Y within Z` cuenta fallos **globales a través de sesiones** y bloquea todo login nuevo — **esta es la defensa real** contra fuerza bruta automatizada.
4. **`login on-failure log` / `on-success log`** — sin esto, un bloqueo no deja rastro de quién lo gatilló.

**Trade-off consciente:** `login block-for` bloquea a **todos**, incluido el admin legítimo — es un DoS auto-infligido posible. Se puede exceptuar una IP con `login quiet-mode access-class`; a esta escala se aceptó sin excepción.

### Los ACL son stateless: cómo se implementa una política unidireccional
Un ACL **no tiene memoria**. "A puede iniciar hacia B pero B no hacia A" **no** se implementa bloqueando B→A: bloquearías las **respuestas** de B al tráfico que A inició, y la política permitida deja de funcionar.

Se implementa permitiendo solo el **tráfico de retorno**, con pistas del encabezado: **`echo-reply`** (ICMP: deja pasar respuestas, no consultas nuevas) y **`established`** (TCP: matchea ACK/RST, o sea sesiones ya iniciadas; un SYN nuevo no matchea).

**UDP no se puede** — no tiene flags, no hay forma de distinguir respuesta de consulta nueva. Ese es el hueco que justifica la existencia de los firewalls stateful.

### Bloquear ICMP en infraestructura: por qué NO
- **El modelo de amenaza que se enseña ya no existe.** *Ping of Death* es de 1996 y está parcheado hace ~25 años. *Smurf* se mata con `no ip directed-broadcast`, **default desde IOS 12.0**. Un **flood volumétrico** consume el ancho de banda **antes** de llegar a tu ACL — se mitiga aguas arriba (scrubbing del ISP). El DDoS moderno es amplificación **UDP** y **SYN flood**; ICMP es marginal.
- **Valor de seguridad ≈ cero:** detiene a alguien tecleando `ping`. Un escáner real (nmap SYN scan) mapea la red igual sin usar ICMP. Es seguridad por oscuridad.
- **El costo es concreto:** **PMTUD depende de ICMP Tipo 3 Código 4.** Bloquearlo no rompe el ping — **cuelga las sesiones TCP grandes**: conectan, transfieren poco y mueren en silencio. En esta red hay **GRE-over-IPSec** (MTU efectivo ~1400 vs 1500 de las LAN), así que todo el tráfico inter-sitio depende de PMTUD. Bloquear ICMP acá es la receta del *black hole* de TCP.
- **Dónde sí:** limitar echo-request **entrante desde internet** hacia IPs públicas es higiene de perímetro razonable, siempre permitiendo Tipo 3 Código 4.
- **La forma adulta del mismo instinto:** bloquear **por política de origen, no por protocolo** — el ACL de Invitados bloquea ICMP como *consecuencia* de bloquear todo hacia adentro. Y si la preocupación real es el plano de control, la respuesta es **CoPP** (limita en tasa hacia la CPU), no romper el diagnóstico.

### Ubicación de ACL: `in` en la SVI de origen
La doctrina de texto ("extended cerca del origen") apunta ahí, pero **su razón clásica —ahorrar ancho de banda— no aplica en un switch**: las ACL corren en TCAM por hardware, `in` y `out` cuestan lo mismo.

Las razones reales son otras: (1) cada ACL se lee como *"qué puede hacer esta VLAN"* → mapea 1:1 con una **fila** de la matriz de política → config **auditable** contra el documento de diseño; (2) descarta antes del lookup de ruteo.

**El gotcha que decide si la política existe:** van en la SVI de **ambos** switches L3. Con HSRP cualquiera puede ser el activo; si la ACL está solo en el activo, **la política se evapora durante el failover** — el momento en que menos quieres perderla.

**La debilidad honesta:** una VLAN nueva sin ACL queda **permitida hacia todo**. Falla abierto ante el olvido. Por eso el activo crítico (V15/DB) lleva además una ACL **`out`** en su propia SVI: una lista blanca de quién puede alcanzarlo, que sobrevive a que alguien agregue una VLAN y olvide su lista.

### Una ACL de SVI filtra también los protocolos de control
Un ACL entrante en una SVI **no solo filtra tráfico de usuarios**: filtra **todo** lo que entra por esa interfaz, incluido lo destinado al propio router.

Los hellos de **HSRP** del switch vecino (multicast `224.0.0.2` UDP 1985 en v1, **`224.0.0.102` en v2** — que es la versión que corre esta red) **entran** por la SVI. Una ACL que termine en `deny ip any any` los descarta → cada switch deja de ver al otro → **ambos se creen Activos** → *split-brain*.

Es el mismo bug que ya evita el `permit ospf any any` que encabeza `INTERSITE-FILTER-IN` en el túnel. **Regla general: toda ACL de SVI que termine en `deny any` necesita un permit explícito del protocolo de control de esa interfaz.** Las ACL **salientes** no filtran tráfico originado por el propio router, así que los hellos que el switch *envía* no se afectan — el problema es siempre el lado entrante del vecino.

### Aislamiento L2: una ACL de SVI no ve el tráfico intra-VLAN
Dos hosts en la **misma VLAN** se hablan por L2 puro, **sin pasar por ninguna SVI** → **ninguna ACL de router puede filtrarlos**. En una red de invitados —donde los dispositivos no son confiables *entre sí*— eso deja el ataque lateral totalmente abierto.

**`switchport protected`:** dos puertos protegidos no se reenvían tráfico entre sí (unicast, multicast **y broadcast**), pero sí hablan con puertos no-protegidos (el uplink). Es el equivalente switch-side del *client isolation* / PSPF de un AP. **Limitación:** solo aísla **dentro del mismo switch**; a escala se usa **Private VLAN** (isolated/community/promiscuous, cruza switches vía PVLAN trunk). Efecto colateral útil: como bloquea broadcast entre puertos protegidos, un DHCP pirata en un invitado **no alcanza** al otro invitado.

### DHCP snooping y el gotcha de Option 82
**Qué resuelve:** un **servidor DHCP pirata** en un puerto de acceso responde primero, se pone como gateway y hace MITM de toda la VLAN. Es *el* ataque clásico contra un segmento con DHCP.

**Cómo:** marca los uplinks como `trust` (por ahí llega el servidor legítimo) y deja los puertos de acceso como *untrusted* por defecto, descartando OFFER/ACK que vengan de un untrusted.

**El gotcha famoso:** al activar snooping, el switch inserta **Option 82** en los paquetes DHCP de puertos untrusted. Como el ALS es L2 y **bridgea** en vez de relayear, el `giaddr` queda en `0.0.0.0`, y el servidor DHCP de IOS **descarta por defecto** los paquetes con Option 82 + `giaddr=0` (huele a suplantación de relay). Resultado: **activas snooping y el DHCP deja de funcionar entero, sin error obvio.** Se resuelve con `no ip dhcp snooping information option`. El `trust` va en la interfaz **Port-channel**, no en los miembros físicos. El binding table que construye es la base de **DAI** e **IP Source Guard**.

### Redundancia de DHCP: split-scope
**IOS no tiene DHCP failover.** Windows Server (2012+) e ISC sí tienen un protocolo real donde los servidores comparten estado de leases. En un switch Cisco, **split-scope es la única redundancia posible**, no un atajo.

- **Costos reales:** sin estado compartido (cada switch conoce solo sus leases → diagnóstico en dos lugares); **capacidad a la mitad** durante una falla; asignación no determinista (ambos responden, el cliente toma la primera OFFER).
- **Por qué DHCP no merece la misma redundancia que HSRP:** **una caída de DHCP no es una caída de red.** Los clientes con lease vigente siguen funcionando por días; solo se afectan los que piden IP nueva. HSRP cae y el tráfico muere al instante — el impacto de DHCP es diferido y menor.
- **En el mundo real no se hace ninguna de las dos:** las empresas corren DHCP en servidores dedicados (Windows con failover, Infoblox/IPAM) y los switches solo **relayean** con `ip helper-address`. Split-scope en el switch es correcto *cuando el switch es el servidor*.

---

## Qué se hizo

**Decisión de iBGP (pendiente del Bloque 5)**
- Analizado y **omitido formalmente** del diseño, con justificación documentada (ver Decisiones).

**Plano de gestión — los 8 equipos**
- `ip domain-name` / `ip domain name` + llave RSA 2048 + `ip ssh version 2`.
- `enable secret` y usuarios `admin` (priv 15) y `operador` (priv 14), todos con hash **tipo 8 (SHA256)**.
- `line vty 0 15`: `transport input ssh`, `login local`, `exec-timeout 10 0`, `access-class MGMT-VTY-ACCESS in`.
- ACL de VTY: permit de la VLAN de gestión de la sede (`10.1.30.0/24` en STGO, `10.2.41.0/24` en VALPO), `deny any log`.
- Endurecimiento: `service password-encryption`, `login block-for 120 attempts 3 within 60`, `login delay 1`, `login on-failure log`, `login on-success log`.
- `security passwords min-length 9` **solo en los bordes** (no soportado en vIOS-L2).
- SVI de gestión nueva en los switches de acceso: `ALS1-STGO` → `10.1.30.10`, `ALS1-VALPO` → `10.2.41.10`, con `ip default-gateway` al VIP de HSRP.

**Rol operador — tres perfiles según rol del equipo**
- **Switch de acceso (ALS):** cadena de privilegio completa (`exec → configure → interface → switchport access vlan`). Gestiona VLANs; **sin** `shutdown`/`no shutdown`, `ip address`, routing ni `username`.
- **Distribución (DLS/MLS):** rol **deliberadamente angosto** — no tienen puertos de acceso de usuario, así que no hay nada "local" que darle. Solo-lectura con cuenta nombrada (trazable); ve HSRP gratis (`show standby` es nivel 1).
- **Bordes (CSR):** solo-lectura total. Un borde no tiene VLAN/DHCP/HSRP; todo lo que tiene (BGP, NAT, crypto) es terreno del admin.

**Reconstrucción de los dos bordes tras incidente de disco** (ver Problemas).

**Correcciones de configuración**
- `no ip routing` en ambos ALS.
- Retirada la ruta por defecto duplicada (`ip route 0.0.0.0 0.0.0.0 GigabitEthernetX`) de ambos bordes.

**Diseño cerrado para el Bloque 7:** VLANs 15/50/55, matrices de política intra-sitio de ambas sedes, DHCP split-scope, protected ports, DHCP snooping. Ver `06-DISENO-RED-AustralPay.md` secciones 3, 4.6, 7.2 y 7.4.

---

## Decisiones y por qué

### iBGP omitido
Se eligió **no implementarlo y documentar el porqué**, en vez de configurarlo para cumplir la pauta.

BGP anuncia **solo 2 prefijos** (los loopbacks públicos /32 que anclan la VPN). Cada borde ya aprende el loopback del otro **por eBGP** vía los ISP (con `allowas-in`). El transporte interno RFC 1918 va por **OSPF sobre GRE**. La salida a internet es por `default-information originate`, no por la tabla BGP. Una sesión iBGP cargaría exactamente los mismos 2 loopbacks que ya se ven por eBGP: **redundante**. Ni siquiera podría ayudar al anclaje del túnel — el túnel se ancla en esos loopbacks, así que no puede aprenderlos "por encima de sí mismo" (huevo y gallina). La pauta pedía iBGP; en este diseño no carga nada que eBGP y OSPF no den ya, así que se documentó por qué se omitió en vez de configurar un protocolo muerto.

### El operador NO recibe `shutdown`/`no shutdown`
**La tensión:** se quería que el operador pudiera levantar un puerto err-disabled (recuperación sin llamar al admin). Pero **en el Modelo 1 el privilegio es por comando, no por interfaz**: darle `no shutdown` para el puerto del servidor le daría también apagar el uplink que sostiene OSPF. No existe "puede hacer `no shut` en Gi1/2 pero no en Gi1/0".

**La decisión:** negárselo. Una violación de port-security en un **servidor** es un **evento de seguridad**, no un tropiezo operativo — que la recuperación **escale al admin** es lo que debe pasar, no una pérdida de capacidad. Lo que sí conserva: entra a `interface` para su trabajo de VLAN (`switchport access vlan`); los niveles **sí** separan subcomandos, solo no separan *cuál* interfaz.

**Cómo se hace en sitios reales:** con **TACACS+** se resuelve por match de argumento (`permit interface Gi1/2`, `deny interface Gi1/0`) o por **grupo de equipos**. Sin eso, el control real en producción suele ser auditoría + entrenamiento + gestión de cambios, no prevención técnica. Es uno de los disparadores concretos para centralizar AAA.

### El rol operador no es idéntico en todos los equipos
Se rechazó forzar simetría. En `DLS1/DLS2/MLS1/MLS2` **no hay puertos de acceso de usuario** (todo es trunk o L3 punto a punto), así que darles `switchport access vlan` sería abrir un comando que nunca tiene con qué ejecutarse — privilegio decorativo. En el switch de acceso el operador gestiona VLANs; en distribución queda con visibilidad de HSRP hasta que el DHCP del Bloque 7 le dé algo que administrar.

### `security passwords min-length`: aplicar donde la plataforma lo soporte
**La tensión:** no existe en la imagen vIOS-L2 (los 6 switches); sí en CSR1000v (los 2 bordes). La alternativa documentada por Cisco (`aaa common-criteria policy`) exige `aaa new-model` **y** tiene un gap conocido: los usuarios creados con `secret 8`/`secret 9` no se evalúan contra la política — que es exactamente el hash usado acá.

**La decisión:** aplicarlo donde existe; en los switches queda como **regla operativa** (las contraseñas ya superan 9 caracteres) documentada como límite de plataforma. Se prefiere un control que se sabe que funciona a uno que aparenta funcionar.

### El ping directo entre IPs internas del túnel queda bloqueado — aceptado
`INTERSITE-FILTER-IN` solo permite OSPF y Gestión↔Gestión. Un ping `10.255.0.1`↔`10.255.0.2` no matchea ninguno → cae en el deny.

**No es una regresión:** el ping de túnel exitoso documentado en el Bloque 5 ocurrió en el **Punto 3**, **antes** de que el **Punto 5** aplicara la ACL sobre `Tunnel0`. Nunca se volvió a correr después. Se optó por no agregar excepción: el diagnóstico de salud del túnel pasa por la **adyacencia OSPF en FULL** (una adyacencia no sube si el túnel no pasa tráfico real), no por ping directo. (En el Bloque 8 se agregó `deny ip any any log` como línea final del filtro, de modo que ese drop dejó de ser silencioso.)

### Alcance de la política intra-sitio: se filtra el interior, no solo el perímetro
**El hallazgo:** el proyecto filtraba el tráfico **inter-sitio** —que cruza cifrado por un túnel, el camino *más* protegido— y dejaba **abierto el intra-sitio**, donde vive el 90% del tráfico real. Una fintech donde cualquier estación alcanza la base de datos sin nada que lo impida. Es el patrón clásico: perímetro protegido, interior plano.

**La decisión:** **lista blanca (default-deny)** — con lista negra solo bloqueas lo que pensaste; con lista blanca **lo que olvidaste queda bloqueado**. Falla cerrado. Granularidad host, no puerto: V20 llega al **app**, nunca al **db** — es la política real de una fintech (las estaciones hablan con la aplicación, jamás con la base de datos directo) y es 100% validable con ping. La granularidad por puerto (`permit tcp ... eq 443`) sería lo correcto en producción, pero no es validable acá: se vería el contador incrementar, nunca el flujo completo. Se documenta como recomendación de producción en vez de fingir evidencia.

### Separación de `austral-app` y `austral-db` en VLANs distintas
**El problema:** compartían la VLAN 10 → el tráfico app→db era **L2 puro**, no pasaba por ninguna SVI, y **ninguna ACL podía filtrarlo**. Quedaba permitido por física, no por política.

**La decisión:** VLAN 15 nueva para la DB. El salto app→db ahora cruza una SVI y es filtrable. Y **la DB no inicia nada hacia afuera:** una base de datos abriendo conexiones salientes es un indicador clásico de exfiltración. En términos SOC no es solo una regla de ACL — es una **regla de detección**.

### VLAN de Invitados: el entregable es la política, no la VLAN
Agregar una VLAN de invitados **sin** el ACL que la aísla sería una VLAN de invitados que llega a la base de datos: decorativa y peor que no tenerla.

**Nombre:** se descartó "Wifi" — no hay AP en el laboratorio y la primera repregunta de una entrevista desarma la afirmación. La config switch-side es **idéntica** con o sin AP (el AP se conecta por un puerto y el switch no sabe que del otro lado hay radio), así que configurarlo no es artificial; lo artificial sería *decir que se probó wireless*. `Invitados` es honesto y no necesita asterisco.

**Port-security en puertos de invitados:** MAC sticky **sería contraproducente** — un puerto de invitados existe para que roten dispositivos distintos. `maximum 1` + `restrict` sin sticky: impide que enchufen un switch/hub y multipliquen invitados, y acepta un dispositivo nuevo cuando el anterior se va.

### CoPP: real, pero fuera de alcance
Es estándar en core empresarial y proveedores: aplica una política de QoS al **plano de control**, limitando en tasa lo que llega a la CPU. Un flood no tumba el equipo porque la CPU nunca ve más de X pps. **Por qué no entra:** un CoPP mal clasificado **dropea hellos de OSPF o keepalives de BGP** y provoca la caída que venía a evitar. Y no es validable acá — probarlo exige generar un flood contra el plano de control, y VPCS solo hace ping. Sería config sin evidencia.

---

## Evidencia capturada

- `16-als1-stgo-ssh-enabled.png` — `show ip ssh` con SSH v2 habilitado + llave RSA generada.
- `17-als1-stgo-usernames.png` — `show run | include username` con `admin` priv 15 y `operador` priv 14, hash tipo 8.
- `18-als1-stgo-login-hardening.png` — `show login`: block-for 120s tras 3 fallos en 60s, delay 1s, logging de éxito y fallo.
- `19-als1-stgo-vty-acl.png` — `show ip access-lists MGMT-VTY-ACCESS` con matches reales en el permit de la VLAN de gestión.
- `20-als1-stgo-operador-priv14.png` — SSH como `operador`: `show privilege` = 14; **`show running-config` denegado** (nivel 15 de fábrica, sin configurarlo).
- `21-als1-stgo-operador-permit-deny.png` — en una sola sesión: `switchport access vlan 20` **ejecuta** (permit) · `shutdown`, `router ospf 1`, `username test privilege 15` los tres **`% Invalid input`** (3 denies).
- `22-dls1-stgo-gestion.png` — `show ip ssh`, usuarios, `show standby brief` (V10 Active, V20 Standby, V30 Active).
- `23-dls1-stgo-operador-readonly.png` — operador: priv 14 · `show standby brief` **funciona** (nivel 1 gratis) · `configure terminal` **denegado** (rol angosto verificado).
- `24-dls2-stgo-operador-readonly.png` — ídem, con V20 en `Active local` (root de DLS2) — cruce coherente con el Bloque 2.
- `25-mls1-valpo-gestion.png` / `26-mls2-valpo-gestion.png` — SSH, usuarios, HSRP (V40 Active en MLS1 / V41 Active en MLS2).
- `27-valpo-operador-cruzado.png` — verificación **cruzada**: MLS1→MLS2 y MLS2→MLS1, ambos priv 14, `standby brief` invertido según perspectiva, `configure terminal` denegado en ambos.
- `28-als1-valpo-operador-permit-deny.png` — mismo paquete permit + 3 denies que ALS1-STGO.
- `29-br-stgo-reconstruccion.png` / `30-br-valpo-reconstruccion.png` — post-reconstrucción: `show ip bgp summary` (2 vecinos Established, PfxRcd=2) · `show crypto isakmp sa` (QM_IDLE/ACTIVE) · `show ip ospf neighbor` (**3 adyacencias FULL**: Tunnel0 área 0 + 2 uplinks a distribución) · `show ip access-lists INTERSITE-FILTER-IN` (44/48 matches reales en el permit de OSPF) · `show ip ssh` con llave regenerada.
- `31-bordes-ping-loopback.png` — `ping 192.0.2.101 source loopback0` desde BR-STGO y su inversa: **5/5** en ambos (alcance BGP punta a punta sobre los ISP).
- `32-br-stgo-operador-readonly.png` — operador en el borde: priv 14, `configure terminal` denegado.
- `33-als1-ip-routing-anomalia.png` — `show ip route` en ALS1-STGO mostrando **tabla de ruteo y "Gateway of last resort is not set"** en un switch de acceso + ping a `10.255.1.2` fallando.
- `34-als1-no-ip-routing-fix.png` — `no ip routing` + `ping 10.255.1.2` **5/5** (primer paquete 1004 ms = ARP resolviéndose al abrirse por fin la ruta).

---

## Problemas / aprendizajes

### `ip routing` activo por defecto en vIOS-L2: el gateway fantasma
**Síntoma:** el SSH del operador desde `ALS1-STGO` hacia el borde daba `% Destination unreachable; gateway or host down`, pero el ping al VIP de gestión (`10.1.30.1`) daba 5/5 y el enlace DLS1↔BR-STGO estaba sano en ambos sentidos.

**Causa raíz:** `show ip route` en el ALS mostró una tabla de ruteo y *"Gateway of last resort is not set"* — un switch L2 puro no muestra eso. **`ip routing` estaba activo**, y con `ip routing` activo IOS **ignora por completo `ip default-gateway`**. Sin ruta por defecto propia, cualquier paquete fuera de `10.1.30.0/24` no tenía por dónde salir y moría en el propio ALS.

**Por qué apareció recién ahora:** todo lo que el ALS había hecho hasta ese punto fue **dentro** de su subred (SSH a DLS1/DLS2, ARP directo). El SSH al borde fue **el primer tráfico fuera de subred** de todo el proyecto.

**Origen verificado:** `ip routing` aparece en los informes **solo** para DLS1/DLS2 (Bloque 3) y MLS1/MLS2 (Bloque 4) — los switches L3. Ninguna mención de haberlo configurado en los ALS; el informe del Bloque 2 lo lista explícitamente como pendiente para el Bloque 3. **Conclusión: viene activo por defecto en la imagen vIOS-L2.** La corrección es también un fix de diseño: un switch de acceso ruteando es una anomalía. `no ip routing` lo devuelve a L2 puro y el `ip default-gateway` ya configurado vuelve a tomar efecto solo, sin agregar nada. **No falla al configurarlo — falla la primera vez que el switch intenta salir de su subred, semanas después.**

### Incidente: corrupción de disco en AMBOS bordes
**Síntoma:** `BR-STGO` y `BR-VALPO` arrancaron al prompt `grub>` en vez de IOS-XE. `confreg` devolvió *"Failed to open confreg"*, ESC no mostró el menú de arranque, y `ls` respondió *"not a valid command"* — el filesystem no respondía. Un CSR1000v nuevo de prueba **sí** arrancó → corrupción de esas dos instancias, no de la imagen maestra.

**Recuperación:** reconstruidos a partir de los **session logs de PuTTY**, que habían capturado `show running-config` completo de ambos equipos antes del incidente. **Fuente autoritativa, no inferencia.**

**La lección que pagó dividendos:** una primera reconstrucción hecha *de memoria + documento de diseño* difería del estado real en tres puntos — había **4 route-maps y no 2** (`LOCALPREF-ISP1-IN`, `LOCALPREF-ISP2-IN`, `PREPEND-ISP2-OUT`, `PASS-OUT`), el deny del filtro inter-sitio era **específico** (`10.2.0.0/16 → 10.1.0.0/16 log`) y no genérico, y arrastraba una **ruta por defecto duplicada**. El log ganó. Se reconstruyó desde el `show running-config` real y no desde la propia memoria del diseño, y aparecieron tres diferencias que se habrían metido mal.

**Hipótesis del origen:** el host corría con **27 de 32 GB de RAM en uso**. Presión de memoria más I/O es causa conocida de apagados no-graceful en EVE-NG, y así se corrompe un disco virtual. Que cayeran los dos CSR a la vez —los nodos más pesados, ~3-4 GB cada uno— encaja. No confirmado, pero es el sospechoso. (El Bloque 8 volvió a ver este mismo patrón de congelamiento de VM bajo presión de memoria, ya como causa raíz reconocida de media docena de anomalías — ver §11.2 del diseño maestro.)

### La ruta por defecto duplicada que nunca se limpió
Ambos bordes arrastraban `ip route 0.0.0.0 0.0.0.0 GigabitEthernetX` (interfaz) **junto a** `ip route 0.0.0.0 0.0.0.0 <next-hop-IP>`. Misma distancia administrativa → IOS las trata como equivalentes.

Es el mismo problema de *recursive lookup* que el Bloque 5 documentó como "corregido": se agregó la versión correcta pero **nunca se hizo `no ip route` de la vieja**. Detectado al leer el `show running-config` completo durante la reconstrucción. **Aprendizaje: "corregir" una config no es solo agregar lo correcto — es retirar lo incorrecto.** Un `show run` completo lo destapa; un `show ip route` no necesariamente.

### `ip domain name` vs `ip domain-name`: fallo silencioso en cadena
En **CSR1000v (IOS-XE 17.3)** el comando es `ip domain name` (con **espacio**). La forma con guión —que el vIOS-L2 sí acepta— devuelve `% Invalid input`.

**El fallo en cadena:** el dominio nunca quedó fijado → `crypto key generate rsa` falló con `% Please define a domain-name first` → **no se generó llave RSA** → SSH configurado (`ip ssh version 2` y los usuarios sí se aplicaron) pero **inoperativo**. **Aprendizaje:** un error de sintaxis en medio de un pegado masivo puede dejar el equipo en un estado *parcialmente* configurado que parece correcto. Verificar el resultado (`show ip ssh` + `show crypto key mypubkey rsa`), no el hecho de haber pegado. (El Bloque 8 encontró la contracara de esto: `show run | include ip domain-lookup` no encuentra la forma con espacio y reporta ausente algo que está.)

### El ping de consola del borde que engañó el diagnóstico
Durante el diagnóstico del SSH, `ping 10.1.30.10` **desde la consola de BR-STGO** fallaba (0/5) aunque el enlace estaba sano. Causa: el paquete se **origina en el propio router** y sale por una interfaz marcada `ip nat inside` hacia una red que la ACL de NAT considera candidata a traducción — caso borde de IOS que descarta o rompe el retorno.

**Eran dos problemas superpuestos** (este + el `ip routing` del ALS), lo que hizo el aislamiento mucho más difícil. El tráfico **de tránsito** (ALS→borde) nunca sufrió esto; solo el **originado por el router**. **Aprendizaje:** un ping desde la consola de un router NAT no prueba lo mismo que un ping de un host real. Cuando el diagnóstico se contradice, cambiar el punto de origen de la prueba antes que la teoría.

### El bug que se detectó en diseño, antes de configurarlo
La ACL `in` de V15 (DB) terminaba en `deny ip any any` según la matriz (la DB no inicia nada). Eso habría descartado los **hellos de HSRP** del DLS vecino que llegan por esa misma SVI → **split-brain en la VLAN de la base de datos**, con síntoma intermitente y difícil de diagnosticar.

Detectado **escribiendo las listas para el documento**, antes de tocar el lab. Corregido con `permit udp any host 224.0.0.102 eq 1985` antes del deny — el multicast de HSRPv2, que es la versión que corre la red. Es el mismo bug que el `permit ospf any any` del túnel ya evitaba: una ACL en una interfaz que transporta un protocolo de control debe permitirlo explícitamente.

---

## Cierre del bloque: deuda detectada, y dónde se resolvió

Este bloque dejó una lista de discrepancias entre el diseño y el estado real de los equipos — algunas descubiertas al validar el plano de gestión, otras al reconstruir los bordes. **Todas se resolvieron en el Bloque 7**, y se dejan registradas acá porque el patrón —el diseño dice una cosa, la config tiene otra— es el mismo que el Bloque 8 formalizaría como sus once divergencias:

- **`no ip routing` faltaba en ALS1-VALPO** (se había aplicado solo en STGO al detectar el gateway fantasma) → aplicado en el B7.
- **La línea de consola (`line con 0`) estaba sin proteger en los 8 equipos** — sin `login local`, alcanzable por quien tenga acceso físico → cerrada en los 8 en el B7.
- **El hostname real de ALS1-VALPO** mostraba `ALS1` en el prompt, no `ALS1-VALPO` → corregido en el B7 (la llave RSA sobrevivió al renombre).
- **Port-security en modo `shutdown` para los puertos de servidor** estaba diseñado y nunca implementado → resuelto en el B7 (coincide con el default de IOS: la decisión existe aunque no se vea en el `running-config`).
- **`portfast` + BPDU guard en los puertos de acceso** — pendiente de verificar del B2 → confirmado presente.
- **Los módulos Gi2 en los ALS y los 8 PCs con `set pcname` + IP** — infraestructura de laboratorio necesaria para las pruebas del B8 → montada en el B7.
- **El test de deny del ACL de VTY** (origen fuera de la VLAN de gestión) → ejecutado en el B8, dentro de la matriz de conectividad de 59 celdas.
