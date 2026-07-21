> ⚠️ **AustralPay y la consultora DiegoAraya son entidades FICTICIAS, creadas exclusivamente con fines de laboratorio y portafolio. Ningún dato, dispositivo o configuración descrito aquí corresponde a una red real.**

# Informe de Bloque — Redes AustralPay · Bloque 8: Verificación y entrega
Fecha: 2026-07-15/16

## Objetivo del bloque

Demostrar que la red hace lo que el diseño promete —cada flujo permitido *y* cada flujo bloqueado, en ambas direcciones— y que sobrevive las fallas para las que fue construida; luego empaquetarla como entregable de consultor.

Único bloque que **no agrega tecnología por diseño**. Lo que produce es **evidencia** y **documentación**. En la práctica sí agregó configuración: nueve decisiones, todas nacidas de divergencias entre lo que el documento declaraba y lo que los equipos hacían.

---

## Teoría cubierta

La teoría de este bloque es la del oficio de verificar: qué prueba realmente una prueba, y cómo no confundir "el componente anda" con "el diseño cumple".

### Validación end-to-end: qué prueba y qué no
Cada bloque anterior verificó su propia capa con su propio `show`. Un `show` prueba que *un componente hace lo que dice hacer*. Nada prueba que *la composición de todos hace lo que el diseño prometió*.

El error clásico es sumar `show`s y llamarlo validación: OSPF `FULL`, HSRP `Active`, BGP `Established`, túnel `up/up` → "la red anda". Ninguno de los cuatro prueba que un usuario de V20 abra la app. Este proyecto ya tenía la prueba: **RPVST+ corría en 3 de 6 switches** (hallazgo B7) y el `show spanning-tree` de cada switch se veía perfecto — el que faltaba era el que compara. Las tres capas se apilan, no compiten: la verificación por componente dice **dónde** está roto; la matriz end-to-end dice **si** está roto; el monitoreo continuo (Zabbix, proyecto NOC) dice **cuándo** se rompió. Dicho corto: un `show` prueba que un componente hace lo que dice; una matriz de conectividad prueba que el diseño hace lo que prometió.

### Por qué el deny pesa más que el permit
Un permit que funciona prueba que no rompiste el servicio. Un deny que funciona prueba que **el control existe**.

La asimetría es el argumento completo: los permits fallan **ruidosamente** — el usuario reclama, entra un ticket. Los denies fallan **en silencio**: nadie llama al helpdesk a decir *"puedo llegar a la base de datos y no debería"*. El deny es la única mitad que la operación normal jamás va a descubrir por ti, y por eso es la que hay que probar deliberadamente. Es también la evidencia que pide una auditoría PCI: no *"el sistema funciona"*, sino *"el acceso no autorizado está bloqueado"*.

### Cómo se lee el síntoma (el diagnóstico está en el resultado)

| Resultado en VPCS | Qué significa |
|---|---|
| `code 13` (*admin prohibited*) | Un router **te contestó**: una ACL denegó **en la ida** y el equipo que la aplicó devolvió el unreachable. Sabes quién y dónde. |
| `timeout` | Nadie contestó. **Ambiguo:** murió en la ida sin generar unreachable, **o llegó bien y la respuesta murió en el retorno**. |
| `host not reachable` | **Falla de ARP.** El echo nunca se emitió: murió una capa antes de ICMP. |
| `host unreachable` desde el gateway | No hay ruta. |

**Regla: `code 13` = deny de ida, identificado. `timeout` = hay que abrirlo con contadores.** El desempate es `show ip access-lists` en los dos extremos: si el permit de ida incrementó y aun así hay timeout, el problema está en el retorno. Es lo que destapó el bug de `V20 → V30` en el B7. En logs de ICMP el par `(X/Y)` es `type*256 / code*256`: `(2048/0)` = type 8 (echo request); `(768/3328)` = type 3 code 13.

### La regla de método: predicción antes del ping
El resultado esperado de cada celda **se escribe antes de correr el ping**. Si pingueas primero y racionalizas después, no estás probando la matriz — la estás justificando.

En este bloque la regla se pagó tres veces: las tres divergencias del Tier 1-3 fueron **predicciones equivocadas**, no fallas de la red. Sin predicción escrita, cada una habría pasado como "salió bien" sin que nadie notara que salió bien **por otro mecanismo**.

### Qué es un failover y cómo se prueba
`show standby` diciendo `Active`/`Standby` prueba que el protocolo eligió roles. **No prueba que al morir el activo el tráfico sobreviva**, y sobre todo no dice **cuánto dura el hoyo** — que es lo único que el cliente siente. Redundancia configurada y redundancia funcionando son cosas distintas, y la diferencia solo aparece rompiendo algo a propósito.

**El ping continuo no es el objetivo: es un reloj.** Un paquete por segundo = una muestra por segundo. Los paquetes perdidos **son la unidad de medida**. Es lo que convierte *"conmutó"* en *"conmutó en 5 segundos"*. De ahí salen tres reglas: **acota, no mide** (con 1 pps, un corte de 0-1 paquetes significa *"menos de ~2 s"*, no *"800 ms"* — la convergencia sub-segundo de RPVST+ no es medible con este instrumento, y se declara así en vez de inflarla); **el "durante" no se recaptura** (el ping tiene que estar corriendo antes de romper; un failover no se repite para la foto); y **qué rompes cambia qué mides** (`shutdown` de la interfaz → link-down, detección inmediata; muerte sin link-down → detección por expiración del hold timer — dos números distintos para "el mismo" failover).

### `Administrative` vs `Operational` en un switchport
`Operational Mode: static access` puede convivir con `Administrative Mode: dynamic auto`. `Operational` dice `static access` **solo porque del otro lado no hay nadie pidiendo nada**. El día que aparece un DTP `desirable` al otro extremo, cambia a `trunk` sin que nadie toque una config — que es la definición del ataque. En un `show switchport`, la línea que importa para seguridad es `Administrative Mode`: `Operational` te dice el estado de hoy; `Administrative` te dice qué está autorizado a pasar mañana.

### `neighbor default-originate` vs las otras dos formas
Hay tres formas de anunciar una default por BGP y solo una sirve para un ISP de tránsito:

- **`network 0.0.0.0`** → exige que el router **tenga** una default en su RIB. Un ISP de tránsito no la tiene: *él es* el destino por defecto. Falla.
- **`default-information originate` (BGP)** → exige redistribución de una default existente. Mismo problema.
- **`neighbor <ip> default-originate`** → **la fabrica de la nada, por vecino, sin necesitar tenerla.** La única que funciona cuando el que anuncia es la raíz.

`network` y `default-information originate` anuncian una default que ya tienes; `neighbor default-originate` la fabrica.

### Distancia administrativa: quién entra a la RIB
Al aplicar `default-originate`, los bordes aprendieron `0.0.0.0/0` por eBGP y **no la instalaron**, porque la estática (AD 1) le gana a eBGP (AD 20). IOS lo dice en voz alta: `Paths: (2 available, best #1, ..., RIB-failure(17))` y `show ip bgp rib-failure` → *"Higher admin distance"*. Traducido: *"la aprendí, la elegí como mejor camino de mi protocolo, y no la instalé"*. No es un error: es la AD decidiendo **quién entra a la RIB**, no quién es mejor en su propio protocolo. Y **BGP anuncia desde su tabla, no desde la RIB** — un prefijo en `RIB-failure` se sigue anunciando.

### NAT con route-map: la interfaz como condición
`ip nat inside source list X interface Gi2 overload` significa *"traduce lo que matchee X, siempre a la IP de Gi2"*. La interfaz ahí **no es una condición, es el resultado**: no pregunta por dónde sale el paquete, solo de dónde saca la IP global.

Con multihoming eso falla peor que no natear: si el ruteo manda el paquete por Gi3, el statement lo traduce igual a la IP de Gi2 — un prefijo sin camino de vuelta. El tráfico sale, nadie vuelve, y todos los `show` se ven bien. La forma con route-map convierte la interfaz en **condición** (`match interface Gi2` = *"solo si el ruteo decidió que sale por acá"*): el NAT deja de tener opinión propia y **sigue a la tabla de ruteo**.

### LocalPref vs AS-path prepend, verificados en el lab
**LocalPref controla el saliente** porque es local a tu AS. **AS-path prepend controla el entrante**, y no es determinista: solo puedes hacer un camino menos atractivo y esperar que el otro AS elija por longitud de path.

El prepend del B5 nunca se había visto **surtir efecto** — el `show ip bgp` del B5 mostraba el prepend en el path, no la decisión de un tercero. En el B8 sí: `show ip bgp 192.0.2.101/32` en BR-STGO devolvió el path `65002 65001 65010`, o sea **ISP-2 peerea directo con BR-VALPO y descartó su propio enlace**, porque BR-VALPO le anuncia con `65010 65010 65010` (3 saltos) y vía ISP-1 son 2. Un AS que no controlas eligiendo lo que querías que eligiera.

### NTP: por qué es prerequisito del SOC, no infraestructura de fondo
**El hallazgo que lo motivó:** durante el failover de ISP no se pudo correlacionar el syslog de ISP-1 (`19:19:54`) con el de BR-STGO (`hace 00:16:56`). Cada nodo corría su propio reloj. El timeline se reconstruyó con **contadores `MsgRcvd`**, no con horas.

Un SIEM que recibe logs de 10 equipos sin reloj común **no puede ordenar dos eventos de la misma cadena de ataque**. Un timeline forense con relojes a la deriva no es evidencia. **Estratos:** 0 = reloj físico (GPS/cesio), 1 = servidor pegado a él, +1 por salto. `ntp master 3` = *"declárate estrato 3 usando tu propio reloj"* — se inventa la hora, y la referencia `127.127.1.1` (`.LOCL.`) **es** la prueba. Es lo mismo que hace un servidor NTP de producción, solo que él la saca de un pool público o un GPS. **UTC, sin `clock timezone`:** Chile cambia a horario de verano y la hora entre 23:00 y 00:00 ocurre dos veces en otoño — UTC en la infraestructura, hora local solo en la pantalla del analista. **La autenticación no es ceremonia:** sin ella, cualquiera en la VLAN de gestión se hace pasar por el servidor y corre el reloj de los 8 equipos — es un ataque anti-forense: mueves el tiempo y los logs dejan de correlacionar, sin borrar una sola línea. **`reach` es un registro de corrimiento en octal** (los últimos 8 polls): `377` = 8 de 8, `177` = perdió el más viejo, `1` = perdió 7 seguidos. Es el health check de NTP y casi nadie lo sabe leer.

### PAT con ICMP: por qué 10 pings son 10 traducciones
ICMP no tiene puertos. PAT usa el campo **Identifier** de la cabecera ICMP como discriminador, y VPCS lo incrementa por paquete. Por eso 10 echoes producen 10 entradas con "puerto" distinto. Con TCP verías una entrada por sesión, no por segmento.

---

## Qué se hizo

### Punto 1 — Matriz de conectividad end-to-end: 59/59

| Tier | Qué validó | Pruebas |
|---|---|---|
| 1 | Política intra-sitio STGO (§7.2) | **28/28** |
| 2 | Política intra-sitio VALPO | **12/12** |
| 3 | Inter-sitio (permit + deny) | **7/7** |
| 4 | Plano de gestión (VTY permit + deny) | **7/7** |
| 5 | Aislamiento L2 (protected ports) | **2/2** |
| 5' | NAT / Internet | **3/3** |

Cada celda con **predicción escrita antes de correr**, cada contador atribuido al dígito con `clear ip access-list counters` previo. Cero matches inexplicados, cero matches faltantes.

**Paso 0 obligatorio — barrido de IPs.** El inventario de hosts era del B7. Los 4 invitados (lease 2 h, ~9 h transcurridas) tenían **lease en cero y la IP funcionando igual**: §11.1 demostrado en vivo — VPCS no implementa T1, así que nunca renovó y el servidor consideró la IP libre hace 7 horas. Sin el barrido, cuatro celdas se apoyaban en direcciones que el `show ip dhcp binding` ya no respaldaba.

**Lo que la matriz probó sin que nadie lo pidiera:**

- **La alineación HSRP/STP del B3-B7, medida.** Los matches cayeron **solo en el switch activo** de cada VLAN (V10/V15 en DLS1, V20/V50 en DLS2, V40 en MLS1, V41/V55 en MLS2), y los `code 13` llegaron desde su IP real. El tráfico transita físicamente el switch planificado: el *"sin hairpinning"* del diseño, medido en estado normal en vez de afirmado. (La corrección de que en estado degradado la alineación se rompe está en Problemas.)
- **La regla del HSRP en las ACL, desde los dos lados.** En el switch *standby* de cada VLAN, todos los contadores en cero **salvo `permit ip any any`**: son los hellos del vecino entrando por la SVI y pasando de rebote. Es por qué V15 —la única lista que cierra en `deny any`— necesita el permit explícito.
- **§5.4 aislado con una sola variable.** Desde `V41A`: hacia internet → 10 traducciones a `198.51.100.2`; hacia `10.1.30.11` → **cero**. Mismo host, mismo borde, misma config de NAT. La única diferencia es el destino, o sea `NAT-EXCLUDE-VPN`. No queda otra explicación posible.
- **El filtro inter-sitio es más estrecho que la política intra-sitio.** `V41A` es Gestión —*"administra todo"* dentro de VALPO, sin ACL que la toque— y cruza a STGO **sin alcanzar la app**. Ser admin en tu sede no te hace admin en la otra.
- **La sumarización del syslog, en vivo.** `2 eventos de syslog para 10 paquetes`, con 5 min exactos entre ellos. Una correlación que cuente *eventos* en vez de leer el campo `N packets` **subcontaría 5×**. Vale para el proyecto SOC.

### Punto 2 — Failover real

| Evento | Corte | Failback | Control |
|---|---|---|---|
| **Caída de ISP-1** (`shutdown` de su interfaz hacia BR-STGO) | **~129 s** · 64 pkt | **~5 s** · 0 pkt | `V40A` 383/383 ✅ |
| **Caída del uplink de DLS1** (`shutdown` Gi1/0) | **~5 s** · 3 pkt | espera de **~30 s** · 0 pkt | `S20A` 313/313 ✅ |

**El contraste es el resultado del bloque: 25× de diferencia entre dos fallas del mismo tipo.** Cuando el equipo que detecta **es** el que ve morir el link (`track 1 line-protocol`), reacciona al instante. Cuando tiene que inferir la muerte del silencio, paga un hold timer completo.

**Tres afirmaciones del diseño quedaron medidas, no supuestas:**

- **`preempt delay minimum 30` evitó 7 s de ruteo a ciegas.** Track Up `20:43:30` → OSPF FULL `20:43:38` → HSRP toma el activo `20:44:01`. **22 s de margen.** Sin el delay, DLS1 habría preempteado en ~1 s con la tabla vacía.
- **Y se comió 5 flaps.** `show track 1` pasó de 10 a **15 changes** con un solo `shutdown`+`no shutdown` (que son 2). El uplink flapeó al recuperarse; HSRP conmutó **una** vez, no cinco. Los dos argumentos con que el B7 justificó ese comando, demostrados en un evento.
- **La prioridad 105 cruzó el umbral:** `105 − 10 = 95 < 100`. Con la 150 original del B7 habrías visto `Pri 140` y un `Active` que no se mueve.

**El `Tag` de BGP como instrumento:** la default trae el AS de origen tatuado. `Tag 65001` → `Tag 65002` → `Tag 65001` dice por qué ISP salió el tráfico **en un solo `show`**, sin traceroute. Y el `ttl` del ping lo confirma desde el otro lado: `253` vía ISP-1, **`252` vía ISP-2** (un salto más, el peering ISP-2→ISP-1). Los **171 paquetes seguidos a `ttl=252`** no dicen "reconvergió": dicen que **el usuario tuvo internet de forma sostenida por el camino de respaldo**. La corrida anterior tenía tabla `Tag 65002` y `NAT-ISP2 refcount 1`, y el usuario sin internet (ver B8-#5).

### Punto 3 — Entrega (parcial)

- **8 configuraciones congeladas y sanitizadas** → `configs/`, con `README.md` declarando la sanitización y los artefactos de fábrica.
- El resto de la entrega (diagramas, `ARQ-001`, documento de entrega, este informe) se completó en la sesión de cierre documental.

---

## Decisiones y por qué

### B8-#1 · Política de puerto no usado (VLAN 888)
**El hallazgo:** §7.0 declaraba *"DTP deshabilitado"*. Era **falso**. El `switchport nonegotiate` se aplicó a las troncales —donde se pensó el control— y nunca tocó lo que nadie configuró. Los 12 puertos libres de los 4 switches L3 estaban en `dynamic auto` / `Negotiation: On` / `Trunking VLANs Enabled: ALL`.

**Por qué no es cosmético:** un puerto en `dynamic auto` no inicia negociación pero **acepta** convertirse en troncal si el otro lado la pide (*switch spoofing*, Yersinia). Y `Trunking VLANs Enabled: ALL` significa que no pasa a "una VLAN": pasa a **todas**. La VLAN nativa 999 **no cubre este ataque** — mitiga *double tagging*, que es el otro. La cadena que lo hace grave sale del propio diseño: puerto libre → trunk negociado → el atacante etiqueta como **VLAN 30** → aterriza en la única VLAN **sin ACL de política** (§7.2) y la única **autorizada a las VTY** de los 8 equipos (§7.3). No entra (le faltan credenciales), pero **el control de §7.3 deja de existir**: la ACL de VTY asume que estar en `10.1.30.0/24` es difícil.

**Se eligió cerrar la clase, no la instancia:** un bloque de validación es donde se cierran clases. Arreglar solo los puertos que miraste no es validar, es parchar. **VLAN 888 y no 998:** 998 es un typo de distancia de 999, y ese typo aterriza el puerto parqueado en la nativa — el fallo exacto que el número venía a evitar. **Cuatro candados:** `switchport mode access` (mata la negociación — el control), `nonegotiate` (deja de emitir DTP), `access vlan 888` (si se levanta, aterriza aislado), `shutdown` de puerto y de VLAN (el segundo mata el riesgo el día que alguien haga `no shut` sin pensar). **Y no se fabricaron puertos para demostrarlo:** los dos ALS están al 100% de ocupación y cumplen la política por no tener superficie. *"Los agregué para mostrar que sé apagarlos"* no es una respuesta de entrevista.

### B8-#2 · `deny ip any any log` en el filtro inter-sitio
**El hallazgo:** el `deny` explícito de `INTERSITE-FILTER-IN` solo cubre `10.2.0.0/16 → 10.1.0.0/16`. Todo lo demás caía en el **deny implícito**, que **no cuenta y no loguea**. Un ping entre las IPs internas del túnel desapareció sin dejar ni un contador ni una línea de syslog en ningún equipo. Es al revés de lo que quieres: **el tráfico raro es justo el que tiene señal**.

**Por qué el `log` aquí no tiene el costo que tiene en otras listas:** el túnel es un canal autenticado por IPSec — solo llega lo que el otro borde cifró, así que el volumen esperado está acotado por diseño. Cumple el criterio de §7.2: *"un log vale cuando su volumen esperado es cero"*. **Se pagó sola en 15 minutos** y atrapó tres casos legítimos el mismo día: el SSH de `ALS1-STGO` a `10.255.2.2` (B8-#3b), los unreachables del filtro espejo, y 16 consultas DNS por broadcast desde BR-STGO (B8-#9). **El mecanismo que explica `timed out` en vez de `refused`:** BR-VALPO deniega el SSH → genera un unreachable sourceado de `10.255.0.2` → cruza el túnel → BR-STGO lo deniega porque `10.255.0.2 ∉ 10.2.0.0/16`. La notificación del deny muere en el mismo control que produjo el deny.

### B8-#3 y #3b · Acceso de gestión asimétrico + loopbacks
**El hallazgo:** §7.2 dice *"Gestión administra todo"* y era verdad **solo dentro de su propia sede**. La ACL de VTY se escribió por sede y nadie preguntó para qué existía el permit inter-sitio. Resultado: los equipos de VALPO solo se administraban desde `10.2.41.0/24` — **y en VALPO no hay staff de TI**. Un equipo que solo se administra desde una sede sin administradores no es un control: es un equipo inadministrable.

**Se eligió asimétrico sobre simétrico:** los 4 equipos de VALPO aceptan además `10.1.30.0/24`; STGO no se toca. La función de TI vive en STGO, y la dirección que se abre es la que ya está más protegida — Gestión-VALPO (la sede con invitados y menos control físico) sigue sin llegar a las VTY de la matriz. **Se descartó el permit por host:** el tráfico intra-VLAN es invisible para las ACL, así que la frontera de confianza real es *"estar en la VLAN de gestión"*, que es lo que el `/24` expresa. El permit por host sería documentación disfrazada de control.

**B8-#3b — el loopback, con el argumento del B5 aplicado a otra capa:** `BR-VALPO` era el único de los 8 sin dirección dentro del rango que el filtro deja cruzar. Se le agregó `Loopback41 10.2.41.254/32` (y su simétrico `Loopback30` en BR-STGO), anunciado en OSPF. **Por qué un loopback y no ensanchar el filtro:** `10.255.2.2` vive en `Gi3` — si ese uplink cae, la dirección desaparece y pierdes la administración justo cuando la necesitas. Es la decisión del B5 aplicada al plano de gestión, y `10.2.41.254` ya cae dentro del permit existente, así que el filtro no se toca. **Verificación que lo prueba:** al hacer SSH al loopback, la línea 20 incrementó y la línea 40 no. **El costo:** comprometer `10.1.30.0/24` pasa de dar 4 equipos a dar 8, y administrar un borde por el túnel que ese borde termina es circular — la respuesta real es out-of-band (§11). `Loopback30` resultó ser, además, la dirección del servidor NTP de toda la red (B8-#7).

### B8-#4 · Default por BGP + NAT con route-map
**El hallazgo, el más grande del bloque:** §1 dice *"multihoming en ambas sedes, porque las dos son sedes de producción que no pueden quedar sin salida"*. Era falso. Los dos bordes tenían **una** estática por defecto hacia ISP-1 y **un** solo statement de NAT atado a su interfaz.

Ante la caída de ISP-1: la conectada desaparece → el next-hop no resuelve → la estática se cae de la RIB → `default-information originate` (sin `always`) retira su LSA → el interior pierde la default. El túnel sobrevive (§6.4 se cumple: anclado a loopback aprendido por ambos ISP). **Internet muere. El multihoming protegía la VPN, no el negocio.** Es la misma clase que §7.0: el documento promete una propiedad global, la config la entrega en un subconjunto.

**Se eligió default por BGP sobre floating static** por tres razones: (1) **convierte el trabajo del B5 en portante** — la LocalPref 200/100 gobernaba 2 prefijos /32 y ahora decide por dónde sale todo el internet de la sede; (2) **cubre un modo de falla que la flotante no cubre** — BGP detecta por teardown de sesión, no solo por link-down, y el caso "ISP muerto con el cable arriba" es el que agujerea en silencio, y **el lab lo reprodujo sin querer**; (3) **es lo que hace una empresa multihomed de verdad**. **El orden de operaciones es lo que no se improvisa:** (1) `default-originate` en los ISP → impacto cero por AD; (2) NAT con route-map → impacto cero, el ruteo sigue por ISP-1; (3) retirar la estática ← el único paso con riesgo, cuando 1 y 2 están verificados. **El paso 3, medido:** cero paquetes perdidos, y la LSA del interior con su timestamp intacto (`13:50:19 ago` en DLS1). Se cambió el protocolo que sostiene la salida a internet de dos sedes y el IGP no registró ni un evento.

### B8-#5 · El instrumento estaba roto
Tras el primer failover, la tabla decía `Tag 65002` y el NAT `NAT-ISP2 refcount 1`. Todo verde. **Y el usuario sin internet.**

**La causa:** vía ISP-2, PAT traduce a `203.0.113.6`. El echo llega a `192.0.2.1` (ISP-1, vivo). **ISP-1 no tenía ruta de vuelta a `203.0.113.4/30`** — el sustrato del lab se armó en el B5 con lo mínimo para que BGP levantara. El hueco quedó latente porque todo el tráfico salía por ISP-1. **Se arregló el sustrato** (cada ISP anuncia sus /30 de cliente — lo que hace un ISP real) porque los ISP son andamiaje del lab, no la red que se entrega: arreglarlos no cambia el diseño, arregla el instrumento de medición. **Esto es la teoría del Punto 1 cobrándose:** tabla + `refcount` = el failover de ruteo y NAT funcionó; **no probaban que el usuario tuviera internet**. Estuve a punto de firmar B8-#4 con la evidencia que este bloque existe para rechazar.

### B8-#6 · Filtro de salida BGP: el tránsito accidental
**El hallazgo, con el comando que nadie corre:** `show ip bgp neighbors 203.0.113.5 advertised-routes` → `Total number of prefixes 7`. BR-STGO le anunciaba a ISP-2 **siete prefijos**: su loopback + los seis que aprendió de ISP-1, re-anunciados con `65010` al frente del AS-path — que en BGP significa *"puedes alcanzar esto a través de mí"*. **Incluida `0.0.0.0/0`**: AustralPay ofreciéndole a un AS de tránsito una ruta por defecto.

**Un cliente multihomed sin filtro de salida se anuncia como tránsito entre sus dos upstreams.** Es la causa raíz de una familia entera de incidentes de BGP reales. El leak existía desde el B5 —`PASS-OUT` era `permit 10` sin `match`— y era invisible porque no había nada que filtrar. **La corrección: 3 líneas por borde** (prefix-list con el loopback propio + `match` en los dos route-maps `out`). **De 7 a 1 en las cuatro sesiones.** Se detectó con `show ip bgp neighbors advertised-routes`, que es el comando que nadie corre.

### B8-#7 · NTP con `BR-STGO` como reloj
**Se eligió el borde STGO sobre ISP-1:** el NTP existe para correlacionar logs, y el momento en que más necesitas correlacionar es durante una falla — un reloj que depende de un ISP se pierde justo cuando lo necesitas, y este mismo bloque midió 129 s sin ISP-1. Además, el colector de Zabbix y el manager de Wazuh van en STGO. **El gotcha que define el punto — `ntp source`:** sin él, `MLS1-VALPO` sourcearía de `10.255.2.1`, que no matchea `10.2.41.0/24` → el filtro inter-sitio lo mata; y `BR-VALPO` sourcearía del túnel → caería en la línea 40 de B8-#2. Es §7.3 en versión chica: *"el control no mira quién eres, mira desde dónde vienes"*. **Resultado:** `BR-VALPO` sincronizado a **2,5 ms a través del túnel GRE-over-IPSec con MD5**. El permit `10.2.41.0 → 10.1.30.0` del Bloque 5 pasó de transportar pings de verificación a transportar el reloj de la sede. **El asterisco:** el `*` que precedía cada log del proyecto significa *"mi reloj no está sincronizado"*; al sincronizar desaparece.

### B8-#9 · `no ip domain lookup`: el typo que sale a internet
**El hallazgo:** BR-STGO emitía consultas DNS a `255.255.255.255:53`, cada 4 s, 16 veces. `ip domain-lookup` viene activo por defecto y no hay `ip name-server`: cuando IOS no reconoce un comando en modo exec, asume que es un hostname al que hacer telnet e intenta resolverlo por todas las interfaces. **Ese broadcast salió por `Gi2` y `Gi3` — hacia los ISP.** En una red real es un typo de la consola de tu router de borde viajando a internet en claro, y a veces el typo es una contraseña mal pegada. Sin la línea 40 de B8-#2, era invisible. No toca `ip domain name australpay.lab`: uno nombra al equipo (y sostiene la llave RSA), el otro busca nombres ajenos.

*(La corrección relacionada — `no ip http server` / `no ip http secure-server` en los 8, la 8ª divergencia — está en las divergencias más abajo.)*

---

## Evidencia capturada

| # | Captura | Qué muestra |
|---|---|---|
| **B8-#1 — Puerto no usado** | | |
| 36 | `DLS1 Gi1/1 switchport` (antes) | `dynamic auto` · `Negotiation: On` · `Trunking VLANs Enabled: ALL` — la prueba de que el control hacía falta |
| 37 | `DLS1 status` + `switchport` (después) | `disabled` / VLAN 888 · `static access` · `Negotiation: Off` |
| 38 | `show interfaces trunk` ×4 | **888 no aparece** en ninguna troncal — el aislamiento no es una afirmación |
| 39 | `show vlan brief` | `888 PARKING-NO-USADO act/lshut` |
| **Punto 1 — Matriz** | | |
| 40 | `S10A` · `S15A` · `S20A` · `S30A` · `S50A` — 28 pings | Tier 1 completo, con los `code 13` desde el activo de HSRP de cada VLAN |
| 41 | `S50A → S50B` | **`host not reachable`** — protected ports mata el **ARP**, no el ICMP |
| 42 | `show ip access-lists` DLS1 + DLS2 | Contadores atribuidos al dígito · `POL-V15-DB` 40 matches |
| 43 | Syslog de `POL-V15-DB` | La DB iniciando 5 flujos — detector de exfiltración/C2 disparando |
| 44 | Tier 2 — 12 pings VALPO + contadores | Reparto HSRP invertido respecto a STGO |
| 45 | `V41A → 10.1.10.10` = T/O + syslog en BR-STGO | **Gestión-VALPO no alcanza la app de STGO** |
| 46 | `show ip ospf neighbor` ×2 + `ping 10.255.0.1` 0/5 | **Túnel sano con ping muerto** — el chequeo de salud de §7.1 |
| 47 | `INTERSITE-FILTER-IN` ×2 con contadores limpios | permit 5/5 · deny 5/5, exacto |
| 48 | `ALS1-STGO → DLS1` login vs `BR-STGO → DLS1` refused | **Equipo propio + credenciales válidas ≠ acceso** |
| 49 | `show ip nat translations` BR-VALPO | `10.2.41.11` → internet **traducido** · `10.2.41.11 → 10.1.30.11` **sin traducir** |
| **B8-#2 / #3 / #4 / #6** | | |
| 50 | `ping 10.255.0.1` sin rastro (antes) → con syslog (después) | El deny implícito dropea a ciegas |
| 51 | `ssh 10.2.41.254` desde `ALS1-STGO` + `INTERSITE-FILTER-IN` | **L20 sube, L40 no** — el acceso asimétrico funcionó como se diseñó |
| 52 | `show ip bgp rib-failure` | *"Higher admin distance"* — aprendida, elegida, no instalada |
| 53 | `show ip route 0.0.0.0` **`Tag 65001`** + LSA `13:50:19 ago` | Migración a BGP transparente para el IGP |
| 54 | `advertised-routes` **7 prefijos** (antes) → **1** (después) | Tránsito accidental, cerrado |
| **Punto 2 — Failover** | | |
| 55 | `S20A` ping continuo — **64 timeouts, luego 171 pkt a `ttl=252`** | Internet **sostenido** por ISP-2 |
| 56 | `show ip route 0.0.0.0` **`Tag 65002`** + `NAT-ISP2 refcount` | El tag delata el ISP en un solo `show` |
| 57 | `V40A` **383/383** durante los dos tests de ISP | Control: la falla no se propagó a VALPO |
| 58 | `S10A` — **3 timeouts** + `%HSRP-5-STATECHANGE` ×3 en 1,1 s | Failover de HSRP |
| 59 | `show track 1` — `Down` · **15 changes** · 5 grupos | Un objeto, cinco grupos · el track comiéndose los flaps |
| 60 | `show standby brief` DLS1 **`Pri 95`** / DLS2 **`Active`** ×5 | 105−10=95 cruzó el umbral |
| 61 | Syslog: track Up `43:30` → OSPF FULL `43:38` → HSRP `44:01` | **22 s de margen** del `preempt delay` |
| 62 | `S10A trace` → `10.1.10.3` + MAC table `vMAC por Po2` | **El track no evita el hairpin** — corrección a §6.1 |
| **B8-#7 — NTP** | | |
| 63 | `show ntp status` BR-STGO `synchronized` + `show clock` **sin `*`** | Reloj autoritativo · `.LOCL.` = hora inventada |
| 64 | `show ntp associations` BR-VALPO — `reach 377` · **offset −1 ms** | Sincronizado **a través del túnel**, con MD5 |
| 65 | `show ntp associations` de los 6 switches — offset 1.000-4.750 ms | Evidencia de §11.1: la config es correcta, la plataforma no |
| 66 | `INTERSITE-FILTER-IN` L20 subiendo sola cada 64 s | El reloj de VALPO cruzando el permit del B5 |
| **B8-#9 + higiene** | | |
| 67 | Syslog: `denied udp 10.255.0.1(x) -> 255.255.255.255(53)` ×16 | El typo saliendo por la VPN y hacia los ISP |
| 68 | `show ip sockets` / `show tcp brief all` ×8 | **Ni 80 ni 443 escuchando** — §7.3 ya es verdad |

---

## Problemas / aprendizajes

### Las once divergencias entre lo declarado y lo real
El bloque de validación existe **porque el documento y los equipos divergen sin avisar**. Once casos:

| # | Declaraba | Realidad |
|---|---|---|
| 1 | §7.0 *"DTP deshabilitado"* | 12 puertos en `dynamic auto` |
| 2 | *"`Gi2/2` y `Gi2/3` están libres"* | **No existen** — el nodo tiene 10 puertos (`Ethernets: 10`) |
| 3 | §4.3 *"`Loopback0` = `10.255.255.x`, router-id"* | Esa interfaz **no existe**; el router-id está fijado a mano y `Loopback0` es el **público que ancla la VPN** |
| 4 | §6.1 *"el track mantiene el tráfico directo por el switch que sí tiene uplink"* | **Falso** — ver abajo |
| 5 | §7.2 documenta que el host no pinguea su gateway | **No documenta el reverso**: el gateway no puede pinguear a sus hosts |
| 6 | §1 *"las dos sedes no pueden quedar sin salida"* | Una estática a ISP-1. Sin internet ante su caída |
| 7 | §5.2 multihoming | AustralPay se anunciaba como **tránsito** entre sus dos ISP |
| 8 | §7.3 *"Acceso: solo SSHv2"* | `ip http server` **y** `ip http secure-server` escuchando en los 8, con auth local y **sin `ip http access-class`** |
| 9 | Config de `ALS1-VALPO` | En **VTP server**: sus VLANs vivían solo en `vlan.dat`. **El archivo no reconstruía el equipo** |
| 10 | §7.3 *"`exec-timeout 10 0`"* | No estaba configurado. **Era el default** — la promesa se cumplía por accidente |
| 11 | §7.2 *"Nueve listas de política"* | **Siete.** El número era correcto antes de la decisión *"V30 y V41 sin ACL"* del B7, y nadie recontó |

**El patrón se repite:** el documento declara una propiedad **global**, la ejecución la aplica donde se **pensó**, y el servicio o el puerto que nadie tocó se queda con su default. Nadie lo nota porque **lo configurado se ve perfecto**. **El caso #3 es el más peligroso:** en el B6 se reconstruyeron los dos bordes desde el diseño + memoria. Si vuelve a pasar, alguien lee §4.3, configura `interface Loopback0 / ip address 10.255.255.2` y le pisa el anclaje del túnel al equipo. La tabla no está incompleta: está en el punto exacto donde equivocarse tumba la VPN.

*(Las once están consolidadas en el changelog del `ARQ-001`, cada una con su corrección propagada al cuerpo del diseño maestro.)*

### El track no evita el hairpin (corrección a §6.1)
§6.1 afirmaba que el track sirve para *(a)* mantener el tráfico directo por el switch con uplink y *(b)* evitar el agujero negro ante una segunda falla.

**(a) es falso, y por STP.** El `traceroute` tras el failover sale por `10.1.10.3` (DLS2, el nuevo activo), pero la tabla MAC de ALS1 muestra la vMAC `0000.0c9f.f00a` aprendida por `Po2` — hacia DLS1. **DLS1 sigue siendo root de V10**, así que el puerto de ALS1 hacia DLS2 está BLK. Camino real: `S10A → ALS1 → DLS1 → peer-link → DLS2 → BR-STGO`. **La trama entra por el switch que perdió su uplink, cruza el peer-link, y se rutea en el otro.** Sin track, DLS1 seguiría activo y rutearía por la adyacencia OSPF de VLAN 99 hacia DLS2: un salto de peer-link en ambos casos. Solo cambia si cruza bridgeado (V10) o ruteado (V99).

**No es un error de config: es un límite del diseño.** Alinear root con activo optimiza el estado normal; en el degradado la alineación se rompe sola porque solo uno de los dos mecanismos se movió. Arreglarlo cuesta ~30 s de reconvergencia STP para ahorrar un salto de peer-link — no vale la pena, y eso es lo que hay que saber decir. **(b) sigue en pie y es la justificación completa: el track vale, por una sola de sus dos razones declaradas.**

### Los nueve errores, y por qué el método los atrapó
El valor de escribir la predicción antes es que **el que se equivoca queda registrado**. Nueve veces en este bloque:

| Predije | Realidad | La lección |
|---|---|---|
| `S50A → S50B` daría **timeout** | **`host not reachable`** | El ARP muere **antes** que el ICMP. `switchport protected` descarta el broadcast → sin MAC no hay trama que enviar. Prueba más fuerte que la esperada: un timeout sería compatible con "el ICMP salió y se perdió"; esto no admite otra lectura |
| El ping entre IPs del túnel matchearía la línea 30 en **BR-VALPO** | **Deny implícito**, y en **BR-STGO** | La ACL está `in`: el echo sale sin evaluarse y lo juzga el otro extremo. Y `10.255.0.2 → 10.255.0.1` no está en ninguno de los dos /16 del deny |
| Contador en 3 = hallazgo | Era **lag del datapath** | En IOS-XE el ACL lo evalúa el forwarding processor (`%FMANFP-`) y los contadores se agregan con retraso. Un contador se relee antes de declararlo hallazgo |
| El NAT nuevo rompió `V40A` | **`V40A` se había reiniciado**: `IP/MASK 0.0.0.0/0` | Casi reverso un cambio correcto y verificado. Ver abajo |
| Los switches derivan **8,7% = 175× el rango de NTP** | Falso: **inestabilidad, no pendiente**. Y el offset **bajaba** | Tomé dos muestras de una curva errática y las traté como una recta |
| *"La mitad alta del split-scope jamás sirvió"* | **Sirvió en el B7** (`10.2.55.133/.134` en el log) | Lo que vimos es que tras un `dhcp -r` MLS1 gana la carrera del primer OFFER. El split-scope no está sin ejercitar: es **no determinista** |
| Hold timer = **180 s** | **129 s** | El hold no cuenta desde la falla: cuenta desde el último keepalive recibido. Con keepalive cada 60 s, el corte cae entre 120 y 180 según cuándo murió el enlace en el ciclo |
| `preempt delay` **no** aplicaría al failback | **Aplicó: 30,5 s exactos** | El delay cuenta desde que el router queda en condiciones de preemptear, no desde que nace la interfaz. Y el error destapó el mejor dato del punto: los 22 s de margen sobre OSPF |
| El túnel recuperó **sin re-formar** la adyacencia OSPF | **Se reformó las dos veces** | Pedí el comando que podía desmentirme y me desmintió |

**Y una fuera de la red:** se afirmó, mirando el reloj de pared, que *"entre los mensajes pasaron horas"* — los timestamps de los equipos decían que la red había reconvergido exactamente cuando debía. Usar la conversación como evidencia sobre el estado de la red es justo lo que este bloque demostró nueve veces que no se puede hacer.

### El caso `V40A`: coincidencia temporal ≠ causalidad
Tras aplicar el NAT con route-map, `V40A` dejó de salir a internet. El síntoma apareció **justo después** del cambio y apuntaba directo a él. **Lo que lo salvó fue exigir el `show` antes del rollback:** `V41A`, `S20A` y `V55A` salían a internet 5/5 con la config nueva, lo que dejó a `V40A` solo en la lista de sospechosos. Y el `show ip` lo cerró: `IP/MASK: 0.0.0.0/0` — **el nodo se había reiniciado**. Con máscara `/0`, todo el universo es on-link → ARPea al destino directo → `host not reachable`. La aritmética cerró: `POL-V40-OFICINA` L40 = 15 + 5 + 5 (los echoes con origen `0.0.0.0`) = 25. **Revertir habría enterrado la causa real bajo un rollback "exitoso".** Un síntoma que aparece justo después de un cambio no prueba que lo causó el cambio.

### El entorno, no la red: un síntoma para seis incidentes
El bloque acumuló anomalías que se estaban tratando por separado: OSPF perdiendo dead timers sin evento de red (incluso de madrugada, sin nadie conectado); `interface resets` y `lost carrier` en enlaces que nunca se tocaron; relojes NTP con offset de 1-5 s y `reach` cayendo a `1`; `track 1` con 15 transiciones; dos VPCS reiniciándose solos; los dos CSR corrompidos a `grub>` en el B6.

**Todo eso es un solo síntoma: las VMs se congelan bajo presión de CPU del host** (27/32 GB). Un equipo congelado no manda hellos, no contesta polls y pierde ticks de reloj. El CSR1000v aguanta (offset 2,5 ms); el vIOS-L2 no (1.000-4.750 ms, drift clavado en los 500 ppm que son el tope del algoritmo). **No es un hallazgo de red — es del entorno**, y va declarado como tal. Pero explica seis cosas que parecían independientes. (Consolidado en §11.2 del diseño maestro.)

### Sorpresas de plataforma acumuladas
- **`ip domain lookup` con espacio en CSR1000v** (IOS-XE); el vIOS-L2 acepta el guión. Y el `show run | include ip domain-lookup` hereda el bug: no encuentra la forma con espacio, así que **el comando de verificación reporta ausente algo que está**. Se verifica con `include ip domain`. Un comando de verificación que no puede encontrar lo que busca es peor que no verificar: da un falso negativo con cara de dato.
- **`show interfaces trunk | include Vlans allowed on`** devuelve solo el encabezado: `include` filtra línea por línea y la línea con los datos no contiene ese texto. Un `show` que no puede fallar no es una verificación. Se usa `begin`.
- **Los puertos salen `connected` sin nada enchufado** (artefacto de EVE-NG). Los contadores lo delatan: `53238 packets output, 0 packets input`. En hardware real dirían `notconnect`. Va declarado — la evidencia sanitizada lo muestra.
- **El `shutdown` de un ISP no baja el link del borde**, por lo mismo. No es un defecto: es el modo de falla "ISP muerto con el cable arriba" que B8-#4 nombró, reproducido sin querer, y el que una floating static con link-down no habría cubierto nunca. Es el argumento de BFD al desnudo.
- **Las keys tipo 7 con el mismo salt producen el mismo hash.** Dos pares de equipos coincidieron: es la prueba de que la contraseña es idéntica en los 8. Si alguno tuviera un typo, no habría coincidido con nadie.

---

## Cierre del bloque: qué quedó probado y qué pasó al diseño maestro

Este bloque cerró la verificación completa: la matriz de conectividad 59/59 con predicción escrita antes de cada celda, el failover de ISP (129 s / 5 s) y de HSRP (5 s / 0 pkt) medidos con ping continuo y control, las nueve correcciones de configuración nacidas de las once divergencias, y las 8 configuraciones congeladas y sanitizadas.

Todo lo que este bloque produjo como **recomendación de producción, limitación de evidencia o pregunta abierta** se consolidó en el diseño maestro (`ARQ-001`, secciones 11 a 11.3), en vez de quedar como una lista de pendientes de informe:

- **Recomendaciones para producción (§11):** BFD (detección sub-segundo — argumento medido: 129 s vs 5 s entre dos fallas del mismo tipo); NTP con fuente externa trazable (`ntp master` es un reloj inventado: hora relativa correcta, absoluta falsa); acceso out-of-band para los bordes (administrar un borde por el túnel que ese borde termina es circular).
- **Limitaciones de evidencia (§11.1):** NTP en vIOS-L2 (offset de 1.000-4.750 ms por congelamiento de VM, contra 2,5 ms en los CSR con el mismo servidor y camino — la config es correcta y funcionaría en hardware real); impacto del failover sobre sesiones NAT existentes (no medible con VPCS, que crea una traducción nueva por ping y no sostiene flujos de larga vida — exige TCP).
- **Preguntas abiertas declaradas (§11.2), sin respuesta inventada:** UDP 2228 (6 switches) y TCP 21111 (2 bordes) en escucha, sin identificar; por qué unas sesiones se caen y otras no; cuánto tardó el túnel en re-formar su adyacencia OSPF (no medible sin reloj común — el hallazgo que motivó B8-#7).
- **Deltas documento ↔ lab (§11.3):** el nodo `Intruso` que sigue en el `.unl` sin aparecer en el diagrama; la asimetría de documentación in-config entre VALPO (sin `description` en `Gi1/0` y varias SVI) y STGO.

Los entregables de cierre —los tres diagramas de topología, el `ARQ-001` con las once divergencias propagadas y su changelog, el documento de entrega y este informe— se completaron en la sesión de cierre documental posterior a la verificación.
