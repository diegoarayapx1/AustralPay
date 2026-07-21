# Informe de Bloque — Redes AustralPay · Bloque 5: Borde y WAN
Fecha: 2026-07-13

## Objetivo del bloque
Desplegar los routers de borde `BR-STGO` y `BR-VALPO`: eBGP multihoming hacia
dos ISP compartidos por ambas sedes, VPN GRE-over-IPSec site-to-site anclada
a loopbacks públicos resilientes, OSPF área 0 sobre el túnel cerrando las
adyacencias pendientes desde los Bloques 3 y 4, NAT/PAT con exclusión del
tráfico VPN, un filtro de seguridad inter-sitio a medida, y manipulación de
atributos BGP para fijar una política de salida/entrada explícita.

## Teoría cubierta

### BGP: qué es y qué problema resuelve
Todo lo configurado hasta el Bloque 4 —VLANs, HSRP, OSPF— opera dentro de un dominio administrativo único, bajo un mismo control. Internet no es eso: son miles de redes independientes (Sistemas Autónomos) que no confían ciegamente entre sí y no comparten una métrica común.

BGP resuelve eso con **vector de ruta** (AS-path, no distancia) y **decisiones por política configurable** (LocalPref, MED, etc.), en vez del "camino más corto" automático de un IGP. Un IGP asume una topología confiable; BGP asume que cada vecino puede tener intereses distintos.

### iBGP vs eBGP
- **eBGP** (entre AS distintos): TTL=1 por defecto, antepone el AS propio al anunciar, cambia el next-hop automáticamente.
- **iBGP** (mismo AS): sin límite de TTL, no modifica el AS-path entre pares, **no** cambia el next-hop por defecto (requiere `next-hop-self`), y tiene split-horizon —una ruta aprendida de un peer iBGP no se reanuncia a otro peer iBGP— que a gran escala se resuelve con route-reflectors.

En este bloque: **eBGP en los 4 enlaces borde↔ISP.** El diseño original contemplaba además iBGP entre `BR-STGO`↔`BR-VALPO` sobre los loopbacks; ese peering **se eliminó en el Bloque 6** al comprobar que la comunicación entre bordes ya la resuelve la adyacencia OSPF de área 0 sobre el túnel, sin necesidad de una sesión iBGP paralela. Con solo dos routers en el AS, el split-horizon iBGP nunca fue un problema práctico —es la razón de ser de los route-reflectors en redes grandes— pero la sesión tampoco aportaba nada que OSPF no diera ya.

### Multihoming y atributos BGP: cuál controla qué
**Multihoming** es un AS con más de una conexión a internet — acá, a dos ISP por sede. Da resiliencia y control de tráfico, a cambio de tener que manipular atributos de forma explícita.

| Atributo | Alcance | Controla | Regla |
|---|---|---|---|
| **Weight** | solo local (propietario Cisco), no se propaga | salida | más alto gana |
| **Local Preference** | se propaga dentro del AS | **salida** — es mío, se respeta siempre | más alto gana |
| **AS-Path prepend** | se propaga a otros AS | **entrada** — alarga el path visto por otros | es una súplica, no una orden |
| **MED** | hacia un AS vecino directo | entrada — el vecino decide si lo respeta | más bajo gana |

**El resumen operativo:** LocalPref controla cómo salgo yo —mi decisión, siempre se respeta—; AS-path prepend y MED son sugerencias para influir cómo el resto de internet entra hacia mí, y el vecino puede ignorarlas.

Orden de decisión de BGP (resumen): Weight → LocalPref → AS-path más corto → origen → MED → eBGP sobre iBGP → métrica IGP al next-hop.

### ISAKMP (fase 1) vs IPSec (fase 2)
- **Fase 1 (ISAKMP/IKE):** protege el propio canal de negociación y produce la ISAKMP SA. Autentica a los dos peers entre sí (PSK o certificados). Modo típico site-to-site: main mode.
- **Fase 2 (IPSec):** protege el tráfico de datos real y produce las SAs de datos, una por dirección. Hereda la confianza de fase 1. Modo típico: quick mode.

En una frase: la fase 1 es dos partes verificando identidad y acordando un idioma cifrado; la fase 2 es la conversación real, ya en ese idioma. Sin fase 1 no hay con quién negociar la fase 2 de forma segura, por eso siempre va primero.

### Por qué GRE-over-IPSec
- **Solo GRE:** da la interfaz ruteable con soporte multicast (necesario para los hellos de OSPF a 224.0.0.5), pero **viaja en claro** — inaceptable con PCI-DSS de por medio.
- **Solo IPSec (crypto map):** cifra bien, pero un crypto map **no levanta una interfaz** — no hay dónde correr OSPF.
- **GRE dentro de IPSec:** el paquete GRE completo —con la IP interna de tránsito y OSPF adentro— se encapsula, y **eso** se cifra. Interfaz ruteable **y** protegida a la vez.

El crypto map protege tráfico, no crea rutas. GRE aporta el `Tunnel0` como interfaz L3 real donde corre OSPF área 0; IPSec protege ese túnel de extremo a extremo. Ninguno hace el trabajo del otro.

### Transport mode vs Tunnel mode (IPSec)
- **Transport:** IPSec inserta su header (ESP) tras el header IP original, sin agregar un segundo header IP. Menor overhead; exige que los extremos IPSec coincidan exactamente con los del paquete protegido.
- **Tunnel:** IPSec envuelve el paquete completo —incluido el header IP original— en un nuevo header IP. Mayor overhead (+20 bytes); no exige que los extremos coincidan, sirve incluso con NAT o proxy en medio. Es el default de Cisco IOS.

**Se eligió tunnel mode.** El argumento correcto **no es "más cifrado"**: el algoritmo de cifrado e integridad es idéntico en ambos modos, y la única diferencia es qué headers IP quedan visibles fuera del paquete cifrado. En un p2p directo como este, ocultar esas IPs no aporta protección real porque no hay un tercero a quien confundir. Se eligió por ser el estándar más reconocible para site-to-site y por no atar los endpoints IPSec al paquete — no por seguridad del payload, que es la misma.

### NAT/PAT en el borde
- **PAT (overload):** muchas IPs internas comparten una única IP pública, multiplexando por puerto. Estándar en cualquier borde corporativo con IPv4 escaso.
- **El interior nunca se anuncia por BGP:** es RFC 1918 —no pasaría los filtros bogon de un ISP real— y anunciarlo contradice la premisa del NAT (el interior no es directamente alcanzable desde afuera).
- **La ACL de NAT excluye el tráfico inter-sitio** (10.1↔10.2) para que viaje con IP real dentro del túnel cifrado, no natteado.

Lo único que se anuncia es el loopback público que ancla la VPN.

### Deslinde de mecanismos: crypto ACL vs `tunnel protection` vs ZPF
- **Crypto ACL ("interesting traffic"):** mecanismo del crypto map clásico — le dice a IPSec qué cifrar cuando el crypto map va sobre una interfaz física. Descartado desde el Bloque 1 junto con el crypto map puro, precisamente porque no transporta el multicast de OSPF.
- **`tunnel protection ipsec profile`:** el mecanismo que sí se usa con GRE-over-IPSec. No necesita ACL de "tráfico interesante": todo lo que entra a `Tunnel0` se encapsula en GRE primero, y el profile protege el GRE en sí.
- **ZPF (Zone-Based Policy Firewall):** firewall **stateful** de IOS (zonas, class-map/policy-map type inspect). No se usó. El filtro inter-sitio aplicado es un **ACL extendido clásico, stateless** —sin memoria de conexión, por eso se escribió simétrico en ambos bordes—. ZPF sería la evolución natural si se quisiera estado.

## Qué se hizo

**Punto 1 — Sustrato de internet:**
- Dos routers ISP (`ISP-1` AS 65001, `ISP-2` AS 65002), cada uno con doble
  rol: ISP-1 conecta a `BR-STGO` (`203.0.113.0/30`) y a `BR-VALPO`
  (`198.51.100.0/30`); mismo patrón para ISP-2 en los otros dos /30.
  Peering ISP-1↔ISP-2 en `192.0.2.0/30`.
- Reachability directa verificada en los 5 enlaces (pings 4-5/5, primer
  paquete perdido por ARP inicial, firma ya conocida de bloques previos).

**Punto 2 — eBGP multihoming + loopbacks públicos:**
- Loopback0 público en cada borde: `192.0.2.100/32` (BR-STGO),
  `192.0.2.101/32` (BR-VALPO), únicos prefijos anunciados por `network`.
- `router bgp 65010` en ambos bordes, 2 vecinos eBGP cada uno (ISP-1,
  ISP-2). `router bgp` en ISP-1/ISP-2, cada uno con 3 vecinos (los 2
  bordes + el otro ISP).
- Se detectó que el anuncio de un borde no llegaba al otro (`PfxRcd: 0`) —
  ver Decisión #4 y Problemas/aprendizajes. Resuelto con `allowas-in 1` en
  los 4 vecinos de los bordes; `PfxRcd` pasó a 2 en ambos, reachability
  loopback-a-loopback confirmada (5/5, dos caminos BGP disponibles).

**Punto 3 — VPN GRE-over-IPSec:**
- ISAKMP policy 10 (AES-256, SHA-256, PSK, group 14) + transform-set
  `AUSTRALPAY-TS` (esp-aes 256, esp-sha256-hmac, modo tunnel) + profile
  `AUSTRALPAY-PROFILE`.
- `Tunnel0` en ambos bordes: `tunnel source Loopback0`,
  `tunnel destination <loopback remoto>`, `tunnel protection ipsec
  profile`. IP de túnel `10.255.0.1/30` (BR-STGO) y `10.255.0.2/30`
  (BR-VALPO).
- Verificado: `Tunnel0 up/up` en ambos extremos, ISAKMP `QM_IDLE ACTIVE`,
  IPSec con `#pkts encaps/decaps` simétricos (20/20), ping cifrado
  extremo a extremo 5/5.

**Punto 4 — OSPF área 0 + cierre de adyacencias pendientes:**
- `Tunnel0` en área 0 en ambos bordes; interfaces borde↔distribución
  (`Gi1`/`Gi4` en BR-STGO hacia DLS1/DLS2, `Gi3`/`Gi4` en BR-VALPO hacia
  MLS1/MLS2) en área 1/área 2 respectivamente, con
  `ip ospf network point-to-point`.
- Se corrigió una asimetría heredada: a `DLS1-STGO`/`DLS2-STGO` les
  faltaba aplicar `ip ospf network point-to-point` en `Gi1/0` (quedó
  pendiente explícitamente en el informe de Bloque 3) — sin eso, la
  adyacencia subía como `FULL/BDR` (broadcast con DR/BDR) en vez de
  `FULL/-` (p2p). Corregido; las 6 adyacencias del diseño (2 en área 1, 2
  en área 2, 1 en área 0 — más la ya existente Vlan99 en cada sitio)
  quedaron `FULL/-`, simétricas entre STGO y VALPO.
- Ambos bordes confirmados como ABR reales: una pata en área 0 (túnel),
  una pata en el área de su sitio (Gi1/0 dual-homed a distribución).

**Punto 5 — NAT/PAT + ACL de exclusión + filtro inter-sitio (extensión):**
- PAT/overload en cada borde hacia la interfaz de ISP-1
  (`ip nat inside source list NAT-EXCLUDE-VPN interface Gi2/Gi1
  overload`), con ACL que excluye el tráfico `10.1↔10.2` del NAT.
- **Extensión no planificada, incorporada por requisito del "cliente"**:
  filtro de seguridad ACL `INTERSITE-FILTER-IN` aplicado `in` sobre
  `Tunnel0` en ambos bordes — únicamente permite tráfico
  Gestión-VALPO(VLAN 41)↔Gestión-STGO(VLAN 30); todo el resto de tráfico
  inter-sitio (incluyendo Oficina-VALPO→Gestión-STGO,
  Operaciones-STGO→Gestión-VALPO) queda denegado y logueado.
- `default-information originate` + ruta estática por defecto
  (next-hop IP, no interfaz) en ambos bordes — necesaria para que el
  interior tuviera a dónde rutear tráfico hacia "internet" antes de que el
  NAT pudiera actuar (ver Problemas/aprendizajes).
- Verificado con tráfico real de PCs: Gestión-VALPO→VIP Gestión-STGO 5/5
  (permit, 5 matches); Oficina-VALPO→VIP Gestión-STGO 5/5 timeout (deny, 5
  matches, logueado); PC STGO→IP pública de ISP-1 5/5 con traducción NAT
  real confirmada en `show ip nat translations` (5 entradas ICMP).

**Punto 6 — Manipulación de atributos BGP:**
- Decisión: ISP-1 primario en ambas sedes, ISP-2 de respaldo.
- Route-maps por borde: `LOCALPREF-ISP1-IN` (localpref 200) y
  `LOCALPREF-ISP2-IN` (localpref 100) aplicados `in` a cada vecino;
  `PREPEND-ISP2-OUT` (`set as-path prepend 65010 65010`) aplicado `out`
  hacia ISP-2; `PASS-OUT` aplicado `out` hacia ISP-1 para no alterar ese
  anuncio.
- Verificado: `show ip bgp` confirma el camino vía ISP-1 como `best` con
  localpref 200 en ambos bordes; el camino vía ISP-2 muestra el AS-path
  alargado (`65010 65010` duplicado). `traceroute` desde BR-STGO hacia el
  loopback de BR-VALPO confirma 2 saltos, ambos por ISP-1 — el tráfico
  real sigue la política configurada, no solo la tabla.

## Decisiones y por qué

### Decisión #1 — sustrato de "internet": 2 ISP compartidos, doble rol
**Tensión.** El diseño original definía 4 AS, uno por rol de ISP (A/B/C/D). Para que el túnel VPN suba, las patas públicas de ambos sitios deben verse — había que decidir cuánta "internet" simular en EVE-NG.

**Opciones.** (A) un solo router/AS — colapsa el multihoming a "dos cables al mismo AS", sin diversidad de camino real; (B) 4 routers/4 AS con tránsito entre ellos — funcionalmente completo pero agrega superficie de lab sin capacidad nueva sobre C; (C) 2 routers/2 AS, cada uno con doble rol (ISP-1 hace de ISP-A/ISP-C, ISP-2 de ISP-B/ISP-D), peereados entre sí.

**Decisión: C**, bajo el filtro *función > seguridad > simplicidad*. En función, B y C empatan: ambas dan multihoming genuino con diversidad de AS real. En seguridad, B aporta aislamiento de fallos **entre sitios** que C no tiene, pero C conserva intacta la resiliencia **per-sitio**, que es la que el diseño necesita. En simplicidad, C gana por lejos. Y **C refleja mejor la realidad chilena:** Santiago concentra la oferta de ISP, y una sede regional frecuentemente reusa los mismos upstreams que la matriz — no son ISP disjuntos por sitio.

### Decisión #2 — anclaje del túnel VPN: loopback + BGP, no IP física
**Tensión.** El túnel necesita una IP de origen/destino fija. Si nace de una pata física de un solo ISP, el multihoming **no protege la VPN**: cae el túnel si ese ISP específico falla, aunque el otro esté arriba.

**Opciones.** (A) IP física de una interfaz de ISP — simple, pero contradice el propio multihoming del bloque; (B) loopback público anunciado por BGP a ambos ISP — el túnel sobrevive la caída de cualquiera de los dos.

**Decisión: B.** La simplicidad pierde contra la función: A rompe exactamente el requisito que el resto del bloque defiende. Con B, `tunnel protection ipsec profile` se usa en vez de crypto map, coherente con el origen dinámico vía loopback. Si cae un ISP, BGP reconverge y el túnel sigue sin intervención manual.

**Caveat documentado.** En producción real, un ISP filtraría cualquier prefijo más específico que /24: se anunciaría un /24 propio (PI space) con el loopback adentro, no un /32 aislado. En el lab se usa el /32 directo porque simplifica sin perder el concepto.

### Decisión #3 — qué se anuncia por eBGP: solo el loopback
**Tensión.** `network` en BGP no descubre nada, solo publica lo que uno decide. Candidatos: las redes internas (10.1/10.2), el loopback público, o ambos.

Anunciar redes internas se descarta de plano: son RFC 1918, ningún ISP real las aceptaría (filtro bogon), y contradice la premisa del NAT.

**Decisión:** anunciar únicamente el loopback público. Es lo mínimo que el diseño necesita (Decisión #2) y nada más, manteniendo la tabla BGP de "internet" limpia y defendible.

### Decisión #4 — `allowas-in` vs separar AustralPay en dos AS
**Problema, descubierto en verificación.** `show ip bgp summary` mostraba `PfxRcd: 0` en ambos bordes: el anuncio de un sitio nunca llegaba al otro. Causa: BGP descarta cualquier ruta que contenga el propio AS en el path (prevención de loop) — y como STGO y VALPO comparten el AS 65010 **y** los mismos dos ISP de tránsito, el anuncio de un sitio vuelve al otro con su propio AS ya presente en el path.

**Opciones.** (A) `allowas-in 1` en los 4 vecinos de los bordes — acepta explícitamente el propio AS una vez en el path; (B) separar `BR-VALPO` a un AS distinto (ej. 65011) — también resuelve el loop, pero convierte la relación `BR-STGO`↔`BR-VALPO` de iBGP a eBGP, contradiciendo la decisión rectora del Bloque 1 (AS 65010 único).

**Decisión: A.** B funciona técnicamente pero reabre una decisión de diseño ya tomada y defendida —AS único = una sola empresa, no dos entidades— solo para resolver un problema puntual que 4 líneas de `allowas-in` cierran sin tocar nada más.

### Decisión — filtro de seguridad inter-sitio (extensión no planificada)
**Requisito** (planteado como ejercicio del "cliente"): Gestión-VALPO debe poder llegar a Gestión-STGO; Oficina-VALPO **no** debe llegar a Gestión-STGO; Operaciones-STGO **no** debe llegar a Gestión-VALPO. Es decir, únicamente Gestión↔Gestión cruza entre sitios.

**Dónde aplicar el filtro.** (A) ACL en el `Tunnel0` de cada borde — todo el tráfico inter-sitio converge ahí, es el único camino posible entre área 1 y área 2; (B) ACL en cada SVI de distribución (4 puntos) — la defensa en profundidad solo tendría sentido si existiera un camino alternativo que evite el túnel, y no existe.

**Decisión: A.** Mismo principio que la VLAN 99 y `passive-interface default`: filtrar en el punto de tránsito único, no duplicar sin necesidad.

**Gotcha crítico.** El primer `permit` de la ACL debe cubrir explícitamente OSPF (`permit ospf any any`): de lo contrario el `deny` implícito final tumba los propios hellos de área 0 y se cae la adyacencia `BR-STGO`↔`BR-VALPO`. Verificado que la adyacencia se mantuvo `FULL` tras aplicar la ACL.

**Distinción de mecanismo.** Esto **no** reemplaza ni se superpone con el cifrado de la VPN — la confidencialidad ya la da IPSec para todo el tráfico del túnel. Esta ACL es de **autorización** (quién puede hablar con quién), no de protección criptográfica. En producción, este control normalmente vive en un firewall stateful dedicado (o ZPF); la versión con ACL es la simplificación adecuada a la escala del lab. Se reservaría un túnel o crypto adicional solo si un requisito de cumplimiento exigiera parámetros criptográficos distintos para ese tráfico — no es el caso.

### Decisión #5 — modo IPSec (tunnel)
Ya detallada en la sección de teoría. Resumen de decisión: tunnel mode por ser el estándar más reconocible y por no atarse a que los endpoints IPSec coincidan exactamente con el paquete — **no** por ser "más seguro" (la protección criptográfica del payload es idéntica en ambos modos).

### Decisión #6 — ISP primario (ISP-1 en ambas sedes)
Decisión de negocio, no técnica: en el lab ambos ISP son idénticos, así que la elección busca simetría y coherencia con la ruta estática de NAT, que ya apuntaba a ISP-1 en ambos bordes desde el Punto 5. LocalPref 200 (ISP-1) vs 100 (ISP-2) controla la salida; AS-path prepend ×2 hacia ISP-2 influye la entrada. Juntas, ambas direcciones prefieren ISP-1 en condiciones normales, con failover automático a ISP-2 si ISP-1 cae.

## Evidencia capturada
- Sustrato de internet: `show ip interface brief` en los 4 equipos +
  batería de pings directos entre los 5 enlaces (ISP-1↔ISP-2,
  ISP-1↔bordes, bordes↔ISP-1/ISP-2).
- `10-bgp-summary.png` — `show ip bgp summary` en los 4 routers BGP,
  sesiones Established; captura adicional mostrando `PfxRcd: 0→2` tras
  aplicar `allowas-in`.
- `show ip bgp <prefijo>` con el detalle de los 2 caminos disponibles
  (vía ISP-1 y vía ISP-2) hacia cada loopback remoto.
- `13-vpn-up.png` — `show crypto isakmp sa` (QM_IDLE ACTIVE) +
  `show crypto ipsec sa` (encaps/decaps 20/20 simétrico) en ambos bordes.
- `show interfaces tunnel0` — up/up, tunnel protection activo, en ambos
  extremos.
- `07-ospf-neighbors.png` (extendido) — `show ip ospf neighbor` en los 6
  equipos del borde/distribución: 6 adyacencias `FULL/-`, áreas 0/1/2
  confirmadas.
- `12-nat-translations.png` — `show ip nat translations` con 5 entradas
  ICMP reales (PC STGO → IP pública ISP-1).
- `show ip access-lists NAT-EXCLUDE-VPN` y
  `show ip access-lists INTERSITE-FILTER-IN` con contadores de match
  reales tras las 3 pruebas de tráfico (permit ospf, permit
  Gestión↔Gestión, deny Oficina→Gestión con log).
- `show ip bgp` post-atributos — localpref 200/100 visible, AS-path
  prepend visible en el camino hacia ISP-2.
- `traceroute` desde BR-STGO hacia el loopback de BR-VALPO confirmando
  2 saltos, ambos vía ISP-1.

## Problemas / aprendizajes
- **Incidente de pérdida de configuración.** `BR-STGO` y `BR-VALPO` perdieron toda la config de VPN/OSPF (y `BR-VALPO` también el BGP completo) por no haber hecho `write memory` tras los Puntos 2/3. Se reconstruyó comparando `show running-config` de los 6 equipos — `DLS1/DLS2/MLS1/MLS2` sí sobrevivieron porque sus bloques anteriores sí se habían guardado. **Regla adoptada para el resto del proyecto: `end` + `write memory` inmediatamente después de verificar cada punto, antes de avanzar.** (Esta regla es la que hizo que, cuando el laboratorio se corrompió de nuevo antes del cierre del Bloque 8, los últimos `show running-config` capturados en los logs de consola bastaran para reconstruir el estado.)
- **Auto-ping confundido con prueba de vecino.** Al reconstruir, un primer intento de verificación confundió un `ping` a la propia IP del router con una prueba real: el auto-ping siempre da 100%/1 ms porque el router se responde localmente. Se verifica siempre contra la IP del **otro** extremo del enlace.
- **`ip route 0.0.0.0 0.0.0.0 <interfaz>` en un enlace Ethernet** (multiacceso, no serial p2p) genera el warning *"Default route without gateway... may impact performance"*: en un segmento multiacceso el router necesita el next-hop IP para no depender de un lookup recursivo por cada destino distinto. Corregido usando `ip route 0.0.0.0 0.0.0.0 <IP-del-ISP>`.
- **La ruta estática de respaldo que este bloque configuró se retiró en el Bloque 8.** El primer intento de generar una traducción NAT real falló con *"Destination host unreachable"* devuelto por la propia SVI de distribución: sin `default-information originate` ni una default real en el borde, el interior no tenía ruta hacia "internet" y el paquete ni llegaba al NAT. Se corrigió agregando la default en OSPF **más una ruta estática de respaldo** hacia ISP-1 en los bordes — pieza que el propio diseño (sección 5.3) ya anticipaba. **El Bloque 8 comprobó que esa estática al primario era justamente lo que rompía la resiliencia del multihoming, y la reemplazó por una default aprendida de ambos ISP vía BGP.** El aprendizaje del B5 sigue siendo válido —el interior necesita una default para que el NAT actúe—; lo que cambió es de dónde sale esa default.
- **Asimetría heredada del Bloque 3.** `DLS1-STGO`/`DLS2-STGO` tenían pendiente aplicar `ip ospf network point-to-point` en `Gi1/0` —el propio informe del Bloque 3 lo dejó anotado como "aplicar cuando BR-STGO se integre en Bloque 5"—. Sin eso, la adyacencia subía como `FULL/BDR` (con elección DR/BDR) en vez de `FULL/-`: funcionaba, pero no coincidía con el criterio ya aplicado en VALPO desde el Bloque 4. Corregido para simetría completa.
