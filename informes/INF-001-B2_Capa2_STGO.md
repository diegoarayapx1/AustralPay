# Informe de Bloque — Redes AustralPay · Bloque 2: Capa 2 STGO
Fecha: 2026-07-11

## Objetivo del bloque
Configurar la capa 2 del sitio matriz STGO: segmentación por VLANs, troncales
endurecidas, spanning-tree rápido con root planificado, agregación de enlaces y
seguridad de acceso en el switch de acceso.

## Teoría cubierta

### VLANs
Segmentación L2 en dominios de broadcast: la unidad de contención del radio de explosión. Aíslan tráfico, contienen el movimiento lateral y sostienen la separación que exige PCI-DSS.

Dos criterios gobiernan su uso en este diseño: **VLAN de gestión dedicada** —el plano de gestión no comparte segmento con el de usuario— y **VTP en modo transparent**, para que ningún switch con número de revisión más alto pueda sobrescribir la base de VLANs del dominio.

### Troncales, VLAN nativa, DTP y poda
Una troncal transporta múltiples VLANs etiquetando con 802.1Q (ISL está obsoleto); sin troncal haría falta un cable por VLAN.

- **VLAN nativa:** la única que viaja sin etiqueta en la troncal. Es el vector del *VLAN hopping* por double-tagging. Mitigación: nativa dedicada y sin uso (999), nunca la VLAN 1.
- **DTP:** negocia automáticamente la formación de troncales, y eso significa que un atacante puede negociar una. Mitigación: `switchport nonegotiate` y modo manual.
- **Poda:** `allowed vlan` con solo lo necesario. Es mínimo privilegio aplicado a capa 2.

### STP: por qué existe, y por qué RPVST+ y no MST
La redundancia física en L2 crea loops, y Ethernet **no tiene TTL**: un broadcast storm no se extingue solo, es infinito y colapsa la red. STP deja un camino activo y bloquea los redundantes.

| | Alcance | Convergencia | Cuándo se justifica |
|---|---|---|---|
| **STP clásico (802.1D)** | un árbol para todas las VLANs | 30-50 s | obsoleto |
| **RPVST+** | un árbol por VLAN | < 1 s | decenas de VLANs |
| **MST (802.1s)** | agrupa VLANs en instancias | < 1 s | cientos de VLANs |

**Se eligió RPVST+.** Con 3-4 VLANs, MST es sobre-ingeniería: agrega complejidad de configuración —regiones, revisiones, mapeo de instancias— a cambio de una eficiencia que a esta escala no se nota. El árbol por VLAN, además, habilita balancear la carga entre los dos switches de distribución y alinear cada root con el activo de HSRP (Bloque 3).

### Dejar puertos en forwarding: qué es legítimo y qué es antipatrón
- **PortFast** (solo en puertos de acceso): salta el delay de ~30 s en puertos de dispositivo final. Correcto y vigente. **No apaga STP.** Va siempre acompañado de BPDUGuard.
- **Forzar forwarding en troncales o apagar STP:** antipatrón — es crear un loop. Funciona en un laboratorio sin tráfico y colapsa en producción.
- La necesidad legítima que hay detrás de esa idea —usar ambos enlaces en vez de tener uno bloqueado— se resuelve con **EtherChannel** o con **balanceo per-VLAN**, no rompiendo STP.

### EtherChannel
Agrupa N enlaces físicos en uno lógico. STP ve un solo enlace y no bloquea ninguno: se usan todos los cables, y la caída de uno no dispara reconvergencia de STP.

- **Balanceo por flujo**, vía hash — no por paquete. Garantiza el orden de entrega; con pocos flujos el reparto puede quedar desparejo.
- **LACP (802.3ad):** estándar abierto, negocia el canal y **detecta desajustes de configuración entre los extremos**.
- **PAgP:** propietario de Cisco, superado por LACP. **Static (`on`):** sin negociación — si un lado queda mal configurado, el resultado es un **loop silencioso**.
- **Modos LACP:** `active` inicia la negociación, `passive` solo responde; passive-passive nunca forma el canal. Se usa active-active.

### Port-security
Limita qué MACs y cuántas puede haber por puerto. Frena laptops no autorizadas, switches piratas y **MAC flooding** — el ataque que llena la tabla CAM y convierte al switch en un hub, habilitando sniffing.

- **`maximum 2`:** cubre el caso real de PC + teléfono IP.
- **`sticky`:** aprende la MAC legítima y la fija en la configuración. Automático y persistente.
- **`violation`:** `protect` descarta en silencio (ciego), `restrict` descarta y genera log y trap SNMP, `shutdown` deja el puerto en err-disable (agresivo, sufre falsos positivos).

**Se eligió `restrict`:** rechaza al intruso y **genera el trap que el NOC (Zabbix) va a monitorear**, sin apagar un puerto de producción ante un falso positivo.

## Qué se hizo
- VLANs 10/20/30 + nativa 999 en DLS1, DLS2, ALS1; VTP transparent en los tres.
- Puertos de acceso en ALS1: Gi1/1 (PC1) VLAN 10, Gi1/2 (PC2) VLAN 20.
- Troncales 802.1Q en todos los enlaces switch-a-switch: native vlan 999, DTP off (nonegotiate), poda a VLANs 10,20,30.
- RPVST+ en los tres switches; root planificado: DLS1 primario en VLAN 10 y 30, DLS2 primario en VLAN 20 (secundarios cruzados).
- EtherChannel LACP (mode active): Po1 peer-link DLS1-DLS2, Po2 ALS1-DLS1, Po3 ALS1-DLS2.
- Port-security en Gi1/1-2: sticky, maximum 2, violation restrict, + PortFast + BPDUGuard.

## Decisiones y por qué
- **VTP transparent** en vez de server/client → evita la sobrescritura accidental de la base de VLANs.
- **Nativa dedicada 999** → cierra el VLAN hopping por double-tagging.
- **RPVST+** en vez de MST → pocas VLANs; el árbol per-VLAN da balanceo alineado con HSRP (Bloque 3).
- **Root planificado y alineado** (DLS1: VLAN 10 y 30 · DLS2: VLAN 20) → balanceo de carga, y base para alinear con el activo de HSRP evitando hairpinning.
- **LACP** en vez de static o PAgP → estándar, negocia y detecta desajustes.
- **`violation restrict`** en vez de shutdown o protect → rechaza y notifica al NOC sin castigar falsos positivos.

## Evidencia capturada
- `01-vlans-trunks.png` — `show interfaces trunk`: trunking, native vlan 999, allowed 10,20,30.
- `02-etherchannel-up.png` — `show etherchannel summary`: Po1/Po2/Po3 en `(SU)`, miembros en `(P)`.
- `03-stp-root.png` — `show spanning-tree root`: DLS1 root de VLAN 10/30, DLS2 root de VLAN 20.
- `04-portsecurity.png` — `show port-security address`: MACs SecureSticky en Gi1/1 (VLAN 10) y Gi1/2 (VLAN 20).

## Problemas / aprendizajes
- **vIOS-L2 exige `switchport trunk encapsulation dot1q` antes de `switchport mode trunk`** — la imagen todavía soporta ISL histórico. Los switches modernos, solo-802.1Q, no tienen ese comando.
- **vIOS-L2 tiene 4 puertos por módulo** (`Gi0/0-3`), y hay que planificar según eso. `Gi0/0` quedó ocupado por el peer-link, así que el enlace L3 hacia el borde (Bloque 3) va en `Gi1/0`.
- **Los puertos de acceso quedaban con DTP activo por defecto**; se cerró con `nonegotiate` y `access` explícito, junto con port-security.
- **EtherChannel elimina el bloqueo de STP dentro de cada par de cables, pero el triángulo de 3 switches sigue siendo un loop:** STP mantiene un channel en `BLK`, y es correcto que lo haga. El balanceo per-VLAN hace que ese bloqueo caiga en uplinks distintos según la VLAN.
- **La configuración vive en RAM hasta el `write memory`.** En EVE-NG, *Stop* conserva la startup y *Wipe* la borra.
