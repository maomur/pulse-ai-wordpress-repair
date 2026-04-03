# Pulse AI — WordPress Repair Skill

> **Limpia y repara sitios WordPress hackeados con un flujo profesional de 7 pasos, directamente desde tu agente de IA.**

---

## ¿Qué hace esta skill?

Esta skill convierte a Claude (u otro agente compatible) en un especialista en recuperación de sitios WordPress comprometidos. Cubre el proceso completo de remediación: desde la evaluación inicial del ataque hasta el endurecimiento post-limpieza y la solicitud de eliminación de listas negras.

### Tipos de ataques que cubre

| Tipo de ataque | Síntomas |
|----------------|----------|
| **Redirect Hack** | Visitantes redirigidos a sitios de spam, farmacia o contenido adulto |
| **Pharma Hack** | Google muestra palabras clave de medicamentos en los resultados |
| **Backdoor** | El atacante recupera acceso después de cada limpieza |
| **Defacement** | La página de inicio fue reemplazada por el atacante |
| **Phishing** | Tu sitio sirve páginas falsas de inicio de sesión |
| **Admin Malicioso** | Usuarios administradores desconocidos en wp-admin |
| **Cryptominer** | CPU del servidor al 100%, sitio lento |
| **Spam Mailer** | Servidor en listas negras por envío de spam |

---

## Flujo de trabajo — 7 pasos

```
Paso 1 → Evaluación inicial         Identificar tipo de hack, verificar listas negras
Paso 2 → Backup & Aislamiento       Backup completo, modo mantenimiento, rotar credenciales
Paso 3 → Escaneo de malware         WP-CLI checksums, archivos PHP sospechosos, base de datos
Paso 4 → Limpieza                   Reinstalar core, eliminar archivos y código malicioso
Paso 5 → Auditoría de plugins       Desactivar, reinstalar desde fuentes oficiales, actualizar
Paso 6 → Endurecimiento             Permisos, secret keys, deshabilitar XML-RPC, WAF
Paso 7 → Verificación y monitoreo   Re-escanear, test final, solicitar remoción de blacklist
```

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
- **find / grep** — Búsqueda de archivos y patrones maliciosos
- **curl** — Verificación del estado del sitio
- **git** — Control de versiones para rastrear cambios
- Escaners externos: **Sucuri SiteCheck**, **Google Safe Browsing**, **VirusTotal**

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
├── SKILL.md       # Skill principal (flujo completo de remediación)
└── README.md      # Este archivo
```

---

## Autor

Desarrollado por **[Pulse AI](https://github.com/maomur)** como parte del ecosistema de skills para agentes de IA.

---

## Licencia

MIT — Libre para usar, modificar y distribuir.
