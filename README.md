# 🚀 Proyecto Final SIS313: Infraestructura de Noticias con CDN Simulado

> **Asignatura:** SIS313: Infraestructura, Plataformas Tecnológicas y Redes<br>
> **Semestre:** 1/2026<br>
> **Docente:** Ing. Marcelo Quispe Ortega

## 👥 Miembros del Equipo (Grupo Proyecto 17)

| Nombre Completo | Rol en el Proyecto | Contacto (GitHub/Email) |
| :--- | :--- | :--- |
| Danner Macuchapi Fernández | Servidor DNS + Firewall (VM1 · `192.168.43.220`) | [Usuario de GitHub] |
| Limbert Mamani Isla | Servidor Editorial — Node.js + PM2 (VM2 · `192.168.43.221`) | [Usuario de GitHub] |
| Fernando José Quispe Gardeazabal | CDN Edge — NGINX + Fail2ban (VM3 · `192.168.43.222`) | [@FerchoJQG](https://github.com/FerchoJQG) |
| Melany Helen Jurado Mamani | Administración + Monitoreo + Backups (VM4 · `192.168.43.223`) | [Usuario de GitHub] |

## 🎯 I. Objetivo del Proyecto

> **Objetivo:** Diseñar e implementar la infraestructura de backend para un portal de noticias que simule el funcionamiento de una red CDN (Content Delivery Network), separando la entrega de contenido estático (imágenes, HTML, CSS) de la generación de contenido dinámico (API REST), con resolución DNS interna por subdominios, protección activa ante intrusiones y respaldo automatizado del contenido editorial.

## 💡 II. Justificación e Importancia

> **Justificación:** En cualquier portal de noticias real, el servidor de aplicación se satura rápidamente si debe responder cada petición de imagen o archivo estático. Sin una capa de caché perimetral, el CPU del servidor de origen trabaja al máximo para servir recursos que no cambian, elevando la latencia del usuario. Este proyecto resuelve ese cuello de botella implementando una **Capa Perimetral CDN** mediante NGINX que descarga el tráfico estático del servidor de origen, mientras DNS BIND9 gestiona la resolución inteligente de subdominios. La seguridad (T11/T13) está cubierta con Fail2ban y UFW distribuido. La automatización (T14/T15) garantiza que los datos se respalden sin intervención humana, y que el administrador pueda operar toda la infraestructura desde un único punto de control, asegurando la **continuidad operacional** del portal.

## 🛠️ III. Tecnologías y Conceptos Implementados

### 3.1. Tecnologías Clave

* **BIND9:** Servidor DNS autoritativo primario. Gestiona los subdominios `www`, `static`, `api` y `admin` del dominio `noticias.local`, con zonas directa e inversa configuradas.
* **Node.js 20 + PM2:** Motor de backend dinámico. Sirve la API REST de noticias en JSON; PM2 garantiza disponibilidad continua con autostart vía systemd.
* **NGINX:** Proxy inverso y servidor de caché perimetral. Distingue peticiones estáticas (responde desde disco) de las dinámicas (reenvía a VM2).
* **Fail2ban:** Defensa activa. Monitorea los logs de NGINX en tiempo real y bloquea IPs que escaneen rutas administrativas.
* **UFW:** Firewall distribuido host-based. Cada VM abre únicamente los puertos estrictamente necesarios.
* **Bash + Cron:** Automatización de monitoreo, menú interactivo de administración y backups remotos programados cada 6 horas.
* **Oracle VirtualBox:** Plataforma de virtualización. 4 VMs en modo Adaptador Puente sobre la red `192.168.43.0/24`.

### 3.2. Conceptos de la Asignatura Puestos en Práctica

* ✅ **Proxy Inverso y Caché (T4):** NGINX en VM3 separa el tráfico estático (caché local) del dinámico (proxy → `VM2:3000`). Cabecera `X-Cache-Status` verificable.
* ✅ **Firewall y Políticas de Acceso (T6):** UFW activo en las 4 VMs con política `deny incoming` por defecto y reglas específicas por IP de origen.
* ✅ **DNS Primario con BIND9 (T7):** Zonas directa (`noticias.local`) e inversa (`43.168.192.in-addr.arpa`) operativas, con los subdominios `www`, `static`, `api` y `admin` resolviendo.
* ✅ **Despliegue de Aplicaciones (T8):** API REST con 4 rutas funcionales. PM2 con `startup systemd` y `pm2 save` configurados para persistencia.
* ✅ **Hardening Integral / SSH (T11):** Puerto `2222`, `PermitRootLogin no` y `PasswordAuthentication no` en las 4 VMs.
* ✅ **Detección de Intrusiones (T13):** Jail `nginx-botsearch` con filtro personalizado. Ban de 24 h tras 2 accesos a rutas prohibidas.
* ✅ **Automatización con Bash (T14):** Scripts `monitor.sh`, `menu.sh` (9 opciones) y `backup_remoto.sh` funcionales en VM4.
* ✅ **Backups Automatizados (T15):** Backup local en VM2 (cron diario 02:00) y backup remoto en VM4 (cron cada 6 h). Verificación de integridad con `tar -tzf`.

## 🌐 IV. Diseño de la Infraestructura y Topología

### 4.1. Diseño Esquemático

> ![Arquitectura de la red](Imagen1.jpg)

| VM/Host | Rol | IP Física | IP Virtual (si aplica) | Red Lógica | SO |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **VM1** (`vm1-dns`) | DNS BIND9 + Firewall UFW | 192.168.43.220 | N/A | `192.168.43.0/24` | Ubuntu Server 22.04 |
| **VM2** (`vm2-editorial`) | Node.js + PM2 (API REST) | 192.168.43.221 | N/A | `192.168.43.0/24` | Ubuntu Server 22.04 |
| **VM3** (`vm3-cache`) | NGINX Proxy + Fail2ban (CDN Edge) | 192.168.43.222 | N/A | `192.168.43.0/24` | Ubuntu Server 22.04 |
| **VM4** (`vm4-admin`) | Admin + Monitoreo + Backups | 192.168.43.223 | N/A | `192.168.43.0/24` | Ubuntu Server 22.04 |

### 4.2. Estrategia Adoptada

* **Red plana sin VLANs:** Se optó por adaptador puente sobre la red `192.168.43.0/24` para maximizar la velocidad de transferencia entre el CDN (VM3) y el Origen (VM2), simulando un enlace de baja latencia dentro de un mismo segmento. La segmentación lógica se implementa con reglas UFW a nivel de host.
* **Separación estático/dinámico:** NGINX evalúa la ruta de cada petición: si es `/static/*` o un recurso (`.css`, `.js`, imágenes), responde desde disco con caché de 7–30 días; si es `/api/*`, actúa como proxy hacia VM2 aplicando caché de 5 minutos para no sobrecargar el origen.
* **DNS como dominio interno:** En lugar de IPs directas, todos los subdominios resuelven a través de VM1-BIND9. Esto permite redirigir tráfico a otra VM modificando un solo registro DNS, sin tocar la configuración de NGINX.

## 📋 V. Guía de Implementación y Puesta en Marcha

### 5.1. Pre-requisitos
* 4 laptops conectadas al mismo punto de acceso WiFi o hotspot.
* Oracle VirtualBox instalado en cada laptop.
* Ubuntu Server 22.04 LTS como sistema operativo de las VMs.
* Adaptador de red en modo **Adaptador Puente** apuntando a la interfaz WiFi activa.
* Repositorio git clonado en cada VM.

### 5.2. Despliegue (Secuencia de Ejecución)
1.  **Red:** Configurar IPs estáticas en Netplan en cada VM. Verificar ping cruzado entre las 4 VMs.
2.  **DNS (VM1):** Instalar BIND9, crear zonas directa e inversa para `noticias.local`, verificar con `named-checkzone`.
3.  **API (VM2):** Instalar Node.js 20, crear la API Express con PM2, configurar autostart con `pm2 startup systemd`.
4.  **CDN (VM3):** Instalar NGINX, configurar `proxy_cache_path`, crear bloques de ubicación para estáticos y dinámicos, activar el sitio.
5.  **Seguridad:** Instalar Fail2ban en VM3 con jail personalizado, configurar UFW en las 4 VMs y aplicar hardening SSH en todas.
6.  **Admin (VM4):** Configurar acceso SSH sin contraseña a las otras VMs, desplegar scripts de monitoreo, menú y backup remoto.

### 5.3. Ficheros de Configuración Clave
* `VM2:/opt/noticias/api/app.js`: Servidor Node.js con las 4 rutas REST de noticias.
* `VM3:/etc/nginx/sites-available/noticias`: Configuración del proxy inverso con caché y bloqueo de rutas.
* `VM1:/etc/bind/zones/db.noticias.local`: Zona directa DNS con todos los subdominios.
* `VM3:/etc/fail2ban/filter.d/nginx-botsearch.conf`: Filtro personalizado para detectar escaneos.
* `VM4:/opt/admin/scripts/monitor.sh`: Verifica el estado de todas las VMs con colores.
* `VM4:/opt/admin/scripts/menu.sh`: Menú interactivo con 9 opciones de administración.
* `VM4:/opt/admin/scripts/backup_remoto.sh`: Copia y verifica el contenido editorial de VM2.

**Incluir además los archivos de configuración y software a utilizar dentro del proyecto y organizados en carpetas.**

> [Para mayor detalle sobre la implementación](https://github.com/FerchoJQG/cfvgbhjnk/blob/main/README.md)

## ⚠️ VI. Pruebas y Validación

| Prueba Realizada | Resultado Esperado | Resultado Obtenido |
| :--- | :--- | :--- |
| `dig @192.168.43.220 www.noticias.local +short` | Devuelve `192.168.43.222` | **[ÉXITO]** DNS resuelve correctamente hacia VM3 |
| `curl http://192.168.43.222/api/noticias` | JSON con lista de noticias del servidor VM2 | **[ÉXITO]** Proxy enruta a VM2 y devuelve noticias |
| `curl http://192.168.43.222/static/` | HTML estático sin contactar VM2 | **[ÉXITO]** Cabecera `X-Cache-Type: STATIC-LOCAL` confirmada |
| `curl http://192.168.43.222/admin` | HTTP 403 Forbidden | **[ÉXITO]** NGINX bloquea la ruta y Fail2ban registra el intento |
| Acceso SSH como root al puerto 22 | Conexión rechazada | **[ÉXITO]** `PermitRootLogin no` y puerto 2222 activos |
| `/opt/admin/scripts/backup_remoto.sh` | Archivo `.tar.gz` íntegro en `/opt/admin/backups/` | **[ÉXITO]** Backup transferido y verificado con `tar -tzf` |
| `pm2 stop noticias-api` → reinicio de VM2 | PM2 vuelve a levantar el proceso automáticamente | **[ÉXITO]** `pm2 startup systemd` + `pm2 save` funcionan |

## 📚 VII. Conclusiones y Lecciones Aprendidas

* **Escalabilidad comprobada:** La arquitectura con NGINX como punto de entrada permite agregar múltiples servidores de origen (VM2) sin que el usuario perciba cambios, ya que solo se modifica el bloque `upstream` de NGINX.
* **Seguridad en capas:** La combinación de Hardening SSH, UFW distribuido y Fail2ban demuestra que no se necesita hardware especializado para proteger una infraestructura. Cada capa actúa de forma independiente, por lo que si una falla, las otras siguen activas.
* **Red plana vs VLANs:** Inicialmente se intentó implementar VLANs para segmentar el tráfico, pero dado que cada VM reside en una laptop diferente conectada por adaptador puente, las VLANs no eran aplicables. La segmentación lógica con UFW resultó igualmente efectiva para controlar qué VM puede comunicarse con quién.
* **PM2 y persistencia:** Instalar PM2 sin ejecutar `pm2 startup systemd` y `pm2 save` hacía que el proceso muriera al reiniciar la VM. Esta secuencia de dos comandos es crítica y debe ejecutarse siempre tras el primer despliegue.
* **Netplan y nombres de interfaz:** El nombre de la interfaz de red (`enp0s3`, `eth0`, etc.) varía según la configuración de VirtualBox. Antes de editar Netplan, siempre verificar con `ip addr show` para no aplicar la configuración a la interfaz incorrecta.
