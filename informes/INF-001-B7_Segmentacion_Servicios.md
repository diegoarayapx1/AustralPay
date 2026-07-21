# Informe de Bloque — Redes AustralPay · Bloque 7: Segmentación y servicios
Fecha: 2026-07-15

> ⚠️ **AustralPay y la consultora DiegoAraya son entidades FICTICIAS, creadas exclusivamente con fines de laboratorio y portafolio. Ningún dato, dirección, dispositivo o infraestructura descrito aquí corresponde a una red real.**

---

## Objetivo del bloque

Ejecutar el diseño que el Bloque 6 cerró en papel: las tres VLANs nuevas (15 Servidores-DB, 50/55 Invitados) con su stack completo, la política intra-sitio de lista blanca, DHCP split-scope y el endurecimiento de puertos de acceso. Sin teoría nueva de protocolos — el B6 la dejó escrita. La "teoría" de este bloque es distinta: es **método de verificación**, y salió de los errores que la ejecución fue destapando.

---

## Teoría cubierta

La teoría de este bloque no es de protocolos, sino de **cómo se lee un equipo sin engañarse**. Cada punto de abajo nació de un diagnóstico que se persiguió en falso hasta entender por qué la herramienta mentía.

### El fallback del parser de IOS: `!` no es `exit`
IOS cae al nivel de configuración padre cuando un comando **no hace match** en el nivel actual. Por eso pegar comandos globales desde `config-if` funciona *casi* siempre.

**Dónde falla:** cuando el comando **comparte prefijo** con uno del nivel actual. `ip dhcp snooping` existe a nivel interfaz (`trust`, `limit`, `information`) → el parser lo resuelve ahí y **no cae** a global. `ip dhcp snooping vlan 20,50` reventó con `% Invalid input`.

**Por qué importa:** el caso ruidoso (`% Invalid input`) es el benigno. El peligroso es el gemelo silencioso: un comando que existe en **ambos** niveles con semántica distinta se aplica al nivel equivocado **sin error**. `ip dhcp snooping information option` es exactamente eso — global significa *"inserta Option 82"*; en interfaz, `... allow-untrusted` significa *"acepta Option 82 de puertos no confiables"*. Mismo texto, sentidos opuestos.

**Regla:** `exit` explícito antes de todo comando global. `!` en IOS es un **comentario**, no un separador de modo.

### Un filtro vacío no dice "no pasó"; dice "no matcheó tu cadena"
Este bloque acumuló **cuatro** falsos negativos por `| include`, todos con la misma raíz:

| Comando | Qué escondió | Por qué |
|---|---|---|
| `show ip route ... \| include 10.1.15.0` | El 2º next-hop del ECMP | Va en una **línea de continuación** sin el prefijo |
| `show logging \| include IPACCESSLOGP` | El evento de ACL | ICMP emite `IPACCESSLOGDP`, no `...LOGP` |
| `show ip dhcp snooping \| include operational` | La lista de VLANs | Está en la **línea siguiente** al encabezado |
| `show interfaces trunk` (sin filtro) | Una troncal existente | **No lista troncales caídas** |

**Las cuatro veces el vacío pareció un problema y ninguna lo era.** Dos derivaron en diagnósticos erróneos que se persiguieron varios minutos. **Regla:** en outputs multilínea, `| section` o sin filtro; `| include` solo cuando el dato vive en la **misma línea** que el ancla, y filtrando por la subcadena **más corta que discrimine** (`| include SEC-6` captura las cuatro variantes de `IPACCESSLOG*`). Es el mismo tropiezo que ya había aparecido en el Bloque 4 (`include Gi1/0` sin resultado) y que reaparecería en el Bloque 8 (`include ip domain-lookup`): un filtro que no puede encontrar lo que busca reporta ausente algo que está.

### `show run` no evidencia política cuando la política es el default
IOS **no escribe defaults** en `running-config`. Consecuencia contraintuitiva: **el puerto más restrictivo del diseño es el que se ve más vacío.**

| | `Gi1/0` STGO (servidor) | `Gi1/0` VALPO (estación) |
|---|---|---|
| `maximum` | *(invisible)* — `1` es default | `maximum 2` ✅ |
| `violation` | *(invisible)* — `shutdown` es default | `violation restrict` ✅ |

Mismo mecanismo en la consola: `exec-timeout 10 0` no aparece en `show run` porque 10 min es el default. La prueba de que tomó fue que DLS1/DLS2 tenían `exec-timeout 0 0` explícito (no-default, por eso se escribía) y el comando lo hizo desaparecer — **la desaparición *es* el cambio.**

**Regla:** la evidencia de port-security es `show port-security`; la de timeouts es `show line`. `show run` miente por omisión. Este principio es el que el Bloque 8 aplicaría a escala: una promesa de "solo SSH" no se verifica mirando lo que está configurado, sino lo que está escuchando.

### El ping no prueba el track: prueba resiliencia, no el mecanismo
Al tirar `Gi1/0` de DLS1, el ping desde S15A **sobrevive con track o sin track**. Si DLS1 se quedara Activo con su uplink caído, igual rutearía: tiene adyacencia OSPF con DLS2 por la **VLAN 99** sobre el peer-link, aprende el default por ahí, y el tráfico sale `S15A → DLS1 → Vlan99 → DLS2 → BR-STGO`. Funciona — **haciendo hairpin**.

**En esta topología el track no evita un agujero negro, evita un camino subóptimo.** Lo que lo hace necesario: (a) mantener el tráfico directo por el switch que sí tiene uplink; (b) si además cayera el peer-link, sin track sí habría agujero negro. **La prueba del track es `show standby brief`, no el ping.** El ping prueba *resiliencia*; el output de HSRP prueba *qué mecanismo* la produjo. El Bloque 8 refinó la mitad (a) de este argumento: midió que, por STP, la trama cruza el peer-link igual — el track protege contra la segunda falla, no contra el hairpin.

### HSRP preemptea, DHCP no — y ahí está el argumento entero
Al restaurar `service dhcp` en DLS2, **S20A se quedó con la IP de DLS1.** No volvió. No hay `preempt` en DHCP: el cliente conserva su lease hasta que expire y le da igual que el servidor original haya vuelto. Contraste: HSRP **sí** preemptea — DLS1 recupera el rol activo apenas el track vuelve a `Up`.

Esa diferencia es el argumento del §4.6: HSRP protege un rol que debe estar vivo *en este instante* — cae y el tráfico muere ya. DHCP protege una asignación que **ya ocurrió** — cae y no se entera nadie que tenga lease vigente. Uno es estado en tiempo real; el otro un trámite que ya pasó.

### El `show` miente, el syslog no
Tras una violación de port-security en un puerto de servidor, y luego de recuperarlo con `shutdown`/`no shutdown`:

| | Durante el err-disable | Tras `shutdown`/`no shutdown` |
|---|---|---|
| `Port Status` | `Secure-shutdown` | `Secure-up` |
| `Last Source Address` | `0050.7966.6815:10` | `0000.0000.0000:0` |
| `Security Violation Count` | `1` | **`0`** |

**La recuperación borró toda la evidencia forense del `show`.** El estado de un `show` es *presente*: refleja la última acción, no la historia. El syslog es *histórico*. Justifica dos decisiones: (1) **`errdisable recovery` desactivado a propósito** — si el puerto se auto-recuperara a los 30 s, el contador volvería a 0 solo y el único rastro sería una línea de log que nadie leyó; auto-recuperar no arregla el problema, borra la alerta. (2) **Exportar logs a un colector** — el motivo entero de que exista un SIEM. Un buffer local es un testigo con amnesia.

### El `log` de una ACL es agregado, no uno por paquete
Los eventos salieron en dos líneas: `1 packet` y después `4 packets`. IOS loguea el **primer** paquete al instante y agrega el resto en una ventana de 5 minutos. **Es rate-limiting deliberado:** sin él, un escaneo inundaría el syslog y el `log` sería un vector de DoS contra tu propio colector. **Para un SOC importa:** un SIEM que cuente eventos en vez de leer el campo `N packets` **subcontará por diez**.

### La prioridad de HSRP es relativa al decremento, no absoluta
`priority 150` con `track ... decrement 10` deja al switch en **140** tras la caída — muy por encima del 100 del vecino. **El track no puede conmutar nada.** El tracking existe en la config, aparece en `show track` como suscrito, y no hace absolutamente nada: redundancia decorativa.

La convención del B3/B4 (**105/100 con decremento 10**) no era arbitraria: deja **exactamente** el margen para que un solo decremento cruce el umbral (105 − 10 = 95 < 100). El error acá fue copiar el patrón sin copiar el porqué: `150` "se ve más fuerte", pero **fuerza y capacidad de conmutar son cosas distintas**.

### `preempt delay minimum`: el argumento correcto no era el obvio
**Argumento erróneo (descartado):** *"cierra la ventana de 7 s donde DLS1 es gateway activo sin adyacencia OSPF"*. Falso: esa ventana **no es un agujero negro** (el peer-link da el default por VLAN 99), y el delay **extiende** ese hairpin de 7 s a 33 s. Medido con esa vara, el delay es peor que no tenerlo.

**Argumento correcto:** (1) **el reload.** Si DLS1 *reinicia*, no tiene default por ninguna vía — la tabla de ruteo nace vacía. Sus SVI levantan, HSRP preemptea en ~1 s, y DLS1 se vuelve gateway **sin una sola ruta**: agujero negro real de 30-40 s hasta que OSPF converge desde cero. (2) **Damping de flaps**, evidenciado en el log: `Down->Up` → `Up->Down` → `Down->Up` en 2 segundos al recuperarse el enlace. El costo es una recuperación más lenta, que es un evento raro.

### HSRP en ACL: las dos mitades del mismo argumento, con contador
- `POL-V15-DB` línea 10 (`permit udp any host 224.0.0.102 eq 1985`): **570 matches**. Sin ese permit, esos hellos morirían en el `deny ip any any` y **ambos switches se creerían Activos** (split-brain).
- `POL-V50-INV` línea 50 (`permit ip any any`): **~500 matches** = los hellos de HSRP entrando por la puerta de atrás.

**Los dos contadores prueban la regla del §7.2 desde lados opuestos:** donde la lista termina en `permit ip any any`, el HSRP pasa de rebote; donde termina en `deny any`, necesita permit explícito o hay split-brain.

### Los dos códigos ICMP que separan la ACL del ruteo
- `type 3, code 13` (*administratively prohibited*) desde `10.1.50.3` = **una ACL me dijo que no**. Lo emite el switch mismo.
- `type 3, code 1` (*host unreachable*) desde `203.0.113.1` = **no tengo ruta**. Lo emite el ISP, fuera del AS.

**El ping a `9.9.9.9` fallando SÍ prueba el permit.** Si la ACL lo hubiera bloqueado, el error vendría de la SVI con code 13. Que venga del ISP prueba la cadena completa: ACL ✅ · ruta por defecto ✅ · NAT saliente ✅ · NAT del error ICMP de vuelta ✅ · ISP vivo ✅. Un ping exitoso a un host falso no probaría nada de eso.

### Sticky vs dinámica: el aging 0 y por qué la rotación de invitados funciona
`show port-security interface` reveló `Aging Time: 0 mins` → **las MAC seguras no expiran nunca.** Toda la rotación de invitados depende **enteramente** de que la MAC dinámica se libere al caer el link.

Experimento con la variable aislada — mismo `shutdown`/`no shutdown`, dos puertos:

| | `Gi1/0` (servidor, sticky) | `Gi2/0` (invitado, sin sticky) |
|---|---|---|
| MAC tras el ciclo | **sobrevive** (`Sticky: 1`) | **se libera** (`Total: 0`) |

Sticky vive en **config** (startup-config); dinámica vive solo en la **tabla de direcciones**. El power cycle del módulo Gi2 lo confirmó por accidente: las 4 sticky volvieron, las 2 dinámicas no. **Si la dinámica sobreviviera, con `maximum 1` y aging 0 el puerto quedaría clavado al primer invitado de por vida.** "Sin sticky" no es descuido: es el mecanismo correcto para un puerto donde rotan dispositivos. **Límite:** la rotación depende del link-down; un invitado con un hub detrás mantendría el link arriba y la MAC no se liberaría — `maximum 1` bloquea ese caso por otro lado, y en producción se agregaría `port-security aging type inactivity`.

### DHCP snooping: el default es la posición segura
DHCP no tiene autenticación: el cliente le cree **al primero que conteste**. Cualquiera que enchufe un servidor DHCP en un puerto de acceso reparte `default-router` apuntando a sí mismo → MITM sobre toda la VLAN. Ni siquiera hace falta un ataque: un router doméstico enchufado al revés hace exactamente esto.

**Cómo:** regla asimétrica — mensajes de **cliente** (DISCOVER/REQUEST) pasan por cualquier puerto; mensajes de **servidor** (OFFER/ACK) solo por puertos `trust`. Los uplinks van trust; el acceso queda untrusted **por defecto**: si te olvidas de un puerto, queda protegido, no expuesto.

**La trampa del Option 82:** con snooping activo, IOS inserta Option 82 por defecto en puertos untrusted. Pero un ALS es L2 —**bridgea, no relayea**—, así que el `giaddr` queda en `0.0.0.0`. El servidor recibe Option 82 presente + `giaddr=0` = inválido según RFC 3046 → **lo descarta en silencio**. DHCP muerto, cero errores en el ALS, cero errores en el DLS. El trust se configura en la Port-channel y se aplica en los puertos físicos.

---

## Qué se hizo

### P0 — Preparación de lab
- `write memory` de confirmación en ambos ALS antes de tocar topología.
- Módulo Gi2 agregado a `ALS1-STGO` (único apagado planificado del bloque). VALPO no lo necesitó: 4 puertos para 4 hosts.
- VLANs 40/41/999 nombradas en `ALS1-VALPO` (traían nombre default `VLAN0040`).
- 10 VPCS con `set pcname` y direccionamiento según rol (4 estáticos, 6 clientes DHCP).

### P1 — VLANs 15/50/55: stack completo
- VLAN + poda de troncales + root RPVST+ + SVI + HSRP con EOT + `network` de OSPF, en las tres VLANs.
- Reasignación de los 6 puertos de acceso de `ALS1-STGO` y los 4 de `ALS1-VALPO`, con `nonegotiate`, PortFast y BPDUGuard.
- **RPVST+ normalizado:** `DLS1-STGO`, `DLS2-STGO` y `ALS1-VALPO` corrían PVST clásico. Los 6 switches quedaron en `rapid-pvst`.
- **HSRP normalizado en las 8 VLANs:** v2, grupo = VLAN, 105/100, `track 1 decrement 10`, `preempt delay minimum 30`.
- Renumeración de grupos en VALPO: `1`→`40`, `2`→`41`.
- Failover verificado: track → HSRP en 1.5 s, V10/V15/V30 conmutan a DLS2.
- ECMP confirmado: las 3 VLANs nuevas con 2 next-hops en ambos bordes.
- ACL de NAT verificada (`/16`): V15/V50/V55 cubiertas sin tocar los bordes.

### P2 — DHCP split-scope
- 4 pools (V20/V40 lease 7 días · V50/V55 lease 2 horas), `dns-server 9.9.9.9`, `default-router` = VIP de HSRP.
- Exclusiones globales: DLS1/MLS1 sirven `.11`–`.132`, DLS2/MLS2 sirven `.133`–`.254`.
- Redundancia probada: `no service dhcp` en el servidor que ganó → el cliente toma IP de la otra mitad.

### P3 — Endurecimiento de puertos
- Port-security diferenciado en 10 puertos: servidores (sticky/1/**shutdown**), usuarios (sticky/2/restrict), invitados (**sin sticky**/1/restrict).
- `switchport protected` en los 4 puertos de invitados (2 por sede).
- Violación provocada en puerto de servidor con nodo `Intruso` → `err-disable` + `Last Source Address` registrado + recuperación manual.
- DHCP snooping en ambos ALS: trust en uplinks, `no ip dhcp snooping information option`, VLANs 20/50 y 40/55. 3+3 bindings poblados.

### P4 — Política intra-sitio
- **7 ACL** (6 `in` + 1 `out`), cada una en **ambos** switches del par.
- STGO: `POL-V10-APP`, `POL-V15-DB`, `POL-V15-DB-ACCESO` (out), `POL-V20-OPER`, `POL-V50-INV`.
- VALPO: `POL-V40-OFICINA`, `POL-V55-INV`.
- `log` en el `deny` final de `POL-V15-DB` (in) y en `deny ip any 10.1.15.0/24` de `POL-V20-OPER`/`POL-V50-INV`.

### P5 — Consola (pendiente del B6)
- `login local` + `exec-timeout 10 0` + `logging synchronous` en los **8 equipos**.
- Hostnames de VALPO corregidos: `MLS1`→`MLS1-VALPO`, `MLS2`→`MLS2-VALPO`. SSH verificado tras el renombre (la llave RSA sobrevivió).

---

## Decisiones y por qué

### 1. Dos invitados por sede (Opción A) en vez de dos solo en STGO
Se eligió A porque `switchport protected` se evidencia con **ping invitado↔invitado fallando dentro del mismo switch** — con un solo invitado por sede, esa captura no existe. Un control sin captura de deny es config, no evidencia. Costo real: cero (VALPO cabía en su módulo existente). El par permit/deny quedó en **ambas** sedes.

### 2. `9.9.9.9` (Quad9) en vez de `1.1.1.1` o `8.8.8.8`
Los tres cuestan lo mismo (un campo). Quad9 **rechaza dominios maliciosos conocidos** vía threat intel — corta C2, phishing y descarga de payloads antes de que salga un paquete. Es el control más barato con mejor retorno de la pila y no requiere firewall, proxy ni agente. Se descartó el DNS diferenciado (interno para corporativas, público para invitados) porque se cae al primer contraargumento: *"¿por qué filtras los dominios maliciosos de los invitados pero no los de tu VLAN de operaciones?"* — le pondría el mejor control a la red menos crítica.

**Honestidad:** un invitado con DNS-over-HTTPS en el navegador evade el `dns-server` del DHCP por completo. Quad9 filtra al que no se esfuerza. El control real sería bloquear DoH en el borde → §11 del diseño.

### 3. Mantener las 4 pools de DHCP (se rechazó "DHCP solo para invitados")
§4.6 ya defiende el criterio: *infraestructura estática, estaciones e invitados por DHCP*. V20/V40 **son** las VLANs de estaciones. Dejarlas estáticas invertiría el criterio por una limitación del lab (un PC por VLAN) disfrazada de decisión de diseño. Además mata el argumento del **lease diferenciado**, que necesita los dos tipos para tener sentido. Y reducir costaba **más** trabajo que mantener (reescribir §4.6 + changelog + ajustar §7.2 + defender la desviación).

### 4. `preempt delay minimum 30` en los 16 grupos (Opción A)
El argumento válido es el **reload** (tabla de ruteo vacía + preempt en 1 s = agujero negro de 30-40 s), no la ventana de convergencia de OSPF. Aplicado también en grupos donde el switch es Standby y el delay nunca dispara — por **simetría de config**, que es el criterio de auditoría del bloque. Config asimétrica = drift esperando aparecer.

### 5. V30 y V41 (Gestión) SIN ACL (Opción B)
Su fila de la matriz es todo-permitido. La lista sería literalmente `permit ip any any`. **La ausencia de ACL *es* esa política.** Aplicar un `permit ip any any` explícito sería ceremonia que consume CPU por cero enforcement. Se documenta la decisión para que la ausencia no se lea como omisión. (Esta es la decisión que redujo de 9 a 7 el número de listas de política, y cuya no-recontabilización el Bloque 8 registró como su divergencia #11.)

### 6. `log` en las dos direcciones del acceso a la DB (Opción C)
- `POL-V15-DB` (in), deny final: captura **la DB iniciando algo**. Volumen normal esperado: **cero**. Una base de datos que inicia conexiones es indicador clásico de exfiltración o C2.
- `deny ip any 10.1.15.0/24 log` en `POL-V20-OPER`/`POL-V50-INV`, **antes** del deny genérico: captura **quién intentó llegar a la DB**, con IP de origen.
- Son las dos mitades de la misma pregunta, y las dos tienen volumen esperado cero — que es el criterio que hace un log valioso en vez de ruido.
- El `deny any log` de `POL-V15-DB-ACCESO` (out) **nunca puede dispararse hoy**: todo lo que podría hacerlo ya muere en su propio ingreso. Ese es su trabajo — es un **cable trampa para el futuro**: si alguien crea una VLAN sin ACL, esa línea grita.

### 7. `ping 192.0.2.1` como check de internet (Opción B), sin agregar un VPCS
§4.4 del diseño ya define `192.0.2.0/30` (peering ISP-1↔ISP-2) como *"el sustrato de internet"*. Agregar un VPCS "Internet" sería un error: tendría que llevar IP RFC 5737 igual, o sea seguiría siendo documentación — se gana un nodo y no se gana realismo. Se mantiene el ping a `9.9.9.9` como evidencia complementaria (ver los códigos ICMP en la teoría).

### 8. Corregir hostnames de VALPO al diseño (Opción A), no el diseño a la config
`MLS1`/`MLS2` eran los únicos 2 de 8 equipos sin sufijo de sede. Cambiar el documento habría consagrado el error. El criterio del bloque fue *"si el maestro dice X, la config dice X"* — se aplicó con RPVST+, HSRPv2 y grupo=VLAN. Costo: la evidencia previa al P5 muestra `MLS1#`/`MLS2#`.

### 9. Documentar el bug de VPCS en vez de montar la captura limpia (Opción A)
Ver "Problemas / aprendizajes". Se rechazó apagar un servidor DHCP para producir tablas prolijas: sería la foto de un estado que la red **no alcanza operando normal**, y eso contradice la regla de honestidad del proyecto.

---

## Evidencia capturada

| Archivo | Qué muestra |
|---|---|
| `01-rpvst-drift.png` | `show spanning-tree summary` en los 6 switches — 3 en `pvst`, 3 en `rapid-pvst` |
| `02-hsrp-init-svi-shutdown.png` | `show standby vlan15` → `State is Init (interface down)` + `administratively down` |
| `03-hsrp-normalizado.png` | `show standby brief` en los 4 L3 — 8 VLANs, v2, grupo=VLAN, 105/100 |
| `04-mac-virtual-v2.png` | `0000.0c9f.f028` en V40 → `0x28` = 40. La MAC virtual codifica la VLAN |
| `05-failover-track.png` | Log de `TRACK Up->Down` → `HSRP Active->Speak` en 1.5s + prioridades en 95 |
| `06-preempt-delay-experimento.png` | V10/V30 Active en +0.3s / +1.0s vs **V15 en +33.1s** — grupo de control en el mismo log |
| `07-ecmp-2-nexthops.png` | `show ip route 10.1.15.0` → 2 Routing Descriptor Blocks |
| `08-dora-dos-servidores.png` | `dhcp -d` de S20A — dos OFFER de rangos disjuntos (split-scope en vivo) |
| `09-dora-un-servidor.png` | `dhcp -d` de S20A con DLS2 apagado — DORA completo con **ACK**, cliente calza con binding |
| `10-lease-diferenciado.png` | `604800` (7d) en V40A vs `7200` (2h) en V55A/B |
| `11-split-scope-failover.png` | `no service dhcp` en DLS2 → S20A toma `.13` de DLS1 (mitad baja) |
| `12-port-security-diferenciado.png` | `show port-security address` — `SecureSticky` y `SecureDynamic` en la misma tabla |
| `13-violacion-err-disable.png` | `%PSECURE_VIOLATION ... MAC 0050.7966.6815` + `Last Source Address:Vlan 0050.7966.6815:10` |
| `14-sticky-vs-dinamica.png` | Mismo `shutdown`/`no shutdown` — sticky sobrevive, dinámica se libera |
| `15-protected-ports-stgo.png` | `S50A → S50B` **not reachable** + `S50A → gateway` 5/5 |
| `16-protected-ports-valpo.png` | `V55A → V55B` **not reachable** + `V55A → gateway` 5/5 |
| `17-snooping-operational.png` | `operational on VLANs: 20,50` + `Insertion of option 82 is disabled` |
| `18-snooping-bindings.png` | 3+3 bindings con MAC ↔ IP ↔ VLAN ↔ **puerto físico** |
| `19-acl-hsrp-matches.png` | `POL-V15-DB` línea 10 con **570 matches** — el permit que evita el split-brain |
| `20-acl-codigos-icmp.png` | code 13 desde `10.1.50.3` (ACL) vs code 1 desde `203.0.113.1` (ISP) |
| `21-acl-doble-candado.png` | `POL-V15-DB-ACCESO` — 5+5 matches, deny en 0 |
| `22-acl-logs.png` | `%SEC-6-IPACCESSLOGDP: 10.1.20.13 -> 10.1.15.10` y `10.1.15.10 -> 10.1.10.10` |
| `23-snooping-aba.png` | A-B-A: snooping ON → 0 matches · OFF → 10 · ON → 10 (sin subir) + 15 drops |
| `24-consola-timeout.png` | `show line con 0` → `Idle EXEC: 00:10:00` |
| `25-ssh-post-rename.png` | SSH a `MLS1-VALPO` y `MLS2-VALPO` tras el renombre |

---

## Problemas / aprendizajes

### Drift encontrado: lo que el B2 declaró vs. lo que había
El informe del B2 decía *"RPVST+ en los tres switches"* y su checklist tenía `[x] RPVST+ con root planificado`. La realidad: **el root sí se hizo** (las prioridades estaban), **el modo solo llegó a ALS1-STGO**. DLS1, DLS2 y ALS1-VALPO corrían PVST clásico.

**Por qué nunca se notó:** PVST+ y RPVST+ interoperan — RSTP hace fallback y la red converge igual. Nada se ve roto. Lo que se pierde es la convergencia sub-segundo: timers de 30-50 s, justo lo que el B2 documentó como *"obsoleto"* y usó como argumento contra 802.1D.

**Otros drifts del mismo tipo:** V10 sin un solo puerto asignado en ALS1-STGO; `Gi1/1` y `Gi1/2` ambos en VLAN 20 cuando el informe decía VLAN 10 y 20; hostnames `MLS1`/`MLS2` sin sufijo de sede. **Aprendizaje:** un informe declara la intención; solo un `show` declara el estado. Este bloque encontró 4 drifts que llevaban entre 2 y 5 bloques sin que ningún ping los delatara — el mismo fenómeno que el Bloque 8 formalizaría como sus once divergencias.

### El bug de VPCS: seis coincidencias no son determinismo
**Síntoma:** en 6 de 6 clientes, la IP que el cliente creía tener estaba en estado `Selecting` en un servidor, mientras el lease `Active` (con otra IP) vivía en el otro.

**Mecanismo, según el `dhcp -d`:** VPCS pide `.133` a DLS2 → DLS2 ACKea → `.133` queda **Active**. Después llega la OFFER tardía de DLS1 con `.12` → **VPCS la adopta sin mandar REQUEST** → `.12` se queda en `Selecting` para siempre. **Es un bug del cliente, no de la red.** Un cliente real ignora ofertas posteriores al REQUEST (RFC 2131 §4.4.1).

**La corrección que importa:** primero se enunció como *"VPCS siempre adopta la última OFFER"* — **falso**, y un contraejemplo lo tumba. Una tanda posterior dio **cero descalce en los 6**. El enunciado que la evidencia sostiene: *"VPCS no valida que una REPLY corresponda a su REQUEST pendiente; el descalce depende del orden de llegada"*. **Es una carrera, no una regla.** Seis coincidencias eran la misma carrera cayendo del mismo lado seis veces.

**Por qué DLS1/MLS1 ganaron la segunda tanda:** ya tenían binding previo para esas MAC → ofrecieron el **remanente** del lease viejo sin asignar nada nuevo y **sin ping de verificación**. El otro switch tuvo que sacar IP del pool y pinguearla: ~1 s de retardo. `ip dhcp ping packets` decide quién gana la carrera. **Y hace el bug inofensivo:** los `Selecting` expiran en ~5 min y esas IPs vuelven al pool mientras el cliente las sigue usando. Sin el ping previo, IOS se las daría a otro → IP duplicada real. El `%DHCPD-4-PING_CONFLICT: server pinged 10.1.20.11` que salió **es ese mecanismo funcionando**, no un error.

### Contradicción en el propio diseño: `V20 → V30`
La matriz del §7.2 tenía `V20 → V30 ❌`. Se detectó porque `S30A → 10.1.20.13` daba **timeout** en vez de code 13: el paquete salía (V30 no tiene ACL) y llegaba, pero **la respuesta** moría en `POL-V20-OPER`.

Era la única celda sin su retorno en toda la red, y contradecía el texto de su propia sección: *"Gestión administra todo"*. Con `V20 → V30 ❌`, gestión **no** administra las estaciones — la VLAN que más se administra en cualquier red real. **Aprendizaje:** la matriz se transcribió fielmente a las ACL, **incluido su error**. VALPO llevaba su `echo-reply` a V41 y STGO no llevaba el suyo a V30. Se copió el documento en vez de leer la política. Un ping lo cazó — un diseño se valida ejecutándolo, no releyéndolo.

### El test que falló y terminó probando otra cosa (A-B-A)
**Objetivo:** validar la línea `permit udp any host 10.1.50.2 eq bootps`, que cubre la **renovación T1** (unicast a la SVI). VPCS no implementa T1 — `dhcp -r` rehace un DORA broadcast. **Idea:** la ACL no inspecciona payload; un UDP:67 al mismo destino ejercita el **mismo match**. Se mandó con `ping 10.1.50.2 -P 17 -p 67`. **Resultado:** 0 matches. El ICMP al mismo destino sí llegaba (code 13), así que el camino estaba sano → el problema era específico de UDP:67.

**Experimento A-B-A en `ALS1-STGO`:**

| Fase | Snooping | ACL línea 10 | `Packets Dropped` |
|---|---|---|---|
| **A** | ON | **0** | 15 |
| **B** | OFF (`no ip dhcp snooping vlan 50`) | **10** | 15 |
| **A'** | ON | **10** (sin subir) | 25 |

**Snooping se los comía en el ALS.** Los 15 drops = exactamente los 15 paquetes UDP:67 enviados durante la fase A. Dos contadores independientes, en dos equipos distintos, midiendo el mismo evento. **Corrección del mecanismo:** se atribuyó el drop a *"venir de un puerto untrusted"*. **`Packets Dropped From untrusted ports = 0`** lo desmiente. Los paquetes iban a puerto 67 = mensajes de *cliente*, que **sí se permiten** desde untrusted. Lo que los mató fue `Verification of hwaddr field is enabled` al parsear relleno de ping como DHCP y no encontrar un `chaddr` que calzara con la MAC de origen.

**Lo que quedó probado y lo que no:**

| Afirmación | Estado |
|---|---|
| La línea de renovación DHCP matchea el 5-tuple correcto | ✅ 10 matches en DLS1 (`.2`) y 10 en DLS2 (`.3`) |
| Snooping descarta DHCP **falsificado** desde un puerto de acceso | ✅ 15 drops, variable aislada |
| Snooping bloquea un **servidor rogue** (OFFER/ACK desde untrusted) | ❌ `From untrusted ports = 0` → §11 del diseño |

El test fallido terminó siendo la evidencia de otro control: se había fabricado un DHCP falso desde un puerto de acceso, que es justo lo que snooping existe para matar.

### La SVI nace en `shutdown` y el `Init` de HSRP no es lo que parece
Las tres SVI nuevas quedaron en `Init/unknown/unknown`. La hipótesis inicial fue timing (HSRP tarda unos hellos en converger) — razonable, pero errónea. El `show standby vlan15` dio la causa exacta: `administratively down` / `State is Init (interface down)`.

**En esta plataforma una SVI recién creada nace administrativamente abajo.** `Init` en HSRP no significa *"no encuentro al vecino"* — significa *"la interfaz no está arriba, no tengo dónde correr"*. Por eso `unknown/unknown`: no hubo un solo hello. El resto del grupo estaba perfecto (prioridad, track, MAC virtual v2): **la config era correcta; la interfaz estaba apagada.** `brief` confirma síntomas, el detalle da causas — tres `show standby brief` seguidos no habrían encontrado nunca la palabra `shutdown`.

### Una troncal caída es invisible en `show interfaces trunk`
`show interfaces trunk` en DLS1 mostró solo `Po1`, sin `Po2`. Se interpretó como un enlace de redundancia muerto desde el B2 — **falso**. `show interfaces trunk` **solo lista troncales operativas**; `ALS1-STGO` estaba reiniciando por el módulo Gi2, sus miembros abajo, el canal abajo, y por eso no aparecía. **El error de método fue peor que el técnico:** se comparó ese output con un `show etherchannel summary` de **antes del apagado** y se trató la diferencia como un hallazgo. Dos fotos de momentos distintos. **Regla:** para saber qué troncales *deberían* existir, `show interfaces status` o `show run` — no `show interfaces trunk`. Y al leer un output, preguntar **de cuándo es**.

### Errores de ejecución del bloque, en orden
Se listan porque el patrón —11 errores propios, cada uno cazado por su verificación— es el método del proyecto en miniatura, y es lo que hizo posible la matriz de 59 celdas del Bloque 8.

| # | Error | Cómo se detectó |
|---|---|---|
| 1 | Falsa alarma: "Po2 no está en trunk" | Comparar outputs de momentos distintos |
| 2 | SVI nuevas sin `no shutdown` | `show standby vlanX` → `interface down` |
| 3 | `standby version 2` solo en las VLANs nuevas | `show run` completo |
| 4 | `priority 150` → el track no podía conmutar | Aritmética: 150−10 = 140 > 100 |
| 5 | Se saltó la capa 2 de V55 en VALPO antes del HSRP | `V55B → gateway` not reachable |
| 6 | `!` usado como separador → 3 comandos globales al nivel interfaz | `% Invalid input` |
| 7 | Filtro `include IPACCESSLOGP` (era `...LOGDP`) | Log vacío con matches subiendo |
| 8 | Filtro `include operational` (VLANs en la línea siguiente) | Se reaplicó snooping 2 veces de más |
| 9 | Se transcribió la contradicción `V20→V30` del §7.2 a las ACL | `S30A → S20A` timeout |
| 10 | Predicción errónea del `log` en `POL-V15-DB-ACCESO` | El code 13 vino de la SVI de V20, no de V15 |
| 11 | Se afirmó que `S10A → S15A` probaba 3 líneas de ACL | `POL-V15-DB` línea 20 en 0 matches |

---

## Cierre del bloque: qué quedó probado, y qué se declaró como límite

Este bloque cerró el stack de segmentación y servicios completo: las 3 VLANs nuevas con HSRP/RPVST+/OSPF, el DHCP split-scope con lease diferenciado, el endurecimiento de los 10 puertos de acceso, las 7 ACL de política validadas celda por celda, la consola endurecida en los 8 equipos y los pendientes del Bloque 6 cerrados (`no ip routing` en ALS1-VALPO, hostnames, portfast/BPDUGuard, el módulo Gi2, los PCs).

**Cuatro afirmaciones quedaron declaradas como límite de evidencia** —no como deuda de trabajo—, y se consolidaron en la §11 del diseño maestro:

- **Renovación DHCP T1 real:** VPCS no la implementa. Se validó el 5-tuple correcto por un camino equivalente (UDP:67 on-link a ambas SVI, 10+10 matches), pero no el mensaje unicast T1 genuino.
- **Servidor DHCP rogue:** `Packets Dropped From untrusted ports = 0`. Ese camino del snooping no se ejercitó porque no hay con qué montar un servidor falso en el emulador. Sí se probó el descarte de DHCP falsificado desde un puerto de acceso.
- **Violación `restrict` en puertos de usuario:** exige un tercer dispositivo en el mismo puerto (`maximum 2`), que el lab no tenía. El mecanismo `restrict` sí quedó probado por otra vía.
- **Puerto de invitado aceptando un dispositivo nuevo:** descartado por costo (dos apagados de `ALS1-STGO`). El mecanismo —la MAC dinámica se libera al link-down— quedó probado con la variable aislada en P3.

Todo lo que quedó pendiente de trabajo real pasó al **Bloque 8**: la matriz de conectividad completa contra el diseño, el failover con ping continuo (HSRP e ISP), y la documentación de entrega con las configs sanitizadas.
