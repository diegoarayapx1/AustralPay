> ⚠️ **AustralPay y la consultora DiegoAraya son entidades FICTICIAS, creadas exclusivamente con fines de laboratorio y portafolio. Ningún dato, dirección, dispositivo o infraestructura descrito aquí corresponde a una red o cliente real.**

# AustralPay — Comparativo: plan inicial vs. resultado final
### PLAN-001 · Cómo evolucionó el diseño entre el Bloque 1 y el Bloque 8

Este documento contrasta el diseño **tal como se planificó en el Bloque 1** (`01-DISENO`, 2026-07-11) con **el que se construyó y validó** (`ARQ-001`, 2026-07-16). No es un registro de errores: es el registro de un alcance que **creció a propósito**, y de un diseño que se **corrigió a sí mismo** cada vez que la ejecución contradijo al papel.

Se escribe porque la distancia entre las dos versiones *es* el trabajo. Un portafolio que muestra solo el estado final pulido está afirmando, sin decirlo, "planifiqué esto perfecto desde el día uno". Esto muestra lo contrario, que es más difícil y más honesto: un plan razonable que la práctica obligó a mejorar.

---

## 1. La foto de una línea

| | Plan inicial (B1) | Resultado final (B8) |
|---|---|---|
| **Bloques planificados** | **6** (el 6º era *"Verificación + documentación"*) | **8** (verificación terminó siendo el 8º) |
| **Diseño maestro** | 233 líneas | ~880 líneas |
| **VLANs** | 6 (10, 20, 30, 40, 41, 999) | **11** (nacieron 15, 50, 55, 99, 888) |
| **Sección de seguridad** | 5 viñetas (§7) | 6 subsecciones (§7.0–§7.5) |
| **Filas de changelog** | 5 | **más de 50** |
| **Sección de recomendaciones de producción (§11)** | no existía | §11 + §11.1 + §11.2 + §11.3 |
| **Divergencias documentadas entre diseño y equipos** | — (nadie las buscaba aún) | **11**, todas corregidas |

**Ninguna fila de arriba es alcance que se desbordó por accidente.** Cada una fue una decisión tomada durante la ejecución, y cada una tiene su fila en el changelog del `ARQ-001` con fecha, bloque y porqué. El changelog es, de hecho, el historial de git que este proyecto no tuvo hasta el final: el registro de cómo el diseño llegó a ser lo que es.

---

## 2. El gancho: el propio plan inicial fecha la ambición

La evidencia de que el alcance creció a propósito no hay que argumentarla — **está escrita en el documento inicial**. El §9 del `01-DISENO` (*"Qué sigue"*) listaba cinco bloques por delante, y el último era:

> *"**Bloque 6 — Verificación:** pruebas end-to-end, failover, diagramas finales."*

En el proyecto final, **la verificación es el Bloque 8**. Entre "diseñar VALPO" y "dejarse verificar" se insertaron **dos bloques enteros que el plan no tenía**:

- **Bloque 6 — Plano de gestión y AAA local:** SSHv2, usuarios nombrados, separación de roles admin/operador, endurecimiento anti-fuerza bruta. **El plan inicial resolvía toda la seguridad de gestión en una sola viñeta** (§7: *"Gestión: solo SSH, ACL en las VTY"*).
- **Bloque 7 — Segmentación y servicios:** las tres VLANs nuevas, la política intra-sitio de lista blanca, DHCP split-scope, endurecimiento diferenciado de puertos. **El plan inicial no tenía política intra-sitio en absoluto** — filtraba solo el tráfico inter-sitio.

No es que el plan estuviera mal. Es que a mitad de camino la decisión fue **no tomar el camino simple** —topología que anda, verificación somera, entrega— sino construir las capas que hacen defendible el proyecto: todo el hardening, todas las ACL, la separación de roles, la segmentación real. Los dos bloques insertados son esa decisión hecha cronograma.

---

## 3. Qué se planificó bien y no cambió

Vale decirlo primero, porque el núcleo del diseño **resistió** ocho bloques de ejecución sin necesitar rediseño:

- **IGP único (OSPF multiárea) con los bordes como ABR reales.** La estructura área 0 (túnel) / área 1 (STGO) / área 2 (VALPO) se planificó en el B1 y se construyó igual.
- **BGP solo en el borde, con NAT y `default-information originate`.** El principio de *no redistribuir la tabla de internet al IGP* se sostuvo entero.
- **Redundancia por capa:** HSRP (gateway) + EtherChannel (enlaces) + RPVST+ (capa 2), la tríada del B1, es la del B8.
- **GRE-over-IPSec** para que OSPF corra sobre el túnel — decidido en el B1, y su porqué (el crypto map no transporta multicast) resultó exacto.
- **RFC 5737** para el direccionamiento "público" y RFC 1918 para el interior.
- **IPv4 puro, direccionamiento jerárquico por sitio** (`10.1/16` STGO, `10.2/16` VALPO, `10.255/16` infraestructura).

Un diseño cuyo esqueleto sobrevive intacto a la construcción es un diseño bien pensado. Lo que cambió fue el músculo, no el hueso.

---

## 4. Qué cambió, y por qué — las decisiones grandes

### 4.1 VALPO: de DR frío a sede de producción con multihoming
**Plan inicial (primeras horas del B1):** VALPO era un sitio de recuperación ante desastres, colapsado en un solo switch, con una sola salida.

**Final:** sede regional de producción, con par de distribución (se agregó `MLS2-VALPO`), HSRP interno, y **multihoming propio a dos ISP** — simétrica a STGO.

**Por qué:** la narrativa realista es de expansión, no de respaldo. Una empresa que crece y abre una sede regional que atiende tráfico propio necesita salida a internet resiliente. El disaster recovery pasó a ser un *subproducto* de tener dos sedes de producción geográficamente separadas, no una función dedicada. El costo fue perder la simetría simple *"multihoming solo en la matriz"*; la ganancia fue un diseño que se parece a una red real, donde una sede de producción no depende de un enlace único.

### 4.2 iBGP: planificado, y luego eliminado
**Plan inicial:** `BR-STGO`↔`BR-VALPO` peereando por iBGP sobre los loopbacks (§4.3, §5.2). Cinco AS de ISP (65001–65004 + el propio 65010).

**Final:** **sin iBGP.** Los bordes se comunican por la adyacencia OSPF de área 0 sobre el túnel.

**Por qué (Bloque 6):** al llegar a configurarlo, se vio que una sesión iBGP cargaría exactamente los mismos dos prefijos (los loopbacks públicos /32) que cada borde ya aprende del otro por eBGP vía los ISP. El transporte interno va por OSPF sobre GRE; la salida a internet, por `default-information originate`. iBGP no aportaba nada que OSPF y eBGP no dieran ya. **Se documentó por qué se omitía en vez de configurarlo para cumplir una pauta** — un protocolo muerto en la config es peor que su ausencia explicada. Esta es la primera vez que el proyecto eligió *no* agregar algo, con argumento.

### 4.3 Los servidores: de una VLAN a dos
**Plan inicial:** `austral-app` y `austral-db` compartían la **VLAN 10 (Servidores)**.

**Final:** la base de datos vive en la **VLAN 15 (Servidores-DB)**, separada de la aplicación.

**Por qué (Bloque 6→7):** con app y db en la misma VLAN, el tráfico entre ellos era L2 puro — no pasaba por ninguna SVI, y **ninguna ACL podía filtrarlo**. Quedaba permitido por física, no por política. Separar la DB en su propia VLAN hizo que el salto app→db cruce una SVI y sea gobernable. Y habilitó algo que no estaba en el plan: una **regla de detección** — una base de datos que *inicia* conexiones hacia afuera es indicador clásico de exfiltración, y ahora eso es una línea de ACL con log. La conversación app→db que hoy sostiene medio argumento de HSRP/STP **es una decisión que el plan inicial no contenía**.

### 4.4 La política de seguridad: del perímetro al interior
**Plan inicial:** filtraba el tráfico **inter-sitio** (el que cruza cifrado por el túnel) y dejaba **abierto el intra-sitio** — el patrón clásico de perímetro duro, interior plano.

**Final:** lista blanca (default-deny) dentro de cada sitio — las estaciones alcanzan la aplicación, nunca la base de datos directo; los invitados no alcanzan nada; gestión administra dentro de su sede. Siete listas de política, cada una validada celda por celda.

**Por qué (Bloque 6→7):** una fintech donde cualquier estación llega a la base de datos sin nada que lo impida no es defendible, por mucho que el perímetro esté cifrado. El 90% del tráfico real es intra-sitio, y era justo el que no tenía política. Este es probablemente el cambio de mayor peso del proyecto, y **no existía en el plan**.

### 4.5 La salida a internet: de estática a BGP
**Plan inicial y hasta el Bloque 7:** una ruta estática por defecto hacia el ISP primario en cada borde, con el NAT atado a esa interfaz.

**Final (Bloque 8):** la default se aprende por BGP de **ambos** ISP, con `neighbor default-originate`, y el NAT sigue al ruteo vía route-map.

**Por qué:** la verificación destapó que §1 prometía *"ninguna sede queda sin salida"* y era falso — la caída del ISP primario dejaba la sede sin internet, porque tanto la estática como el NAT colgaban de esa única interfaz. El multihoming protegía la VPN (anclada a loopback), no el negocio. El cambio convirtió la manipulación de LocalPref del Bloque 5 —que hasta entonces gobernaba dos prefijos /32 casi sin tráfico— en la política que decide por dónde sale **todo el internet de la sede**. Esta corrección solo fue posible **porque el Bloque 8 existió para buscarla**.

---

## 5. La segunda historia: el diseño que se corrigió a sí mismo

El crecimiento de alcance (sección 4) fue deliberado. Hay una segunda clase de cambio, distinta: las **11 divergencias** que el Bloque 8 encontró entre lo que el diseño maestro *declaraba* y lo que los equipos *hacían*. Ninguna es alcance nuevo; todas son deriva silenciosa entre el papel y la máquina — el tipo de brecha que le ocurre a toda red del mundo y que casi nadie va a buscar.

Ejemplos del patrón:
- §7.0 decía *"DTP deshabilitado"*; **12 puertos** estaban en `dynamic auto`, porque el control se aplicó a las troncales y no a lo que nadie configuró.
- §7.3 decía *"acceso: solo SSHv2"*; los servidores HTTP y HTTPS **estaban escuchando en los 8 equipos**, con las credenciales de admin.
- §4.3 documentaba un `Loopback0 = 10.255.255.x` que **no existe** — y en un equipo que ya se reconstruyó una vez desde el documento, esa ficha estaba en el punto exacto donde equivocarse tumba la VPN.

**El patrón es siempre el mismo:** el documento declara una propiedad *global*, la ejecución la aplica donde se *pensó*, y lo que nadie tocó se queda con su valor por defecto — invisible, porque lo configurado se ve perfecto.

Encontrar y corregir estas once, y **escribir por qué cada una ocurrió**, es el núcleo de lo que un rol de analista N1 hace todo el día: no confiar en que "el sistema funciona" porque cada componente reporta verde, sino probar la composición y perseguir la diferencia. El proyecto empezó a hacerlo antes de que existiera el bloque que le puso nombre: el **Bloque 6** ya registraba discrepancias diseño↔equipo (`ip routing` fantasma, ruta duplicada), y el **Bloque 7** cazó cuatro *drifts* que llevaban entre 2 y 5 bloques sin que ningún ping los delatara.

---

## 6. Qué dice esta evolución sobre el método

Tres cosas que el contraste plan-vs-real deja a la vista, y que un diseño entregado como foto final escondería:

1. **El esqueleto se planificó bien y el músculo se construyó sobre la marcha.** Lo que no cambió (sección 3) prueba criterio de diseño; lo que cambió (sección 4) prueba criterio de ingeniería — saber cuándo el plan simple no alcanza.

2. **Las decisiones difíciles fueron a menudo decisiones de *no* hacer:** no configurar iBGP, no poner ACL en la VLAN de gestión, no bloquear ICMP, no fabricar puertos para demostrar que se saben apagar. Un diseño maduro se reconoce tanto por lo que omite con argumento como por lo que incluye.

3. **El proyecto se sometió a su propia prueba.** Las 11 divergencias y los ~20 errores de ejecución registrados en los informes no son manchas: son la evidencia de que la verificación fue real. Un bloque de validación que no encuentra nada, o no se corrió, o no se miró.

El diseño final no es el que se planificó. Es mejor — y el registro de por qué es más valioso que cualquiera de las dos versiones por separado.

---

> ⚠️ **AustralPay y DiegoAraya son ficticias, con fines exclusivos de laboratorio y portafolio.**
