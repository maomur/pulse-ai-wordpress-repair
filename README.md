# Pulse AI — WordPress Repair Skill

> **Limpia y repara sitios WordPress hackeados con un flujo profesional de 7 pasos, directamente desde tu agente de IA.**

---

## ¿Qué hace esta skill?

Esta skill convierte a Claude (u otro agente compatible) en un especialista en recuperación de sitios WordPress comprometidos. Cubre el proceso completo de remediación: desde la evaluación inicial del ataque hasta el endurecimiento post-limpieza y la solicitud de eliminación de listas negras.

**Flujo de uso:** le indicas al agente el **directorio infectado** de WordPress (y el dominio) y, opcionalmente, un **volcado de la base de datos** (`.sql` o `.sql.gz`); el agente **analiza los archivos y la base de datos**, aplica las correcciones y al final genera un **dashboard HTML autónomo** (un solo archivo, se abre con doble clic, sin internet) con **dos vistas**:

- **👤 Modo simple** — explicación clara para un usuario sin conocimientos técnicos (qué pasó, qué se hizo, qué debe hacer).
- **🔧 Modo técnico** — rutas exactas, tablas, comandos, CVEs y resultados de verificación.

El dashboard se imprime a PDF desde el navegador. La plantilla está en [`dashboard-template.html`](dashboard-template.html).

### Tipos de ataques que cubre

| Tipo de ataque | Síntomas |
|----------------|----------|
| **Redirect Hack** | Visitantes redirigidos a sitios de spam, farmacia o contenido adulto (a menudo solo en móvil o para Googlebot) |
| **Pharma / SEO Spam** | Google muestra palabras clave de medicamentos/casino en los resultados |
| **Backdoor** | El atacante recupera acceso después de cada limpieza |
| **Defacement** | La página de inicio fue reemplazada por el atacante |
| **Phishing** | Tu sitio sirve páginas falsas de inicio de sesión |
| **Admin Malicioso** | Usuarios administradores desconocidos en wp-admin |
| **Cryptominer** | CPU del servidor al 100%, sitio lento |
| **Spam Mailer** | Servidor en listas negras por envío de spam |
| **Supply-chain** | Muchos sitios infectados a la vez tras actualizar un plugin comprometido/vendido |

### Contexto de amenazas 2025–2026 (en qué se basa la skill)

- **Los plugins son la puerta de entrada nº1**: ~91% de las vulnerabilidades nuevas están en plugins, no en el core.
- **Explotación en ~5 horas**: ese es el tiempo medio entre que se publica una vulnerabilidad y su explotación activa. Un plugin desactualizado es un riesgo inmediato.
- **Persistencia moderna**: los backdoors ya no viven solo en `uploads/`. La skill busca también en `mu-plugins/`, *drop-ins* (`object-cache.php`, `db.php`…), tareas **WP-Cron** maliciosas, inyecciones en `wp-config.php`, `auto_prepend_file` y admins/contraseñas de aplicación ocultos.
- **La base de datos es un objetivo de primera**: `wp_options` es el escondite nº1 (blobs base64 bajo nombres hexadecimales, abuso de `autoload`, *transients* y `cron` con payloads), además de spam SEO en `wp_posts`, admins ocultos en `wp_usermeta` y PHP guardado por plugins de snippets. La skill analiza la BD en vivo **o un volcado `.sql` independiente**, y edita datos serializados de forma segura (`wp search-replace`, no `REPLACE()` a ciegas).
- **Ataques de cadena de suministro**: plugins legítimos que son vendidos o hackeados y distribuyen malware en una actualización rutinaria.
- **Una actualización NO es una limpieza**: parchear cierra el agujero pero no elimina el código ya inyectado.

---

## Flujo de trabajo — 7 pasos

```
Paso 0 → Triaje y autorización      Confirmar permiso/acceso, ruta de WordPress, WP-CLI
Paso 1 → Evaluación inicial         Tipo de hack, listas negras, WPScan, punto de entrada
Paso 2 → Backup & Aislamiento       Backup forense, modo mantenimiento, rotar credenciales y salts
Paso 3 → Escaneo de malware         Checksums, mu-plugins, drop-ins, cron, wp-config, admins falsos
Paso 3B→ Forense de base de datos    Analiza dump .sql o BD en vivo: wp_options/autoload, cron, transients, usuarios ocultos, spam
Paso 4 → Limpieza                   Reinstalar core, eliminar malware Y mecanismos de persistencia
Paso 5 → Auditoría de plugins       Cerrar el punto de entrada: reinstalar, actualizar, virtual patching
Paso 6 → Endurecimiento             Permisos, salts, XML-RPC, REST API, registro, 2FA, WAF
Paso 7 → Verificación y monitoreo   Re-escanear, test (Googlebot), blacklist, re-escaneo a +30 días
```

> **Regla de oro:** si el sitio se reinfecta, quedó un mecanismo de persistencia. La limpieza no termina hasta que el sitio sobrevive **30 días** sin reinfectarse.

---

## Instalación

### Requisitos

- [Claude Code](https://claude.ai/code) u otro agente compatible con el ecosistema de skills
- Node.js 18+ (para el comando `npx`)

### Instalar en un proyecto específico

```bash
npx skills add maomur/pulse-ai-wordpress-repair
```

Esto instala la skill en `.agents/skills/` dentro del directorio de tu proyecto actual.

### Instalar globalmente (todos tus proyectos)

```bash
npx skills add maomur/pulse-ai-wordpress-repair -g
```

Con la instalación global, la skill estará disponible en cualquier proyecto sin necesidad de reinstalarla.

### Instalación manual

Si no tienes el CLI de skills, puedes copiar el archivo directamente:

```bash
# Clonar el repositorio
git clone https://github.com/maomur/pulse-ai-wordpress-repair.git

# Copiar al directorio de skills de tu proyecto
mkdir -p tu-proyecto/.agents/skills/pulse-ai-wordpress-repair
cp pulse-ai-wordpress-repair/SKILL.md tu-proyecto/.agents/skills/pulse-ai-wordpress-repair/SKILL.md
```

O para instalación global:

```bash
mkdir -p ~/.agents/skills/pulse-ai-wordpress-repair
cp pulse-ai-wordpress-repair/SKILL.md ~/.agents/skills/pulse-ai-wordpress-repair/SKILL.md
```

---

## Uso

Una vez instalada, la skill se activa automáticamente cuando describes un problema relacionado con WordPress comprometido. Ejemplos de frases que la activan:

- *"Mi sitio WordPress fue hackeado"*
- *"Tengo malware en WordPress"*
- *"WordPress está redirigiendo a sitios de spam"*
- *"Google marcó mi sitio como peligroso"*
- *"Hay usuarios administradores desconocidos en mi wp-admin"*
- *"My WordPress site got hacked"*
- *"WordPress malware cleanup"*

También puedes invocarla directamente:

```
/pulse-ai-wordpress-repair
```

---

## Herramientas que utiliza la skill

La skill guía al agente para usar las siguientes herramientas durante la remediación:

- **WP-CLI** — Gestión de WordPress desde línea de comandos
- **WPScan** — Detección de versiones y vulnerabilidades conocidas (CVE)
- **find / grep** — Búsqueda de archivos y patrones maliciosos (incluyendo ofuscación)
- **curl** — Verificación del estado del sitio (incluido test con user-agent de Googlebot para detectar redirects encubiertos)
- **diff** — Comparación contra copias limpias conocidas
- **git** — Control de versiones para rastrear cambios
- Escáners y bases de datos externas: **Sucuri SiteCheck**, **Google Safe Browsing**, **VirusTotal**, **WPScan DB**, **Patchstack**, **Wordfence Threat Intel**

---

## Comandos WP-CLI incluidos

La skill incluye referencia completa de comandos para:

```bash
wp core verify-checksums       # Verificar integridad del core
wp core download --force       # Reinstalar core sin afectar wp-content
wp plugin verify-checksums     # Verificar integridad de plugins
wp user list --role=admin      # Listar administradores
wp db export / import          # Backup y restauración de base de datos
wp maintenance-mode activate   # Activar modo mantenimiento
```

---

## Informe final de incidente

Al terminar la remediación, la skill genera automáticamente un **informe profesional completo** con todos los detalles reales de la sesión:

| Sección | Contenido |
|---------|-----------|
| **Resumen ejecutivo** | Qué pasó, cuándo, tipo de ataque, impacto y estado final |
| **Análisis de infección** | Tipo de ataque, punto de entrada, archivos y tablas afectadas, estado en listas negras |
| **Archivos eliminados** | Tabla con ruta, tipo de malware encontrado y acción tomada |
| **Limpieza de base de datos** | Tabla con cambios realizados por tabla y campo |
| **Usuarios** | Usuarios maliciosos eliminados, contraseñas rotadas |
| **Plugins y temas** | Lista de reinstalados, eliminados, actualizados, nulled encontrado |
| **Credenciales rotadas** | Checklist de todas las contraseñas cambiadas |
| **Endurecimiento aplicado** | Tabla con cada medida de seguridad y su estado |
| **Verificación final** | Resultado del re-escaneo, checksums, test de carga |
| **Estado en listas negras** | Google Safe Browsing, Sucuri, fecha de solicitud de remoción |
| **Causa raíz** | Análisis del punto de entrada y recomendaciones específicas |
| **Línea de tiempo** | Cronología completa del incidente y la remediación |

El informe es lo suficientemente detallado para que otro técnico entienda exactamente qué ocurrió y qué se hizo, sin necesidad de preguntas adicionales.

---

## Checklists incluidas

- Pre-limpieza (backup y evaluación)
- Durante la limpieza (archivos, base de datos, usuarios)
- Post-limpieza (verificación y endurecimiento)
- Solicitud de remoción de listas negras (Google, Sucuri)

---

## Compatibilidad

| Plataforma | Compatible |
|------------|-----------|
| Claude Code | ✅ |
| Claude (claude.ai) | ✅ |
| ChatGPT | ✅ |
| Gemini | ✅ |
| Otros agentes con soporte de skills | ✅ |

---

## Estructura del repositorio

```
pulse-ai-wordpress-repair/
├── SKILL.md                 # Skill principal (flujo completo de remediación)
├── dashboard-template.html  # Plantilla del dashboard de reporte (modo simple + técnico)
└── README.md                # Este archivo
```

---

## Autor

Desarrollado por **[Pulse AI](https://github.com/maomur)** como parte del ecosistema de skills para agentes de IA.

---

## Licencia

MIT — Libre para usar, modificar y distribuir.
