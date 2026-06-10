# Fox Engine — Seguridad de red y acceso remoto

Documentación de **mi contribución** al TFG [Fox Engine](https://github.com/MadMark-Fox/FoxEngine) (equipo Fox Hound · 2ASIR · IES Zaidín Vergeles): el diseño e implementación de la **capa de seguridad global** del proyecto — perímetro con pfSense, acceso remoto zero-trust con Tailscale y autenticación entre nodos.

> Fox Engine es una infraestructura empresarial simulada de 5 nodos sobre Proxmox, gobernada por una IA local que ejecuta comandos solo tras confirmación humana. El repositorio del equipo contiene el proyecto completo; aquí se detalla la parte de red y seguridad.

---

## Contexto y arquitectura

Toda la infraestructura corre virtualizada en Proxmox dentro de la red de un centro educativo, lo que impone dos restricciones de diseño:

1. **No se pueden abrir puertos** en el cortafuegos del centro.
2. El laboratorio debe poder ejecutar pruebas agresivas (chaos engineering, simulación de DDoS) **sin riesgo para la red del instituto**.

```
[ Internet / Red del centro ]
          │
    ┌─────▼──────┐
    │  pfSense   │  ← Firewall / Router + salida Tailscale
    │ (WAN/LAN)  │
    └─────┬──────┘
          │  LAN privada 192.168.1.0/24 (aislada)
    ┌─────┼──────────────┬─────────────┐
┌───▼───┐ ┌▼─────────┐ ┌─▼──────┐ ┌────▼─────────┐
│Mother │ │  Plant   │ │ Tanker │ │  Fox Engine  │
│ Base  │ │  .1.12   │ │  .1.11 │ │ (IA, mesh)   │
│ .1.10 │ └──────────┘ └────────┘ └──────────────┘
└───────┘
```

---

## 1. Segmentación: LAN privada aislada

- Red interna `192.168.1.0/24` completamente separada de la red del centro mediante pfSense como única puerta de entrada/salida.
- **DHCP estático** con reservas fijas: Mother Base `.10`, Tanker `.11`, Plant `.12`. Direccionamiento predecible para automatización, monitorización y reglas de firewall.
- El aislamiento permite la demo de **Ingeniería del Caos** (saturación de discos, simulación de DDoS) sin que el tráfico toque la red real del instituto.

## 2. pfSense: el perímetro

pfSense CE desplegado como VM dedicada (1 vCPU / 1 GB RAM) con interfaz WAN hacia la red del centro y LAN hacia la red privada del clúster.

- Control de todo el tráfico entrante y saliente de los nodos.
- Integración con la capa de IA del proyecto: ante la detección de un ataque (alerta de Prometheus/Alertmanager), Fox Engine propone la regla de bloqueo y, **tras validación humana**, se aplica en el firewall. Este flujo se demuestra en directo en el escenario de chaos engineering del proyecto.

<!-- TODO: añade aquí una tabla con las reglas principales de pfSense (interfaz, origen, destino, puerto, acción) y 1-2 capturas del panel -->
📌 *Pendiente: tabla de reglas y capturas del panel de pfSense.*

## 3. Tailscale: acceso remoto zero-trust

El problema: administrar el clúster desde fuera del centro **sin abrir un solo puerto**.

La solución: red mesh **Tailscale (WireGuard)** con NAT traversal mediante conexiones salientes:

- Todos los dispositivos del equipo se autentican contra la tailnet; el que no está en la red, no ve nada.
- El nodo de IA (Fox Engine) **no es visible desde internet**: solo es alcanzable desde dentro de la mesh.
- Administración completa del clúster desde cualquier red externa (4G, casa, etc.) como si se estuviera en la LAN.
- OpenVPN evaluado y mantenido como alternativa secundaria de acceso.

<!-- TODO: captura del panel de Tailscale con los nodos de la tailnet (oculta IPs/cuentas si hace falta) -->
📌 *Pendiente: captura de la tailnet.*

## 4. Autenticación entre nodos: SSH con ED25519

- Acceso entre todos los nodos mediante **claves asimétricas ED25519**, sin contraseñas.
- Necesario para que el orquestador ejecute comandos en cualquier nodo en milisegundos tras la confirmación humana, sin sacrificar seguridad por velocidad.

---

## Lecciones aprendidas

- Diseñar la red **antes** de desplegar servicios ahorra semanas: el direccionamiento estático y la segmentación inicial evitaron rehacer configuraciones después.
- Zero-trust con Tailscale resultó más simple y más seguro que la alternativa clásica (abrir puertos + VPN tradicional), especialmente en una red que no controlas.
- Un firewall no es solo bloquear: bien integrado con la monitorización, se convierte en parte activa de la respuesta a incidentes.

---

*Proyecto completo del equipo: [MadMark-Fox/FoxEngine](https://github.com/MadMark-Fox/FoxEngine) · Memoria del TFG disponible en ese repositorio (PDF).*
