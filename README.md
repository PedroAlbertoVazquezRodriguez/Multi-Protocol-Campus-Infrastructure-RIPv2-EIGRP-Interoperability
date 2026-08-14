# Red de Campus Heterogénea — Interoperabilidad RIPv2 / EIGRP

Proyecto Integrador de Aprendizaje (PIA) que diseña, configura y valida una red de campus multi-sede en **Cisco Packet Tracer**, interconectando seis instituciones educativas mediante dos protocolos de enrutamiento distintos (RIPv2 y EIGRP) redistribuidos en un núcleo común, con telefonía IP, servicios de red y hardening de seguridad.

Proyecto académico — **FIME UANL**, Redes de Telecomunicaciones, Grupo 003 / Equipo 05, Enero–Junio 2026.

---

## Contenido del repositorio

| Archivo | Descripción |
|---|---|
| `Multi-Protocol_Campus_Infrastructure_RIPv2___EIGRP_Interoperability.pdf` | Presentación técnica completa del proyecto |
| `Multi-Protocol_Campus_Infrastructure_RIPv2___EIGRP_Interoperability.pptx` | Presentación editable (PowerPoint) |

## Resumen del proyecto

El entorno interconecta seis sedes académicas — **FIME, UANL, UDEM, TEC, CECATI y CONALEP** — con un Proveedor de Servicios de Internet (ISP), sobre una topología jerárquica que combina dos dominios de enrutamiento distintos, unidos por un núcleo de redistribución cruzada (router FIME, actuando como ASBR).

**Objetivo técnico**: demostrar la convergencia de protocolos disímiles (RIPv2 y EIGRP) asegurando la entrega de servicios vitales — VoIP, DNS, HTTP, FTP y correo electrónico — bajo un esquema de seguridad perimetral.

### Cifras del proyecto

| Métrica | Valor |
|---|---|
| Instituciones interconectadas | 6 |
| LANs segmentadas con VLSM | 17 |
| Routers troncales + 1 ISP | 4 |
| Extensiones VoIP (SCCP) gestionadas | 20 |
| Escenarios de prueba validados | 8 / 8 |

## Arquitectura de red

Topología jerárquica de dos capas:

- **Capa superior — dominio RIPv2**: TEC, UANL, UDEM.
- **Capa inferior — dominio EIGRP**: CECATI, CONALEP y el núcleo.
- **Núcleo (Router FIME)**: actúa como **ASBR** (Autonomous System Boundary Router), uniendo los dominios RIP, EIGRP, rutas estáticas y el enlace al PBX de telefonía. Es el nodo con mayor densidad de conexiones de la topología (6 interfaces activas).
- **Router ISP (IR829GW)**: frontera exterior con interfaz celular simulada (`Cellular0`), separando la red interna del internet público.

## Ingeniería de tráfico y VLSM

Direccionamiento de longitud variable calculado con la fórmula `H = 2ⁿ − 2`:

| Sede / Función | Base | Máscara | Rango utilizable |
|---|---|---|---|
| UDEM / UANL (LAN VII–X) | 172.16.16.0 → .64.0 | /20 | .1 – .4094 |
| TEC (LAN XI–XII) | 172.16.82.0 → .84.0 | /23 | .1 – .510 |
| CECATI (LAN XIII–XIV) | 172.16.87.0 → .128 | /25 | .1 – .126 |
| CONALEP (LAN XV–XVI) | 172.16.90.0 → .32 | /27 | .1 – .30 |
| FIME / Ed4 / Ed7 (LAN I–VI) | 192.168.100.16 → .96 | /28 | .1 – .14 |
| WAN Inter-Router | 192.168.100.4 → .128 | /30 | .1 – .2 |

## Protocolos de enrutamiento

### RIPv2 — capa superior (TEC / UDEM / UANL)

- Vector-distancia, classless.
- Métrica: conteo de saltos (máx. 15).
- Multicast: `224.0.0.9`.
- Se usa v2 (y no v1) porque transporta la máscara de subred, requisito para VLSM.

```
router rip
 version 2
 network 172.16.0.0
 network 192.168.100.0
 no auto-summary
```

### EIGRP — segmento inferior (CECATI / CONALEP / núcleo)

- Híbrido avanzado (algoritmo DUAL).
- Métrica: ancho de banda + retraso, ponderados.
- Rutas libres de bucles, convergencia casi instantánea.
- Mantiene tabla de topología y sucesores factibles.

```
Métrica EIGRP = [ (10⁷ / BW mín kbps) + Σ retrasos(10µs) ] × 256
```

### Redistribución en el núcleo (Router FIME — ASBR)

```
router eigrp 100
 redistribute rip
 redistribute static metric 10000 100 255 1 1500
 network 192.168.100.0
 network 172.16.0.0
 network 10.10.10.0 0.0.0.3   ! Anuncio de la red del PBX
ip route 0.0.0.0 0.0.0.0 192.168.100.0
```

CONALEP redistribuye de vuelta hacia EIGRP con una ruta estática predeterminada de respaldo, blindando la convergencia ante fallos:

```
router eigrp 100
 redistribute rip metric 10000 0 255 1 1500
 redistribute static metric 10000 100 255 1 1500
 network 192.168.100.0
 network 172.16.90.0 0.0.0.31
 no auto-summary
ip route 0.0.0.0 0.0.0.0 192.168.100.130
```

## Telefonía IP — arquitectura SCCP

Router **CME** (Cisco Call Manager Express) centralizado en `10.10.10.2`, gestionando hasta **20 extensiones** con teléfonos Cisco 7960.

- Señalización: **SCCP (Skinny Client Control Protocol)** sobre **TCP 2000**.
- Cada teléfono se valida estáticamente emparejando su dirección MAC con una extensión del bloque `1000–1019`.
- Aprovisionamiento vía **DHCP option 150**, apuntando al servidor TFTP del propio CME.

```
ip dhcp pool VOZ_LAN_TEC1
 network 172.16.82.0 255.255.254.0
 default-router 172.16.83.254
 option 150 ip 10.10.10.2   ! TFTP → Router CME
```

## Seguridad — hardening de dispositivos

Cuatro capas de protección aplicadas de forma uniforme en todos los routers:

| # | Medida | Descripción |
|---|---|---|
| 1 | Cifrado MD5 | `enable secret 5 ...` protege el modo privilegiado con hash de Nivel 5 |
| 2 | Auditoría ciega | `service password-encryption` camufla contraseñas heredadas en consolas |
| 3 | Seguridad de puertos VTY | Líneas `line vty 0 4` exigen credenciales para Telnet/SSH remoto |
| 4 | Disuasión legal (MOTD) | Banner unificado de acceso prohibido previo al login |

## Bitácora de pruebas — 8 / 8 escenarios validados

| # | Prueba | Capa validada |
|---|---|---|
| 1 | Ping PC LAN VII → Servidor LAN XII | L3 — Red |
| 2 | Tracert Laptop LAN I → Impresora Producción | L3 — Red |
| 3 | Traceroute Router Ed4 → Laptop LAN VIII (enrutamiento híbrido) | L3 — Red |
| 4 | FTP PC LAN VII → Servidor LAN XI | L7 — Aplicación |
| 5 | Telnet PC LAN X → Router CONALEP (cruce de todos los troncales) | L4/L7 |
| 6 | Servicio Web desde Laptop LAN XV (DNS + HTTP) | L7 — Aplicación |
| 7 | Servicio Email (SMTP push / POP3 sync) | L7 — Aplicación |
| 8 | Convergencia VoIP — RTP entre teléfonos (SCCP + CME) | L7 — VoIP |

Todos los caminos de prueba cruzan al menos un punto de redistribución RIPv2 ↔ EIGRP, validando la interoperabilidad extremo a extremo.

## Conclusión

El proyecto demuestra que dos protocolos de enrutamiento disímiles — RIPv2 y EIGRP — pueden coexistir en una misma topología de campus cuando la redistribución y las rutas estáticas se administran con disciplina. El núcleo FIME actúa como ASBR, garantizando entrega extremo a extremo de servicios críticos (datos, voz, web y correo) bajo un perímetro de hardening uniforme.

## Equipo 05 — Grupo 003

| Integrante |
|---|
| Dante O. Rodríguez |
| Emilio O. Ramírez |
| Gerardo D. Lozano |
| Pedro A. Vázquez |
| Oscar Viniegra |

**Profesora**: Ing. Guadalupe Pineda Acha
**Curso**: Redes de Telecomunicaciones — FIME UANL
**Fecha**: 25 de mayo de 2026

## Cómo ver el proyecto

- Abre directamente el `.pdf` en GitHub para ver la presentación completa sin descargar nada.
- El `.pptx` está disponible para quien necesite editar o reutilizar las diapositivas.

---

*Proyecto académico con fines educativos — FIME UANL 2026.*
