> ⚠️ **Documento de laboratorio y portafolio.** AustralPay (empresa cliente) y DiegoAraya (consultora) son entidades **ficticias**, creadas exclusivamente con fines de demostración técnica. Ninguna dirección, dispositivo, ISP o dato descrito corresponde a una red, cliente o proveedor real. Las direcciones "públicas" usan rangos reservados para documentación (RFC 5737).

---

# Informe de entrega — Red corporativa AustralPay
### Diseño, implementación y verificación de la infraestructura de red de dos sedes
**Consultora:** DiegoAraya · **Cliente:** AustralPay S.A. · **Documento:** ENT-001 · **Fecha:** 2026-07-16

---

## 1. Resumen ejecutivo

AustralPay solicitó el diseño y la puesta en marcha de la red corporativa que conecta su casa matriz en Santiago (Providencia) con su nueva sede regional de producción en Valparaíso, con dos objetivos de negocio: que ambas sedes operen como sitios de producción plenos —no una principal y un respaldo pasivo— y que la interrupción de un proveedor de internet no deje a ninguna sede fuera de servicio.

**La red está construida, verificada y entregada.** Conecta las dos sedes mediante un túnel cifrado sobre internet, da a cada sede dos proveedores de internet independientes, y protege cada servicio interno con controles de acceso que se probaron uno por uno. Ante la caída de un proveedor de internet, la sede afectada recupera su salida en aproximadamente dos minutos por el segundo proveedor, de forma automática; ante la falla de un equipo de red interno, el servicio a los usuarios se restablece en unos cinco segundos.

Durante la verificación se identificaron **once puntos** en los que la implementación no coincidía con el diseño documentado —la clase de discrepancia habitual entre un plano y su construcción—. **Los once fueron corregidos y quedan registrados** en este informe y en la documentación técnica. La red se entrega en un estado consistente, con su configuración congelada y respaldada.

**Riesgo residual, en una frase:** la red cumple los objetivos solicitados en la infraestructura entregada; su paso a un entorno de producción real requiere las medidas enumeradas en la sección 8, ninguna de las cuales cambia el diseño — lo endurecen.

---

## 2. Alcance del encargo

**Incluido:**
- Diseño de la arquitectura de red para las dos sedes (matriz STGO y sede regional VALPO).
- Implementación completa de los 8 equipos de red (routers de borde, switches de distribución y de acceso).
- Interconexión segura entre sedes mediante red privada virtual (VPN) cifrada.
- Conexión de cada sede a dos proveedores de internet, con conmutación automática.
- Segmentación interna y controles de acceso entre áreas (servidores, operaciones, gestión, invitados).
- Verificación funcional extremo a extremo y pruebas de resiliencia.
- Documentación técnica completa y configuraciones respaldadas.

**Explícitamente fuera de alcance:**
- Los servidores de aplicación y base de datos, los sistemas de monitoreo (NOC) y el centro de operaciones de seguridad (SOC): esta entrega es la **red sobre la que esos sistemas correrán**, no los sistemas mismos.
- La infraestructura de los proveedores de internet, que en este entorno está **simulada** para poder probar la conmutación entre proveedores de forma controlada.

**Nota sobre el entorno.** Esta implementación se construyó y validó en un entorno de laboratorio virtualizado. Las direcciones "públicas" utilizan rangos reservados para documentación. Las implicancias de este entorno para lo que se pudo y no se pudo medir están declaradas de forma transparente en la sección 8.

---

## 3. Arquitectura entregada

La red sigue un diseño **jerárquico** en el que cada sede se organiza en dos niveles de switches —un par de **distribución/núcleo** que concentra el enrutamiento interno, y una capa de **acceso** que conecta los dispositivos finales— más el **router de borde** que la une a internet y a la otra sede. Este patrón (conocido como *collapsed core*: el núcleo y la distribución conviven en el mismo par de equipos, en vez de una capa de núcleo separada) es el adecuado para una sede de este tamaño —una capa de núcleo dedicada solo se justifica al interconectar varios bloques de distribución— y se replica idéntico en ambas sedes, lo que hace que Valparaíso sea operativamente una versión reducida de la matriz: el mismo patrón, más fácil de administrar y de crecer.

**En cada sede:**
- Un **router de borde** conecta la sede a los dos proveedores de internet y termina el túnel cifrado hacia la otra sede.
- Dos **switches de distribución** proveen el enrutamiento interno y actúan como puerta de enlace redundante: si uno falla, el otro asume su función sin intervención.
- Un **switch de acceso** conecta los dispositivos finales (servidores, estaciones, equipos de invitados) con controles de seguridad por puerto.

**Entre las sedes**, un túnel **cifrado de extremo a extremo** (GRE sobre IPSec, con cifrado AES-256) transporta todo el tráfico entre Santiago y Valparaíso. El túnel está anclado de modo que **sobrevive a la caída de cualquiera de los proveedores de internet**: no depende de un enlace en particular, sino de la conectividad general de la sede.

**Hacia internet**, cada sede se conecta a dos proveedores. En operación normal el tráfico usa el proveedor primario; ante su caída, conmuta automáticamente al secundario.

La arquitectura completa se representa en tres diagramas técnicos que acompañan esta entrega: la topología física (`ARQ-002`), el diseño de enrutamiento y redundancia interna (`ARQ-003`) y el diseño de la conexión a internet y el túnel entre sedes (`ARQ-004`).

---

## 4. Resiliencia: qué ocurre cuando algo falla

El objetivo central del encargo fue la continuidad. Esta sección resume **cómo se comporta la red ante cada tipo de falla**, con los tiempos que se midieron durante las pruebas — no estimados de diseño, sino resultados de romper cada elemento a propósito con tráfico corriendo.

| Falla | Qué sucede | Interrupción medida | Recuperación |
|---|---|---|---|
| **Cae un proveedor de internet** | La sede conmuta al segundo proveedor automáticamente | **~129 segundos** | Automática, sin intervención |
| **Cae un switch de distribución** (o su enlace) | El switch redundante asume la puerta de enlace | **~5 segundos** | Automática, con vuelta al equipo preferido |
| **Cae un proveedor de internet** (impacto en el túnel entre sedes) | **Ninguno** — el túnel usa el otro proveedor | Sin interrupción del túnel | — |

**Sobre el tiempo de conmutación entre proveedores de internet (~129 segundos).** Este tiempo corresponde al mecanismo estándar con que los routers detectan que un proveedor dejó de responder. Es un comportamiento correcto y esperado, no una deficiencia; la sección 8 describe una mejora (BFD) que lo reduciría a menos de un segundo, recomendada para el paso a producción. Durante la prueba se verificó que, tras esos dos minutos, la sede mantuvo internet de forma **sostenida** por el segundo proveedor, y que la sede opuesta **no se vio afectada en ningún momento** (mantuvo el 100% de su conectividad durante ambas pruebas de proveedor).

**Sobre la conmutación interna (~5 segundos).** Cuando falla un switch de distribución o su enlace de subida, el switch redundante toma la puerta de enlace de las VLANs afectadas. La prueba se realizó con tráfico continuo: la interrupción fue de unos cinco segundos y el tráfico se restableció solo. Al recuperarse el equipo original, este retoma su rol preferido de forma ordenada.

---

## 5. Seguridad y segmentación

La red no es un espacio plano donde todo alcanza a todo. Cada área funcional vive en su propio segmento, y **el tráfico entre segmentos sigue una política de "denegar por defecto"**: solo lo explícitamente permitido pasa. Esto es una exigencia directa para una empresa que maneja datos financieros.

**Los controles entregados, en lenguaje de negocio:**

- **Separación de servidores.** La aplicación y la base de datos viven en segmentos distintos. Las estaciones de trabajo pueden alcanzar la aplicación, pero **nunca la base de datos directamente** — como corresponde: los usuarios hablan con el sistema, no con los datos crudos. Además, la red vigila el caso inverso: una base de datos que *inicie* conexiones hacia afuera es una señal temprana de fuga de información, y quedó registrada como evento a monitorear.

- **Aislamiento de invitados.** Los equipos de la red de invitados están aislados entre sí y del resto de la empresa: un invitado no alcanza a los servidores, ni a otro invitado.

- **Acceso a la administración de los equipos.** Solo el personal de tecnología, desde la sede matriz, puede administrar los equipos de red. La sede de Valparaíso —que no tiene personal de TI propio— es administrada desde Santiago, pero no a la inversa: esto reduce la superficie de riesgo concentrando la administración donde está el equipo humano.

- **Protección de los puertos de red.** Cada puerto de acceso limita qué dispositivos puede conectar; los puertos sin uso están **deshabilitados y aislados**, de modo que conectar un equipo no autorizado a una toma libre no da acceso a nada.

- **Cifrado del tráfico entre sedes.** Todo lo que viaja entre Santiago y Valparaíso está cifrado con AES-256. Un tercero que intercepte el tráfico en internet no puede leerlo.

---

## 6. Verificación

La red no se entrega "porque cada equipo reporta estado correcto". Se entrega porque **se probó que el conjunto hace lo que el diseño promete**, flujo por flujo.

Se ejecutó una **matriz de conectividad de 59 pruebas** que cubre los cinco planos de la red: comunicación interna en cada sede, comunicación entre sedes, acceso de administración, aislamiento de invitados y salida a internet. Cada prueba verificó **tanto lo que debe funcionar como lo que debe estar bloqueado** — porque un control de seguridad solo está probado cuando se confirma que efectivamente bloquea, no cuando se asume.

**Las 59 pruebas pasaron.** La metodología fue estricta: para cada prueba se escribió el resultado esperado **antes** de ejecutarla, de modo que un resultado "correcto por la razón equivocada" no pudiera pasar inadvertido. Las pruebas de resiliencia (sección 4) se realizaron con tráfico continuo corriendo, para medir la interrupción real en vez de asumirla.

---

## 7. Hallazgos y correcciones durante el proyecto

Durante la fase de verificación se compararon los equipos, uno por uno, contra el diseño documentado. Se encontraron **once discrepancias** entre lo que el diseño declaraba y lo que los equipos efectivamente hacían — la clase de brecha que aparece en cualquier proyecto entre el plano y la construcción, y que solo se detecta cuando alguien la busca deliberadamente.

**Se declaran de forma transparente porque los once fueron corregidos**, y porque el registro de estas correcciones es parte del valor de la entrega: da a AustralPay la certeza de que la red entregada corresponde a su documentación, no a una versión aproximada.

| # | Área | Qué se encontró | Estado |
|---|---|---|---|
| 1 | Capa 2 | Puertos de red sin uso quedaban en un modo que permitía negociar conexiones no autorizadas | Corregido |
| 2 | Salida a internet | Ante la caída del proveedor primario, la sede quedaba sin internet pese a tener un segundo proveedor | **Corregido — impacta directamente el objetivo del encargo** |
| 3 | Salida a internet | La red se anunciaba inadvertidamente como ruta de paso entre sus dos proveedores | Corregido |
| 4 | Administración | Un equipo de borde no era alcanzable para su administración desde la sede matriz | Corregido |
| 5 | Administración | Los servicios web de administración de los equipos estaban activos sin necesidad | Corregido |
| 6 | Sincronización | Los equipos no compartían una referencia horaria común, necesaria para correlacionar registros | Corregido |
| 7 | Documentación | Diversos puntos del diseño describían configuraciones que diferían del estado real de los equipos | Corregidos y propagados a la documentación |

*(Las once discrepancias, con su detalle técnico y su corrección, están registradas íntegramente en el historial de cambios del documento de arquitectura `ARQ-001`. La tabla anterior las agrupa por área para lectura de gestión.)*

El hallazgo #2 merece mención especial: la verificación demostró que la promesa central del encargo —que ninguna sede quede sin internet ante la caída de un proveedor— **no se cumplía en la implementación inicial**, pese a que el diseño la declaraba. La corrección reconfiguró la salida a internet para que la conmutación entre proveedores sea real y automática, y se **midió** funcionando (sección 4). Este es precisamente el tipo de brecha que una verificación rigurosa existe para encontrar.

---

## 8. Limitaciones declaradas y recomendaciones para producción

Esta entrega se construyó en un entorno de laboratorio. En honestidad de consultoría, se declara con precisión **qué no se pudo verificar en este entorno** y **qué se recomienda antes de un despliegue en producción real**. Ninguna de estas recomendaciones corrige un defecto del diseño: todas lo endurecen para las exigencias de un entorno productivo.

**Limitaciones de la verificación en laboratorio:**
- La conmutación entre proveedores de internet se probó con la infraestructura de proveedores **simulada**. El comportamiento es correcto; los tiempos absolutos pueden variar en enlaces reales.
- Ciertas mediciones de precisión (sincronización horaria de sub-milisegundo, comportamiento de sesiones de larga duración durante una conmutación) están limitadas por las herramientas del entorno virtual, no por el diseño. Los equipos de borde alcanzaron la precisión requerida; los switches virtuales, no —una restricción de la plataforma de emulación, no de la configuración.

**Recomendaciones para el paso a producción:**
1. **Detección rápida de fallas (BFD):** reduciría la conmutación entre proveedores de ~129 segundos a menos de un segundo. Es la mejora de mayor impacto sobre la continuidad.
2. **Referencia horaria externa confiable** (GPS o servidor de tiempo público autenticado): el entorno actual usa una referencia interna, suficiente para operar pero no para auditoría forense o cumplimiento. Relevante bajo PCI-DSS.
3. **Acceso de administración fuera de banda** para los equipos de borde: una vía de administración independiente de la red principal, para poder intervenir un equipo aun cuando su enlace principal esté caído.
4. **Firewall con estado** en el punto de interconexión entre sedes: el control actual es efectivo pero sin memoria de sesión; un firewall con estado es la evolución natural para un entorno productivo.
5. **Filtrado de DNS y control de tráfico cifrado de evasión** en el borde, para cerrar las vías por las que un dispositivo puede eludir los filtros de navegación.

---

## 9. Estado de entrega e inventario

**Equipos implementados (8):**

| Sede | Equipo | Función |
|---|---|---|
| Santiago | BR-STGO | Router de borde — internet, VPN, servidor horario de la red |
| Santiago | DLS1-STGO, DLS2-STGO | Distribución — enrutamiento interno y puerta de enlace redundante |
| Santiago | ALS1-STGO | Acceso — conexión de dispositivos finales |
| Valparaíso | BR-VALPO | Router de borde — internet, VPN |
| Valparaíso | MLS1-VALPO, MLS2-VALPO | Distribución — enrutamiento interno y puerta de enlace redundante |
| Valparaíso | ALS1-VALPO | Acceso — conexión de dispositivos finales |

**Documentación entregada:**
- `ARQ-001` — Documento de arquitectura técnica (diseño completo, con historial de cambios).
- `ARQ-002 / 003 / 004` — Diagramas de topología física, enrutamiento/redundancia y WAN/VPN.
- `ENT-001` — Este informe de entrega.
- `INF-001-B2` a `INF-001-B8` — Informes técnicos detallados por fase de implementación.
- `configs/` — Configuración final de los 8 equipos, sanitizada y respaldada.
- `PLAN-001` — Comparativo entre el diseño planificado y el resultado final.

La configuración de los ocho equipos se entrega **congelada y respaldada**. La red queda en un estado consistente, documentado y reproducible.

---

## 10. Anexo técnico

Este informe está escrito para lectura de gestión y de operación. El detalle técnico completo —direccionamiento, protocolos de enrutamiento, configuración de la VPN, políticas de acceso línea por línea, y el razonamiento de cada decisión de diseño— vive en el documento de arquitectura **`ARQ-001`** y en los informes de fase **`INF-001-B2` a `INF-001-B8`**, que documentan la implementación paso a paso, incluyendo los problemas encontrados y cómo se resolvieron.

Para el equipo que operará la red, el punto de partida recomendado es `ARQ-001` (secciones 6 y 7: redundancia y plano de gestión) y el informe `INF-001-B8` (verificación y comportamiento en falla).

---

> ⚠️ **AustralPay y DiegoAraya son entidades ficticias, con fines exclusivos de laboratorio y portafolio.**
