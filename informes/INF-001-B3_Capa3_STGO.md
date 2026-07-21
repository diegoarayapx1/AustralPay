# Informe de Bloque — Redes AustralPay · Bloque 3: Capa 3 STGO
Fecha: 2026-07-12

## Objetivo del bloque
Levantar la capa 3 del sitio matriz STGO: enrutamiento InterVLAN entre las
VLANs 10/20/30, gateway redundante con HSRP alineado al root de STP del
Bloque 2, adyacencia OSPF de área 1 entre los switches de distribución, y
preparación del enlace L3 hacia el borde (BR-STGO, a integrarse en el
Bloque 5).

## Teoría cubierta

### InterVLAN: ROAS vs switch multicapa
Las VLANs aíslan dominios de broadcast; algo con capacidad L3 tiene que rutear entre ellas.

- **ROAS (router-on-a-stick):** un solo enlace troncal con subinterfaces por VLAN. Todo el tráfico interVLAN comparte un cable → **cuello de botella y punto único de falla**. Sirve para redes chicas con switch solo-L2.
- **Switch multicapa (SVI + `ip routing`):** rutea en hardware vía CEF —FIB más adjacency table con el header L2 ya armado— a velocidad de cable, y habilita HSRP entre dos switches de distribución para dar gateway redundante, algo que ROAS no resuelve limpio.

**Se eligió el switch multicapa:** DLS1/DLS2 ya son una distribución colapsada, y es lo único que permite HSRP entre dos equipos.

### `ip routing` y autostate de las SVI
`ip routing` activa el proceso de reenvío L3 entre SVIs. Sin él, las SVIs sirven solo como gateway de gestión: no reenvían entre VLANs.

**Autostate:** una SVI solo pasa a `up/up` si existe al menos un puerto miembro de esa VLAN en estado STP **forwarding** — no basta con que esté conectado. Una troncal que permite la VLAN también cuenta como miembro, y por eso el peer-link `Po1` sostiene las SVIs arriba en ambos switches aunque no haya un solo host final conectado.

> **Gotcha de laboratorio:** una SVI en `down/down` con la IP bien puesta casi siempre es autostate, no un typo de configuración.

### Redundancia de gateway: HSRP, VRRP y GLBP

| | Origen | Balanceo | Cuándo se justifica |
|---|---|---|---|
| **HSRP** | propietario de Cisco | manual — se alterna el activo por VLAN | ambiente 100% Cisco |
| **VRRP** | estándar IETF | manual, mismo patrón que HSRP | cuando hay multi-vendor real |
| **GLBP** | propietario de Cisco | **automático** entre varios routers con un solo VIP (AVF/AVG) | cuando el reparto manual no alcanza |

**Se eligió HSRP.** El ambiente es 100% Cisco, así que VRRP no aporta la interoperabilidad que sería su única ventaja. GLBP agrega complejidad de operación —balanceo automático— para resolver algo que acá ya se resuelve alternando el activo por VLAN, alineado con el root de STP del Bloque 2.

### Cómo elige HSRP, y cómo NO
- **Desempate:** primero `priority` — gana el más alto. Si hay empate, gana **la IP configurada más alta** en esa interfaz para ese grupo. **No la MAC:** ese es el desempate del Bridge ID de STP, y son mecanismos distintos.
- **`track <objeto> decrement <N>`:** el decremento por defecto es 10, y con múltiples objetos trackeados los decrementos se acumulan. La priority nunca baja de 0.
- **El decremento por sí solo no conmuta nada.** Baja la priority local; es el peer con `preempt` quien reclama el rol, si su priority sin degradar queda por encima.
- **Aritmética que hay que hacer antes de configurar:** si la diferencia de priority es menor o igual al decrement, el resultado es un **empate**, no una victoria. La diferencia debe quedar por debajo del decrement para garantizar un cruce limpio — con 105/100 y decrement 10: `105 − 10 = 95`, cruza limpio por debajo de 100.

### Alineación del root de STP con el activo de HSRP
Si el root L2 de una VLAN y el activo L3 de esa misma VLAN están en switches distintos, el tráfico toma el camino L2 más corto hacia un switch que no procesa L3 para esa VLAN, y tiene que cruzar el peer-link para llegar al gateway real: **hairpinning**.

Alinear ambos en el mismo switch, por VLAN, hace que el camino L2 más corto y la salida L3 coincidan.

**`preempt` es lo que sostiene la alineación en el tiempo:** sin él, se restaura sola tras un failover y la vuelta del switch preferido nunca la recupera — la alineación queda como una foto del día 1.

### Enhanced Object Tracking
La sintaxis moderna de IOS **desacopla el objeto trackeado del protocolo que lo consume**: `track <N> interface X line-protocol` por un lado, `standby X track <N>` por otro. Un objeto, múltiples clientes posibles (HSRP, VRRP, GLBP, PBR), en vez de un mecanismo de tracking aislado dentro de cada protocolo como en la sintaxis vieja.

### OSPF multiárea: por qué, y cuándo un ABR es real
Un área plana obliga a que cada router tenga el mapa link-state completo de toda la red, y cualquier cambio dispara SPF en todos los routers. No escala.

En multiárea, cada área es su propio dominio link-state y el **ABR** resume rutas entre áreas vía LSA tipo 3, evitando que un cambio local dispare recálculo completo en las demás.

**Regla dura:** todo tráfico entre dos áreas no-backbone pasa obligatoriamente por el área 0. No existe el área 1 → área 2 directo.

**Por qué acá es real y no decorativo:** cada sitio es su propia área (1 en STGO, 2 en VALPO) y los bordes son ABR de verdad — una pata en el túnel GRE (área 0) y otra en el área del sitio. Un solo sitio no justificaría multiárea.

### La máquina de estados de vecindad OSPF
`Down → Init → 2-Way → ExStart → Exchange → Loading → Full`

- **2-Way:** hello bidireccional confirmado. En un segmento broadcast, acá se elige DR/BDR; en point-to-point pasa directo a ExStart.
- **ExStart / Exchange:** negociación maestro-esclavo e intercambio de DBD (resúmenes de LSAs, no el contenido completo). **Atascarse acá es sospecha de MTU mismatch.**
- **Loading:** LSR pidiendo el contenido completo de las LSAs faltantes.
- **Full:** bases de datos idénticas, adyacencia formada.

### Tipos de red OSPF, DR/BDR y LSA
- **Broadcast** (el default en SVI/Ethernet): asume que puede haber múltiples routers → elección de DR/BDR para evitar N×(N−1)/2 adyacencias.
- **Point-to-point:** asume exactamente 2 vecinos → sin elección de DR/BDR, va directo a Full. Es lo correcto para un enlace de tránsito de dos puntas como la VLAN 99: elegir DR/BDR ahí es overhead sin beneficio.
- **LSA tipo 1 (Router):** cada router describe sus enlaces; se inunda dentro del área. **Tipo 2 (Network):** lo genera el DR de un segmento broadcast — no se genera en enlaces p2p. **Tipo 3 (Summary):** lo genera el ABR para resumir rutas entre áreas.

### `passive-interface default`
El default de OSPF es que **toda** interfaz con `network` sea activa: manda hellos y puede formar vecinos. Eso expone las VLANs de usuario a que un host hable OSPF y forme vecindad no autorizada o inyecte rutas.

`passive-interface default` invierte la lógica: **todo pasivo salvo lo explícitamente activado.** Las SVIs de usuario se siguen anunciando —otros routers aprenden esas subredes— pero no aceptan vecinos ahí. Reduce la superficie de inyección de rutas a las interfaces que de verdad la necesitan.

## Qué se hizo
- `ip routing` habilitado + SVIs 10/20/30 en DLS1 (.2) y DLS2 (.3), /24 cada
  una, verificadas up/up y con conectividad cruzada.
- HSRP por VLAN: DLS1 activo (priority 105) en VLAN 10 y 30, DLS2 activo
  (priority 105) en VLAN 20 — alineado con el root de STP del Bloque 2.
  `preempt` en los tres grupos.
- VLAN 99 de tránsito (/30, `10.255.99.0/30`) creada sobre el peer-link Po1,
  dedicada exclusivamente a la adyacencia OSPF DLS1-DLS2.
- OSPF área 1 habilitado: `passive-interface default` global, excepción
  explícita solo en Vlan99 y Gi1/0; `ip ospf network point-to-point` en
  Vlan99; router-id fijado manualmente en ambos switches.
- Adyacencia OSPF DLS1-DLS2 verificada en estado `FULL/-` sobre Vlan99.
- Interfaz Gi1/0 (uplink a BR-STGO) preparada con IP y `network` OSPF en
  ambos switches — dual-homed (`10.255.1.0/30` en DLS1, `10.255.1.4/30` en
  DLS2) — sin adyacencia real todavía (BR-STGO se integra en Bloque 5).
- Enhanced Object Tracking: objeto `track 1` vigilando line-protocol de
  Gi1/0, referenciado por los tres grupos HSRP con `decrement 10`.

## Decisiones y por qué
- **VLAN de tránsito dedicada (99) para OSPF, en vez de vecinar sobre las SVIs 10/20/30.** Vecinar sobre las SVIs de usuario habría obligado a sacarlas de `passive-interface default` —exponiendo esas VLANs a hablar OSPF— y habría generado **tres adyacencias redundantes** entre el mismo par de switches. La VLAN 99 mantiene la higiene completa de passive-interface y deja **una sola adyacencia**, dedicada al plano de control.
- **Priority HSRP 105/100 (diferencia de 5) en vez de tocar el `decrement`.** Con el decrement por defecto de 10, `105 − 10 = 95` cruza limpio por debajo de 100. Evita configurar un valor de decrement no estándar que después haya que explicar.
- **`ip ospf network point-to-point` en la Vlan99.** Exactamente dos vecinos en ese segmento: DR/BDR sería overhead sin beneficio.
- **Enhanced Object Tracking en vez de `standby track <interfaz>` directo.** Es la sintaxis que la plataforma exige, y además es más flexible: el objeto es reusable por múltiples protocolos.
- **Dual-homing de BR-STGO a ambos DLS, con dos /30 separados.** Decisión ya registrada en el diseño antes de este bloque; ejecutada acá en `Gi1/0` para que **el tracking de HSRP de cada switch tenga un uplink propio que vigilar**, no uno compartido.

## Evidencia capturada
- `05-svi-routing.png` — `show ip interface brief | include Vlan` en DLS1 y
  DLS2 (6 SVIs up/up) + ping cruzado exitoso entre SVIs de ambos switches.
- `06-hsrp-active-standby.png` — `show standby brief` en DLS1 y DLS2,
  convergido: DLS1 activo en Vlan10/30, DLS2 activo en Vlan20.
- `07-ospf-neighbors.png` — `show ip ospf interface brief` (Vlan99 P2P,
  Vlan10/20/30 pasivas) + `show ip ospf neighbor` (FULL/- entre DLS1-DLS2)
  en ambos switches.
- `track/EOT` — `show track 1` confirmando el objeto y los tres grupos HSRP
  suscritos (`Tracked by`).

## Problemas / aprendizajes
- **El primer ping cruzado entre SVIs (DLS1→DLS2) salió 4/5.** El primer paquete se pierde en la resolución ARP inicial; no es una falla de ruteo. Es la firma normal de un primer ping hacia un vecino nuevo.
- **`show standby brief` mostró la VLAN 30 en estado `Speak`** transitoriamente en DLS2 justo después de aplicar la configuración. Es un estado normal de convergencia (hello 3 s / holdtime 10 s) y se resolvió solo a `Standby` en segundos.
- **HSRP no desempata por MAC.** Se confundió su mecanismo con el de STP (priority + MAC del Bridge ID). HSRP desempata por **priority y luego por la IP más alta**; la MAC no participa. Son dos protocolos con dos criterios distintos y es un cruce fácil de hacer.
- **La sintaxis `standby X track <interfaz>` no existe en esta versión de IOS.** El equipo exige Enhanced Object Tracking (`track <N> interface X line-protocol` + `standby X track <N>`). Confirmado en el lab tras varios intentos con la sintaxis vieja.
- **`Gi1/0` reporta `line-protocol Up` en `show track` incluso sin BR-STGO conectado.** Es una particularidad de EVE-NG con vIOS, no un error de configuración: la NIC virtual está contra un bridge que está arriba. Queda pendiente el comportamiento real una vez que BR-STGO se cablee en el Bloque 5.
- **Pendiente de confirmar:** `show ip protocols | include Router ID` en DLS1, para verificar que el router-id quedó exactamente en `10.1.1.1`. Una captura previa mostró `1.1.1.1` en el neighbor ID visto desde DLS2 — posible recorte de columna en una consola angosta.
