# Arquitectura de Infraestructura y HomeLab (Clúster Híbrido Multi-Nodo)

## ¿Qué es esto?
Infraestructura auto-gestionada de 3 nodos orientada a la soberanía de datos, resiliencia operativa y práctica de arquitectura de sistemas (Alta Disponibilidad, Copias de Seguridad, Infraestructura como Código y Redes Privadas). El sistema combina cómputo de aplicaciones mediante contenedores con un almacenamiento central en ZFS y réplicas automáticas fuera de sitio (offsite DR).

---

## Topología de Red y Arquitectura

- **Nodo 1 (Aplicaciones):** Ejecuta los contenedores y el tráfico de descargas aislado por VPN (Mullvad).
- **Nodo 3 (NAS Central):** Guarda los datos en ZFS y comparte las carpetas con el Nodo 1 por NFS.
- **Alta Disponibilidad (DNS):** Una IP virtual compartida (192.168.1.12) con Keepalived conmuta el DNS automáticamente entre el Nodo 1 y el Nodo 3 si uno cae.
- **Red Privada (Tailscale):** Conecta de forma segura los nodos locales con el **Nodo 2 (Offsite en Cadaqués)** para enviar las copias de seguridad fuera de casa.

## Especificaciones de Hardware y Nodos

| Nodo | Hardware | Memoria RAM | Discos y Almacenamiento | Sistema Operativo | Función Principal |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **Nodo 1** | HP Elite Slice G2 (Intel i5-7500T) | 32 GB DDR4 | 1 TB NVMe | Debian 13 | Servidor de aplicaciones Docker y procesamiento de alertas |
| **Nodo 2** | Lenovo ThinkCentre M600 Tiny | 4 GB DDR3 | 1 TB HDD (ZFS) | Debian 13 | Destino remoto de copias de seguridad (ZFS Receive, ARC limitado a 1GB) |
| **Nodo 3** | AOOSTAR WTR PRO (AMD Ryzen 7 5825U) | 48 GB DDR4 | 2x NVMe M.2 500GB + 2x SSD 1TB + 2x HDD 2TB CMR | Debian 13 | NAS principal, almacenamiento ZFS y puerta de enlace de seguridad |

---

## Redes y Alta Disponibilidad (HA)

### DNS Redundante con Keepalived (VRRP)

Alta disponibilidad en la resolución DNS local mediante el protocolo VRRP en el sistema operativo host (Keepalived).

- **VIP compartida:** `192.168.1.12` entre Nodo 1 (MASTER, prioridad 150) y Nodo 3 (BACKUP, prioridad 100).
- **Comprobación de estado:** Un script en BASH (`check_dns.sh`) comprueba el servicio DNS (AdGuard Home) en el puerto 53. Si falla en el nodo activo, reduce la prioridad VRRP para migrar la IP virtual al otro nodo al instante.
- **Sincronización:** Replicación automática de reglas y registros DNS entre ambos nodos con `adguardhome-sync` cada 5 minutos.

### Seguridad Perimetral y Redes Privadas

- **Red Malla Privada:** Conexión remota segura entre todos los nodos y dispositivos mediante Tailscale (basado en WireGuard) usando certificados HTTPS y MagicDNS.
- **Túnel de Salida Aislado:** El tráfico de descargas en el Nodo 1 se fuerza a salir cifrado mediante un contenedor de Gluetun conectado a Mullvad VPN (nodo Suiza), incluyendo cortafuegos automático (kill-switch) si cae la VPN.
- **Mínima Exposición Externa:** Cierre estricto de puertos en el router. Solo hay un puerto redirigido para un servicio específico (servidor de Minecraft en Nodo 3). Toda la administración se realiza exclusivamente por la red local o vía Tailscale.

---

## Seguridad y Protección de Servidores

Protección en capas aplicada directamente sobre el cortafuegos del sistema operativo:

```text
[ Tráfico Entrante WAN ]
       │
       ▼
 [ Cortafuegos UFW (Nodo 3) ] ──> [ Motor CrowdSec (Logs) ]
       │                                  │
       ▼                                  ▼
 [ IP Permitida ]               [ IP en Lista Negra ]
       │                                  │
       ▼                                  ▼
 [ Conexión a Contenedor ]      [ Bloqueo Kernel (nftables) ]
```

### Aislamiento con UFW
Regla por defecto de denegar todo el tráfico entrante en el Nodo 3:
- Permite acceso completo desde la red local (`192.168.1.0/24`).
- Permite acceso SSH únicamente desde el rango de la red privada Tailscale (`100.64.0.0/10`).
- Permite la entrada al puerto expuesto para el servidor del juego.

### Detección y Filtrado Activo
- **Análisis de Registros con CrowdSec:** Motor de CrowdSec analizando los registros del sistema en tiempo real para detectar intentos de fuerza bruta o escaneos.
- **Bloqueo a Nivel de Kernel:** El plugin (bouncer) inyecta las IPs bloqueadas directamente en las tablas de nftables. Esto descarta las conexiones maliciosas a nivel de sistema antes de que consuman recursos en las aplicaciones.

---

## Almacenamiento y Copias de Seguridad (Estrategia 3-2-1)

El almacenamiento central está configurado en el Nodo 3 utilizando ZFS, lo que evita la corrupción silenciosa de datos mediante comprobaciones de integridad (checksums).

### Estructura de Pools ZFS

- **`fast_pool` (Mirror de SSDs 1TB):** Almacenamiento rápido y redundante para bases de datos (PostgreSQL), gestión de documentos (Paperless-ngx) y fotos (Immich).
- **`media_pool` (Discos HDD CMR):** Almacenamiento para archivos multimedia, compartido por NFSv4 al Nodo 1 (montado con `systemd.automount` en `/etc/fstab` para no bloquear el sistema si el NAS no responde).
- **`cold_backup` (Disco HDD Secundario):** Destino local para copias frías de respaldo.
- **`minepool` (NVMe Dedicado):** Pool aislado con cuota de disco asignada para evitar que un servidor de aplicaciones llene el espacio del resto del sistema.

### Sistema de Copias de Seguridad

El flujo de trabajo sigue la regla 3-2-1 (3 copias, 2 medios distintos, 1 fuera de casa):

- **Snapshots Locales (Sanoid):** Creación automática de instantáneas. En los datos críticos se guardan 10 snapshots diarios y 4 semanales de forma rotativa.
- **Copia Remota Offsite (Syncoid sobre Tailscale):** Tarea programada cada madrugada (03:00 AM) que envía de forma incremental y cifrada los snapshots de ZFS desde el Nodo 3 hacia el Nodo 2 (ubicado en otra vivienda en Cadaqués) usando la VPN privada.
- **Alertas de Salud del Hardware:** Un script revisa cada hora el estado de los discos (SMART) y la salud de los pools de ZFS (`zpool status`). Si detecta un fallo, envía un aviso mediante un webhook a n8n en el Nodo 1, que notifica inmediatamente por Telegram.

---

## Gestión de Servicios y Aplicaciones

Los servicios están estructurados con Docker Compose y gestionados mediante Dockge.

### Gestión de Identidad y Contraseñas
- **Vaultwarden:** Servidor de contraseñas (implementación ligera de Bitwarden). Funciona de forma totalmente aislada de internet, accesible solo por HTTPS desde la red privada de Tailscale y protegido con autenticación en dos factores (2FA).

### Procesamiento de Documentos
Sistema para digitalizar y organizar documentos:
- Sincronización de carpetas de documentos entre ordenadores y el NAS mediante Syncthing.
- Procesamiento automático de los PDFs con Paperless-ngx (lectura OCR, etiquetado y búsqueda por contenido) guardando los datos en la base de datos sobre el pool rápido ZFS.

### Otros Servicios Almacenados
- **Monitorización y Automatización:** n8n (Gestión de alertas) y Scrutiny (Salud de discos duros).
- **Herramientas:** Obsidian LiveSync (Sincronización cifrada de notas) y Stirling-PDF.
- **Servicios Multimedia:** Jellyfin (Transcodificación por hardware en la CPU), Navidrome y Deemix (Música), y el stack de descargas (*Arr) aislado tras la VPN Gluetun.
- **Servidores de Juegos:** Servidor de Minecraft alojado en el almacenamiento NVMe aislado, protegido con lista blanca, verificación de cuentas oficiales y bloqueo de IPs anómalas con CrowdSec.
