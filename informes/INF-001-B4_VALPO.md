# Informe de Bloque — Redes AustralPay · Bloque 4: Sitio VALPO (Capa 2 + Capa 3)
Fecha: 2026-07-12

## Objetivo del bloque
Levantar el sitio regional de producción VALPO completo: capa 2 (VLANs
40/41, RPVST+, EtherChannel, port-security) y capa 3 (SVIs, HSRP, VLAN de
tránsito OSPF, área 2, preparación del uplink dual-homed a BR-VALPO),
replicando el patrón validado en STGO (Bloques 2 y 3), con las dos
decisiones de simetría resueltas explícitamente.

## Teoría cubierta
Sin teoría nueva. VLANs, troncales, STP/RPVST+, EtherChannel y port-security (Bloque 2), más SVIs/`ip routing`, HSRP y OSPF/áreas/ABR (Bloque 3), ya se cubrieron con cuadro completo en STGO.

**Repetir la explicación en VALPO habría inflado el bloque sin aportar.** El valor de este bloque no es re-explicar los protocolos: es **demostrar que el criterio adquirido en STGO se aplica igual en un sitio distinto** —mismo patrón, distintos números— y defender las dos decisiones donde VALPO podía divergir de STGO y se optó por no hacerlo. Un diseño consistente entre sedes es más fácil de operar y de defender que dos sedes resueltas cada una a su manera.

## Decisiones y por qué

### Decisión A — VLAN de tránsito OSPF en VALPO
**Tensión.** MLS1 y MLS2 necesitan formar adyacencia OSPF en área 2, igual que DLS1-DLS2 en área 1. Vecinar directo sobre las SVIs 40/41 habría obligado a sacarlas de `passive-interface default` y habría generado adyacencias redundantes sobre el mismo par de switches.

**Opciones.** (A1) replicar el patrón de STGO con una VLAN de tránsito dedicada sobre el peer-link `Po1`; (A2) vecinar sobre una SVI de usuario.

**Decisión: A1.** Se reutilizó el mismo VLAN ID 99 —los dominios L2 están separados por sitio, no hay colisión real— con subred paralela `10.255.99.4/30` (MLS1 `.5`, MLS2 `.6`) e `ip ospf network point-to-point`. Mismo patrón de tránsito que STGO con subred paralela: **consistencia de diseño entre sedes, no solo entre switches.**

### Decisión B — dual-homing de BR-VALPO
**Tensión.** El diseño original de VALPO —heredado de cuando el sitio era un switch colapsado único— mostraba un solo enlace borde↔distribución compartido (`10.255.2.0/30`). STGO ya había migrado a dual-homing en el Bloque 3, para que el tracking de HSRP de cada switch vigile su propio uplink físico. **VALPO se había quedado asimétrico por inercia, no por criterio.**

**Opciones.** (B1) dual-homing simétrico a STGO, dos /30 independientes; (B2) mantener un solo enlace compartido.

**Decisión: B1.** `MLS1↔BR-VALPO` en `10.255.2.0/30` (MLS1 `.1`), `MLS2↔BR-VALPO` en `10.255.2.4/30` (MLS2 `.5`). Cada switch prepara su propia `Gi1/0` ruteada, sin adyacencia real todavía (BR-VALPO se despliega en el Bloque 5).

**Por qué importa.** Sin B1, el `track 1` de HSRP en el switch que no tocara físicamente al borde **no vigilaría nada real**: el failover de gateway por caída de enlace quedaría decorativo en la VLAN donde ese switch es el activo. Ambas sedes dual-homean el borde a los dos switches de distribución para que el tracking siga la **caída real del enlace**, no solo la muerte del switch.

### Decisiones heredadas de STGO, sin cambios
VTP transparent, nativa dedicada 999, RPVST+ sobre MST, LACP sobre static/PAgP, port-security sticky/max 2/restrict, HSRP sobre VRRP/GLBP, Enhanced Object Tracking sobre track directo y `passive-interface default` — todas justificadas en los Bloques 2 y 3, se aplican acá sin reabrir el debate.

## Qué se hizo

**Capa 2:**
- VLANs 40/41 + nativa 999 en MLS1, MLS2, ALS1; VTP transparent en los tres.
- Puertos de acceso en ALS1: Gi1/1 (PC1) VLAN 40, Gi1/2 (PC2) VLAN 41.
- Troncales 802.1Q: native vlan 999, DTP off (nonegotiate), poda a VLANs
  40,41 (99 agregada luego, solo en el peer-link Po1).
- RPVST+ en los tres switches; root planificado: MLS1 primario en VLAN 40,
  MLS2 primario en VLAN 41 (secundarios cruzados).
- EtherChannel LACP (mode active): Po1 peer-link MLS1-MLS2, Po2
  ALS1-MLS1, Po3 ALS1-MLS2.
- Port-security en Gi1/1-2 de ALS1: sticky, maximum 2, violation restrict,
  + PortFast + BPDUGuard.

**Capa 3:**
- `ip routing` habilitado + SVIs 40/41 en MLS1 (.2) y MLS2 (.3), /24 cada
  una, verificadas up/up y con conectividad cruzada.
- HSRP por VLAN: MLS1 activo (priority 105) en VLAN 40, MLS2 activo
  (priority 105) en VLAN 41 — alineado con el root de STP. `preempt` en
  ambos grupos.
- VLAN 99 de tránsito (`10.255.99.4/30`) sobre el peer-link Po1, dedicada
  a la adyacencia OSPF MLS1-MLS2 (decisión A).
- OSPF área 2 habilitado: `passive-interface default` global, excepción
  explícita solo en Vlan99 y Gi1/0; `ip ospf network point-to-point` en
  Vlan99; router-id fijado manualmente (`10.2.1.1` MLS1, `10.2.1.2` MLS2).
- Adyacencia OSPF MLS1-MLS2 verificada en estado `FULL/-` sobre Vlan99.
- Interfaz Gi1/0 (uplink a BR-VALPO) preparada con IP y `network` OSPF en
  ambos switches — dual-homed (`10.255.2.0/30` MLS1, `10.255.2.4/30`
  MLS2) — sin adyacencia real todavía (decisión B; BR-VALPO se integra en
  Bloque 5).
- Enhanced Object Tracking: objeto `track 1` vigilando line-protocol de
  Gi1/0, referenciado por ambos grupos HSRP con `decrement 10`.

## Evidencia capturada
- VLANs 40/41/999 activas en los tres switches (`show vlan brief`).
- `show spanning-tree vlan 40/41` — MLS1 root de VLAN 40, MLS2 root de
  VLAN 41; ALS1 con bloqueo alternado correcto por VLAN (Po2/Po3).
- `show etherchannel summary` — Po1/Po2/Po3 en `(SU)` LACP en los tres
  switches.
- `show port-security address` en ALS1 — MACs SecureSticky aprendidas en
  Gi1/1 (VLAN 40) y Gi1/2 (VLAN 41).
- `show ip interface brief` — SVIs 40/41/99 y Gi1/0 up/up en MLS1 y MLS2.
- `show standby brief` — MLS1 Active/40 + Standby/41, MLS2 Standby/40 +
  Active/41, convergidos.
- `show ip ospf neighbor` — adyacencia FULL/- entre MLS1 (10.2.1.1) y MLS2
  (10.2.1.2) sobre Vlan99, tipo P2P.
- `show track 1` — objeto trackeado por ambos grupos HSRP en los dos
  switches.
- Pings cruzados SVI-a-SVI y PC-a-VIP exitosos (primer paquete con
  timeout por ARP inicial, firma normal).

## Problemas / aprendizajes
- **`show spanning-tree vlan X` reportó "does not exist" antes de configurar los trunks.** No es un error: una instancia RPVST+ solo nace cuando la VLAN tiene al menos un puerto miembro en forwarding — el mismo principio que el autostate de las SVIs visto en el Bloque 3. Orden de dependencias, no fallo de configuración.
- **`show port-security address` salió vacío hasta que los PCs generaron tráfico** (un simple ping, aunque fallara por falta de gateway). La MAC sticky aprende con la primera trama recibida, no al aplicar la configuración.
- **`show ip interface brief | include Gi1/0` no devolvió nada.** El filtro es un grep literal y `"Gi1/0"` no es substring de `"GigabitEthernet1/0"`. Hay que filtrar por el nombre completo o correr el comando sin filtro. (Es la misma clase de falso negativo que reaparecería en el Bloque 8 con `include ip domain-lookup`: un filtro que no encuentra lo que busca reporta ausente algo que está.)
- **HSRP mostró estado transitorio `Speak` en VLAN41/MLS1 justo tras aplicar la config.** Mismo patrón de convergencia normal (hello 3 s / holdtime 10 s) ya visto en STGO; se resolvió solo a `Standby`.
- **`show track 1` reportó `line-protocol Up` pese a que Gi1/0 no tiene a BR-VALPO conectado.** Misma particularidad de EVE-NG/vIOS ya documentada en el Bloque 3. Pendiente de comportamiento real cuando BR-VALPO se cablee en el Bloque 5.
- **`no switchport` en Gi1/0 fue necesario** para convertirlo de puerto L2 a interfaz L3 ruteada antes de asignarle IP: el vIOS-L2 nace todo en modo switchport por defecto.
