# Guía Completa de Preguntas de Copier — doodba-copier-template

## Comando inicial

```bash
copier copy gh:Halltic/doodba-copier-template /ruta/a/tu/proyecto --trust
```

- `gh:Halltic/doodba-copier-template` → fork de Halltic del template de Tecnativa
- `/ruta/a/tu/proyecto` → carpeta destino donde se genera el proyecto (ej: `/opt/odoo/prueba`)
- `--trust` → necesario porque el template ejecuta tareas post-copia (`invoke after-update` + `invoke develop`). Sin `--trust`, Copier te avisa y no las ejecuta

Para actualizar un proyecto existente con nuevas versiones del template:
```bash
cd /ruta/a/tu/proyecto
copier update --trust
```

Cada pregunta alimenta las plantillas Jinja2 que generan los Docker Compose y configuraciones del proyecto. Las respuestas se guardan en `.copier-answers.yml`.

---

## Preguntas en orden (26 total)

---

### 1. `odoo_version` — Versión de Odoo

**Pregunta:** "On which Odoo version is it based?"
**Valores posibles:** 12.0, 13.0, 14.0, 15.0, 16.0, 17.0, 18.0
**Tú pusiste:** `18.0`

**Qué afecta:**
- La imagen Docker base que se descarga (`tecnativa/doodba:18.0`)
- Los puertos de desarrollo: `localhost:{ODOO_MAJOR}069` → para Odoo 18 es `localhost:18069`
- Qué linters y herramientas se activan (ruff en vez de flake8 para versiones modernas)
- Compatibilidad con dependencias (`jingtrang` solo desde Odoo 13, `pathlib` solo para < 11)
- Opciones de `--dev` en el comando de arranque

**Recomendación:** Siempre la versión que use el cliente. En proyectos nuevos, la última estable.

---

### 2. `odoo_proxy` — Reverse proxy

**Pregunta:** "Which proxy will you use to deploy Odoo?"
**Valores posibles:** `traefik`, `none`
**Tú pusiste:** `Traefik`

**Qué afecta:**
- **Traefik:** Genera labels en Docker Compose para routing automático, HTTPS con Let's Encrypt, redirecciones entre dominios. Es lo que usan en producción para exponer Odoo al exterior.
- **None:** Sin proxy. Odoo se expone directamente. Solo para desarrollo local o detrás de otro proxy ya existente (nginx, Apache).

**Dev vs Prod:**
- **Dev:** No usa Traefik, accedes directamente por `localhost:18069`
- **Test/Prod:** Traefik gestiona dominios, TLS y routing

**Recomendación:** Siempre `traefik` salvo que la empresa use otro proxy.

---

### 3. `traefik_version` — Versión de Traefik

**Pregunta:** "Indicate Traefik version"
**Valores posibles:** `v1.0`, `v2.0`, `v3.0`
**Tú pusiste:** `v3.0`

**Qué afecta:**
- La sintaxis de labels de Docker cambia radicalmente entre v1, v2 y v3
- v3 soporta las features más modernas (middlewares, TCP routing)
- v1 está deprecado

**Recomendación:** `v3.0` para proyectos nuevos. Solo v2 si el servidor de producción ya tiene Traefik 2 corriendo.

---

### 4. `odoo_initial_lang` — Idioma inicial

**Pregunta:** "If you want to initialize Odoo automatically in a specific language..."
**Formato:** `ll_CC` (código de idioma + país)
**Tú pusiste:** `es_ES`

**Qué afecta:**
- Al crear la base de datos, Odoo carga automáticamente las traducciones de ese idioma
- Se establece como variable `INITIAL_LANG` en los Docker Compose

**Recomendación:** `es_ES` para proyectos españoles. Vacío si quieres elegir luego.

---

### 5. `odoo_admin_password` — Contraseña maestra de Odoo

**Pregunta:** "What will be your Odoo admin password?"
**Tú pusiste:** (oculto, 62 caracteres)

**Qué afecta:**
- Es la contraseña maestra que protege la gestión de bases de datos (`/web/database/manager`)
- Permite crear, duplicar, eliminar y restaurar bases de datos
- **CRÍTICA en producción**: si alguien la obtiene, puede borrar tu base de datos

**Dev vs Prod:**
- **Dev:** Puede ser simple, es tu máquina local
- **Prod:** Debe ser larga y aleatoria (64+ caracteres). Genera una con `openssl rand -base64 48`

**Recomendación:** Siempre contraseña fuerte. Se guarda en `.docker/odoo.env` (gitignored en producción).

---

### 6. `odoo_listdb` — Listar bases de datos públicamente (producción)

**Pregunta:** "Do you want to list databases publicly in the production environment?"
**Valores:** `Yes` / `No`
**Tú pusiste:** `Yes`

**Qué afecta:**
- Si `Yes`: cualquiera que acceda a tu Odoo puede ver qué bases de datos existen
- Si `No`: hay que escribir el nombre exacto de la BD para acceder

**Dev vs Prod:**
- **Dev:** `Yes` — necesitas ver y cambiar entre BDs fácilmente
- **Prod:** **`No`** — por seguridad, no expones nombres de BD al público

**Recomendación:** `No` en producción siempre. `Yes` en dev/test.

---

### 7. `odoo_listdb_test` — Listar bases de datos (staging)

**Pregunta:** "Do you want to list databases publicly in the staging environment?"
**Tú pusiste:** `Yes`

**Recomendación:** `Yes` para staging, facilita el testing.

---

### 8. `odoo_oci_image` — Registro de imágenes Docker

**Pregunta:** "If you are using an OCI/Docker image registry..."
**Ejemplo:** `docker.io/myteam/example-odoo`
**Tú pusiste:** (vacío)

**Qué afecta:**
- Si lo rellenas, el proyecto se configura para hacer `docker push` a ese registro
- Útil cuando dev construye la imagen y prod la descarga del registro (CI/CD)
- Vacío = builds locales en cada entorno

**Cuándo usarlo:**
- Equipos con CI/CD: GitHub Actions construye la imagen → la sube a Docker Hub/GHCR → producción hace `docker pull`
- Sin CI/CD: cada entorno construye su propia imagen localmente

---

### 9. `project_author` — Autor del proyecto

**Pregunta:** "Tell me who you are."
**Tú pusiste:** `Halltic Tech S.L.`

**Qué afecta:**
- Pylint verifica que los módulos privados incluyan este autor en su `__manifest__.py`
- Aparece en el `README.md` generado
- Si tu módulo tiene `author: "Pablo"` en vez de `"Halltic Tech S.L."`, pylint dará warning

---

### 10. `project_name` — Nombre del proyecto

**Pregunta:** "What's your project name?"
**Restricción:** Solo `A-Za-z0-9-_`, sin puntos ni espacios
**Tú pusiste:** `prueba`

**Qué afecta:**
- Prefijo de los contenedores Docker: `prueba-odoo-1`, `prueba-db-1`
- Nombre de las redes Docker
- Se usa en labels de Traefik para identificar el proyecto
- Nombre del workspace de VSCode

**Recomendación:** Nombre del cliente o proyecto, ej: `cliente-ecommerce`, `halltic-erp`.

---

### 11. `project_license` — Licencia

**Pregunta:** "What's your project's license?"
**Valores posibles:** BSL-1.0, Apache-2.0, GPL-3.0, LGPL-3.0, AGPL-3.0, OPL-1.0, no_license
**Tú pusiste:** `Boost Software License 1.0`

**Qué afecta:**
- Genera el archivo `LICENSE` en la raíz del proyecto
- Configura el linter para verificar que los manifests de módulos usen esta licencia

**Recomendación:**
- Módulos propietarios de cliente → `OPL-1.0` o `no_license`
- Módulos open source / OCA → `LGPL-3` o `AGPL-3`
- El BSL es lo que usa el propio template, no es lo habitual para módulos Odoo

---

### 12. `gitlab_url` — URL de Gitlab

**Pregunta:** "If you host this project in Gitlab..."
**Tú pusiste:** (vacío)

**Qué afecta:**
- Si lo rellenas, genera configuración de CI/CD para Gitlab (`.gitlab-ci.yml`)
- Configura el registro de imágenes de Gitlab
- Vacío = no se genera nada de Gitlab

---

### 13. `domains_prod` — Dominios de producción

**Pregunta:** "Configure the production domains for this project."
**Formato:** YAML en una línea
**Tú pusiste:** `{}`

**Ejemplo real para producción:**
```yaml
- hosts: [www.micliente.com, micliente.com]
  redirect_to: www.micliente.com
  cert_resolver: letsencrypt
```

**Qué afecta:**
- Traefik crea reglas de routing para cada dominio
- Genera certificados TLS automáticamente con Let's Encrypt
- Configura redirecciones (ej: micliente.com → www.micliente.com)
- `cert_resolver: letsencrypt` → HTTPS automático
- `cert_resolver: true` → certificado autofirmado
- `cert_resolver: false` → sin TLS

**Opciones avanzadas:**
- `path_prefixes: [/shop]` → solo enruta ese path al contenedor
- `redirect_permanent: true` → 301 en vez de 302
- `entrypoints: [websecure]` → solo escucha en ciertos entrypoints de Traefik

**Dev vs Prod:**
- **Dev:** No se usa, accedes por localhost
- **Prod:** **Obligatorio** rellenarlo con los dominios reales del cliente

---

### 14. `domains_test` — Dominios de staging

**Pregunta:** "Configure the test domains for this project."
**Formato:** Igual que `domains_prod`
**Tú pusiste:** `{}`

**Ejemplo:**
```yaml
- hosts: [prueba.staging.halltic.com]
  cert_resolver: letsencrypt
```

**Dev vs Test:**
- **Dev:** No se usa
- **Test:** Dominio interno del equipo para validar antes de producción

---

### 15. `paths_without_crawlers` — Rutas bloqueadas para bots

**Pregunta:** "Tell me the list of paths where you want to forbid crawlers."
**Tú pusiste:** `[/web, /website/info]`

**Qué afecta:**
- Traefik añade headers que impiden que Google/Bing indexen esas rutas
- `/web` = el backend de Odoo (no quieres que Google indexe tu panel de admin)
- `/website/info` = página de info del website de Odoo

**Recomendación:** Mínimo `/web`. Añade más si tienes rutas privadas.

---

### 16. `paths_with_crawlers` — Excepciones al bloqueo de bots

**Pregunta:** "Tell me the list of paths where you want to allow crawlers..."
**Tú pusiste:** `[/web/image/website]`

**Qué afecta:**
- Dentro de las rutas bloqueadas, permite bots en estas subrutas
- `/web/image/website` = imágenes públicas del website que sí quieres indexar

---

### 17. `cidr_whitelist` — Whitelist de IPs

**Pregunta:** "If you need to whitelist certain CIDR..."
**Tú pusiste:** (vacío)

**Qué afecta:**
- Solo las IPs listadas pueden acceder a tu Odoo
- Formato CIDR: `["1.2.3.4/32", "10.0.0.0/8"]`
- Solo funciona con Traefik 2+

**Cuándo usarlo:**
- Odoo de uso interno → whitelist la IP de la oficina
- Odoo público (ecommerce) → vacío

---

### 18. `postgres_version` — Versión de PostgreSQL

**Pregunta:** "Which PostgreSQL version do you want to deploy?"
**Valores:** 9.6, 10, 11, 12, 13, 14, 15, 16, 17
**Tú pusiste:** `17`

**Qué afecta:**
- Imagen de PostgreSQL en Docker: `postgres:17-alpine`
- PostgreSQL 9.6 **no tiene soporte de backups** en este template
- Versiones modernas (15+) mejor rendimiento con JSONB y particiones

**Recomendación:** La última estable (17) para proyectos nuevos. Mantener la del cliente si es migración.

---

### 19. `postgres_username` — Usuario de PostgreSQL

**Pregunta:** "Which user name will be used to connect to the postgres server?"
**Tú pusiste:** `odoo`

**Qué afecta:**
- Variable `PGUSER` en todos los entornos
- El usuario que Odoo usa para conectar a la BD

**Recomendación:** `odoo` es el estándar. No hay razón para cambiarlo.

---

### 20. `postgres_dbname` — Nombre de la base de datos

**Pregunta:** "What is going to be the main database name?"
**Tú pusiste:** `prueba`

**Qué afecta:**
- Variable `PGDATABASE` en todos los entornos
- El nombre de la BD que Odoo crea/usa
- El `db_filter` la usa para restringir acceso

**Dev vs Prod:**
- **Dev:** Puede ser cualquier cosa (`prueba`, `test`)
- **Prod:** Suele ser `prod` o el nombre del cliente. Usar el mismo en todos los entornos facilita restaurar backups

---

### 21. `postgres_password` — Contraseña de PostgreSQL

**Pregunta:** "What will be your postgres user password?"
**Tú pusiste:** (oculto)

**Dev vs Prod:**
- **Dev:** Puede ser simple
- **Prod:** Fuerte y aleatoria. Se guarda en `.docker/db-access.env` (gitignored)

---

### 22. `postgres_exposed` — Exponer PostgreSQL

**Pregunta:** "Do you want to expose database service?"
**Valores:** `Yes` / `No`
**Tú pusiste:** `No`

**Qué afecta:**
- `Yes`: PostgreSQL accesible desde fuera del contenedor (útil para conectar con pgAdmin, DBeaver)
- `No`: Solo accesible desde dentro de la red Docker

**Dev vs Prod:**
- **Dev:** `Yes` si quieres usar herramientas de BD externas
- **Prod:** **Siempre `No`** — nunca expongas la BD en producción

---

### 23. `postgres_dbfilter` — Filtro de base de datos

**Pregunta:** "Set your Odoo db filter."
**Formato:** Expresión regular
**Tú pusiste:** `^prueba`

**Qué afecta:**
- Solo se aplica en producción
- Odoo solo mostrará/usará bases de datos que coincidan con esta regex
- `^prueba` = solo BDs que empiecen por "prueba"
- `.*` = todas las BDs (inseguro en producción)

**Recomendación:** `^nombre_exacto$` en producción para máxima seguridad.

---

### 24. `smtp_default_from` — Email remitente por defecto

**Pregunta:** "In case an email doesn't have a valid From: header..."
**Tú pusiste:** (vacío)

**Qué afecta:**
- Si un email de Odoo no tiene remitente válido, usa esta dirección
- Ejemplo: `noreply@micliente.com`

**Dev vs Prod:**
- **Dev:** No importa, los emails van a MailHog
- **Prod:** Debería ser un email real del dominio del cliente

---

### 25. `smtp_relay_host` — Servidor SMTP

**Pregunta:** "What is your SMTP host?"
**Tú pusiste:** (vacío)

**Qué afecta:**
- Si lo rellenas, Doodba configura un relay SMTP local que encola y reenvía emails
- No necesitas configurar `ir.mail_server` dentro de Odoo
- Ejemplo: `mail.halltic.com`, `smtp.gmail.com`

**Dev vs Prod:**
- **Dev:** Vacío. Todos los emails van a MailHog (fake SMTP)
- **Prod:** El servidor SMTP real del cliente. Si se deja vacío, hay que configurar el servidor de correo desde dentro de Odoo

---

### 26. `backup_dst` — Destino de backups

**Pregunta:** "Where should the backups be stored?"
**Formato:** URL de Duplicity
**Tú pusiste:** (vacío)

**Ejemplos:**
- S3: `boto3+s3://mi-bucket/backups/cliente`
- SFTP: `sftp://user@server/path`
- Local: `/mnt/backups` (no recomendado)

**Qué afecta:**
- Si lo rellenas, se crea un servicio `backup` en `prod.yaml` con Duplicity
- Hace backups automáticos diarios: dump SQL + filestore de Odoo
- Si está vacío, **no hay backups automáticos**

**Dev vs Prod:**
- **Dev:** Vacío. No necesitas backups en desarrollo
- **Prod:** **Obligatorio rellenarlo.** Sin backups en producción es jugar con fuego

---

## Resumen: Qué cambia entre entornos

| Configuración | Dev | Test | Prod |
|---|---|---|---|
| Docker Compose | `devel.yaml` | `test.yaml` | `prod.yaml` |
| Dominios | localhost:18069 | dominio staging | dominio real |
| HTTPS | No | Sí (Traefik) | Sí (Let's Encrypt) |
| SMTP | MailHog (fake) | MailHog (fake) | SMTP real |
| Red | Interna (configurable) | Interna + whitelist | Interna + whitelist |
| Backups | No | No | Sí (Duplicity) |
| Listar BDs | Sí | Sí | **No** |
| Exponer PostgreSQL | Opcional | No | **No** |
| Contraseñas | Simples | Medias | **Fuertes** |
| Debug (wdb/debugpy) | Sí | No | No |
| Hot-reload | Sí | No | No |
| Demo data | Sí | No (`WITHOUT_DEMO=all`) | No |

---

## Después de Copier: Qué se ejecuta automáticamente

Al final del `copier copy --trust` se ejecutan dos tareas automáticas:

1. **`invoke after-update`** → Genera archivos derivados, ajusta permisos
2. **`invoke develop`** → Inicializa Git, instala pre-commit hooks, y prepara el entorno de desarrollo

Por eso en tu terminal ves la inicialización del repositorio Git al final.
