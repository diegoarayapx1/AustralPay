> ⚠️ **Proyecto de laboratorio y portafolio.** AustralPay y la consultora DiegoAraya son entidades **ficticias**, creadas exclusivamente con fines de demostración técnica y desarrollo profesional. Nada aquí corresponde a una red, cliente o proveedor real; las direcciones "públicas" usan rangos reservados para documentación (RFC 5737). Este documento **no** representa trabajo realizado para un cliente real.

---

# AustralPay — Informe final maestro
### Diseño, implementación y validación de una red empresarial de dos sedes

---

## Contexto

AustralPay es una fintech ficticia con dos sedes: la matriz en Santiago y una sede regional de producción en Valparaíso. El proyecto consistió en diseñar, construir y verificar desde cero la red corporativa que las conecta: enrutamiento interior OSPF multiárea, multihoming BGP a dos ISP en cada sede, un túnel GRE-over-IPSec cifrado entre sedes, redundancia por capa (HSRP, EtherChannel, RPVST+) y una política de segmentación de seguridad completa. Se implementó sobre EVE-NG con equipos Cisco (CSR1000v en el borde, vIOS-L2 en el interior) a lo largo de ocho bloques de trabajo, cada uno cerrado con su verificación y su documentación.

Este documento no relata el proyecto paso a paso —para eso están los informes de fase—. Organiza lo que el proyecto **demuestra**, por competencia, con la evidencia concreta de cada una.

---

## El arco del proyecto

El proyecto tiene una historia que vale la pena contar en tres frases, porque explica por qué se ve como se ve.

**Empezó como un diseño de red de dos sedes de alcance acotado** —seis bloques, con la verificación al final—. **A mitad de camino tomé la decisión deliberada de no quedarme en lo mínimo:** en vez de una topología que funcionara y una verificación somera, construí todas las capas que hacen a una red defendible —segmentación interna real, endurecimiento de acceso, separación de roles administrativos, una política de seguridad de lista blanca— e inserté dos bloques enteros que el plan no tenía. **El resultado fue un bloque de verificación final que encontró once puntos donde mi propia documentación no coincidía con los equipos, los corrigió, y registró por qué cada uno había ocurrido.** El contraste entre lo planificado y lo construido está documentado por separado en `PLAN-001`, y esa distancia *es* parte del trabajo.

Lo que sigue son las competencias que ese recorrido demuestra.

---

## Competencias demostradas

### 1. Diseño de arquitectura de red

**Qué demuestra:** capacidad de tomar un requerimiento de negocio (dos sedes de producción, sin punto único de falla de internet) y traducirlo en una arquitectura coherente, eligiendo entre alternativas con criterio.

- **Arquitectura *collapsed core* por sede**, no tres niveles con núcleo dedicado. Es la elección correcta a esta escala: un núcleo separado solo se justifica al interconectar varios bloques de distribución, y agregarlo aquí sería equipo ocioso. La misma lógica descartó un segundo router de borde por sede.
- **OSPF multiárea con los bordes como ABR reales** (área 0 en el túnel, área 1 en STGO, área 2 en VALPO). El multiárea no es decorativo: cada sede es su propia área y los bordes tienen una pata en el backbone y otra en el sitio.
- **Ambas sedes simétricas y multihomed.** VALPO se rediseñó de "sitio de respaldo colapsado en un switch" a sede de producción con par de distribución, HSRP interno y dos ISP propios — porque una sede que atiende tráfico propio necesita salida resiliente, no depender de la matriz.
- **Direccionamiento jerárquico** (`10.1/16` STGO, `10.2/16` VALPO, `10.255/16` infraestructura) y **RFC 5737** para el espacio público, en vez de quemar IPs ruteables reales.

**El criterio que lo respalda:** varias de las decisiones de diseño más fuertes fueron decisiones de *no* agregar algo. Se planificó un peering iBGP entre bordes y luego **se eliminó con argumento** —OSPF sobre el túnel ya resolvía la comunicación interna, y una sesión iBGP habría cargado los mismos prefijos que eBGP ya entregaba—. Documentar por qué se omite un protocolo es más valioso que configurarlo para cumplir una pauta.

### 2. Seguridad y segmentación

**Qué demuestra:** pensar la red como superficie de ataque, no solo como conectividad — la mentalidad de un rol de seguridad.

- **Política de segmentación de lista blanca (denegar por defecto)** dentro de cada sede: siete listas de control de acceso, validadas una por una. Las estaciones alcanzan la aplicación pero **nunca la base de datos directamente**; los invitados no alcanzan nada; la gestión se administra solo desde donde hay personal de TI.
- **Separación de aplicación y base de datos en segmentos distintos**, para que el tráfico entre ambas sea filtrable —antes era L2 puro, invisible a cualquier ACL— y para habilitar una **regla de detección**: una base de datos que inicia conexiones salientes es indicador de exfiltración, y quedó registrada como evento a vigilar.
- **Endurecimiento de capa 2 diferenciado por rol de puerto:** MAC sticky en servidores, rotación controlada en invitados, aislamiento entre invitados (*protected ports*), DHCP snooping contra servidores DHCP no autorizados, y una política de puertos no usados que los deja deshabilitados y aislados en una VLAN de estacionamiento.
- **Plano de gestión endurecido:** solo SSHv2, servidores HTTP/HTTPS apagados, separación de roles administrador/operador por privilegio, protección anti-fuerza bruta y acceso administrativo restringido por origen.
- **VPN cifrada AES-256 / SHA-256** entre sedes, anclada a loopbacks para sobrevivir la caída de cualquier ISP.

**El criterio que lo respalda:** las decisiones de seguridad más maduras también fueron de *no* hacer. No se bloqueó ICMP en la infraestructura —porque romper PMTUD cuelga las sesiones TCP grandes sobre el túnel, y el valor de seguridad de bloquear ping es casi nulo frente a ese costo—. No se puso una ACL en la VLAN de gestión —porque su política es "todo permitido" y una ACL `permit ip any any` sería ceremonia sin enforcement—. Saber qué controles *no* aplicar, y por qué, es criterio de seguridad, no desconocimiento.

### 3. Enrutamiento avanzado y WAN

**Qué demuestra:** dominio de los protocolos que hacen funcionar internet y las conexiones entre sitios, más allá del enrutamiento básico.

- **Multihoming BGP** a dos ISP por sede, con manipulación de atributos: LocalPref para controlar el tráfico saliente, AS-path prepend para influir el entrante. El prepend se **verificó surtiendo efecto real** —un ISP que no controlo eligió el camino que el prepend buscaba—, no solo apareciendo en la tabla.
- **Salida a internet por default aprendida vía BGP** de ambos ISP, con el NAT siguiendo al enrutamiento mediante route-map. Esto reemplazó una ruta estática al ISP primario que —según descubrió la verificación— dejaba la sede sin internet ante la caída de ese ISP, pese a que el diseño prometía lo contrario.
- **Filtrado de anuncios BGP** con prefix-list, tras detectar que la red se anunciaba inadvertidamente como ruta de tránsito entre sus dos ISP —un error real de configuración multihomed, encontrado con el comando que casi nadie ejecuta (`show ip bgp neighbors advertised-routes`)—.
- **Túnel GRE-over-IPSec con OSPF corriendo por dentro:** GRE aporta la interfaz ruteable y el transporte de multicast que OSPF necesita; IPSec aporta el cifrado. Ninguno hace el trabajo del otro, y saber por qué se necesitan los dos es la pregunta de diseño de esta WAN.
- **NTP autenticado** con jerarquía de estratos, como prerequisito de la correlación de logs que un SOC necesita.

### 4. Diagnóstico y resolución de problemas

**Qué demuestra:** la competencia central de un rol de operaciones — perseguir un síntoma hasta la causa raíz sin engañarse en el camino.

- **Diagnósticos de causa raíz reales**, documentados con su cadena de razonamiento: un switch de acceso que se comportaba como router por un default de plataforma; un fallo de SSH que era en verdad dos problemas superpuestos; una pérdida de internet que parecía causada por un cambio y era un nodo reiniciado; seis anomalías aparentemente independientes que resultaron ser un solo síntoma de entorno.
- **Método sobre intuición:** en la verificación final, cada prueba se escribió con su resultado esperado **antes** de ejecutarla, de modo que un resultado "correcto por la razón equivocada" no pudiera pasar. Ese método atrapó cerca de veinte predicciones propias equivocadas a lo largo del proyecto — y cada una quedó registrada, porque un error entendido es más valioso que un acierto sin explicar.
- **Lectura crítica de las herramientas:** aprender que un `show run` no evidencia una política cuando esa política es el valor por defecto; que un filtro de salida vacío no significa "no pasó nada" sino "no coincidió con mi búsqueda"; que un comando de verificación puede reportar ausente algo que está. Saber cuándo una herramienta miente es lo que separa verificar de creer.

### 5. Verificación y rigor de entrega

**Qué demuestra:** la diferencia entre "lo configuré" y "probé que funciona" — y entre "funciona" y "está entregado".

- **Matriz de conectividad de 59 pruebas, 59 aprobadas**, cubriendo los cinco planos de la red y probando tanto lo permitido como lo bloqueado — porque un control de seguridad solo está probado cuando se confirma que deniega.
- **Failover medido, no supuesto:** cada mecanismo de redundancia se probó rompiendo el elemento que protege, con tráfico continuo corriendo. Conmutación entre ISP en ~129 s, conmutación de gateway interno en ~5 s, con la sede opuesta intacta durante ambas — números reales, no estimaciones de diseño.
- **Once discrepancias entre diseño y equipos, encontradas y corregidas.** El bloque de verificación existió precisamente para buscar la brecha que aparece en todo proyecto entre el plano y la construcción. Encontrar once, corregirlas y **escribir por qué cada una ocurrió** es el núcleo del trabajo de un analista: no confiar en que "el sistema funciona" porque cada componente reporta verde, sino probar el conjunto y perseguir la diferencia.
- **Entrega profesional completa:** configuraciones congeladas y sanitizadas, documento de arquitectura con historial de cambios, informe de entrega en lenguaje de cliente, tres diagramas de topología, siete informes de fase, y las recomendaciones para llevar la red a un entorno de producción real.

---

## Qué sigue

Esta red es el cimiento de dos proyectos que corren sobre ella: un **NOC** (monitoreo con Zabbix) que consume su plano de gestión y su referencia horaria común, y un **SOC** (Wazuh) que consume sus logs correlacionados. Varias decisiones de este proyecto —la VLAN de gestión dedicada, el NTP autenticado, la separación app/db como regla de detección— se tomaron pensando en esos consumidores. La red no es un fin en sí misma: es la base sobre la que se construye la operación de seguridad.

---

## Dónde está el detalle

| Documento | Contenido |
|---|---|
| `ARQ-001` | Diseño técnico completo, con el razonamiento de cada decisión y el historial de cambios |
| `ARQ-002 / 003 / 004` | Diagramas: topología física · enrutamiento y redundancia · WAN/BGP/VPN |
| `INF-001-B2` … `B8` | Informes de fase: implementación paso a paso, con los problemas y sus soluciones |
| `ENT-001` | Informe de entrega en lenguaje de cliente |
| `PLAN-001` | Comparativo entre el diseño planificado y el construido |
| `configs/` | Configuración final de los 8 equipos, sanitizada |

---

> ⚠️ **AustralPay y DiegoAraya son ficticias, con fines exclusivos de laboratorio y portafolio. Este proyecto no fue trabajo para un cliente real.**
