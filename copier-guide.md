# Guía Doodba Completa — Halltic Tech

> **Audiencia:** Devs junior que se incorporan a Halltic Tech S.L.  
> **Template:** `gh:Halltic/doodba-copier-template` (fork de Tecnativa)  
> **Stack:** Odoo 17 / 18 · Docker + Doodba · Traefik · PostgreSQL

---

## Índice

1. [Requisitos previos](#1-requisitos-previos)
2. [Copier: comando inicial y actualización](#2-copier-comando-inicial-y-actualización)
3. [Perfiles de proyecto Halltic](#3-perfiles-de-proyecto-halltic)
4. [Las preguntas de Copier](#4-las-preguntas-de-copier)
5. [Archivos generados tras Copier](#5-archivos-generados-tras-copier)
   - [5b. Formato de repos.yaml y addons.yaml](#5b-formato-de-reposyaml-y-addonsyaml)
6. [Comandos Invoke — referencia completa](#6-comandos-invoke--referencia-completa)
7. [Flujos de trabajo](#7-flujos-de-trabajo)
8. [Resumen: qué cambia entre entornos](#8-resumen-qué-cambia-entre-entornos)
9. [Errores frecuentes de juniors](#9-errores-frecuentes-de-juniors)
10. [Integración con VSCode](#10-integración-con-vscode)
11. [Cheatsheet — Referencia rápida](#11-cheatsheet--referencia-rápida)

---

## 1. Requisitos previos

| Herramienta | Instalación | Para qué sirve |
|---|---|---|
| Docker + Compose V2 | Docs oficiales de Docker | Contenedores de Odoo, PostgreSQL, Traefik... |
| Git ≥ 2.24 | `apt install git` | Versionado. Copier lo necesita internamente |
| Python ≥ 3.8.1 + venv | `apt install python3 python3-venv` | Base para las herramientas CLI |
| Copier ≥ 9 | `pipx install copier` | El generador de proyectos |
| Invoke | `pipx install invoke` | Tareas automatizadas del proyecto |
| pre-commit | `pipx install pre-commit` | Linters y formateo automático al hacer commit |

```bash
# Instalación rápida (Ubuntu/Debian)
sudo apt install git python3 python3-venv python3-pip
python3 -m pip install --user pipx
pipx install copier invoke pre-commit
pipx ensurepath
# Reinicia la terminal para que PATH se actualice
```

---

## 2. Copier: comando inicial y actualización

**Antes de empezar:** Copier te pedirá varias contraseñas. El propio template te da un enlace para generarlas:

👉 [https://ddg.gg/?q=password+64+strong](https://ddg.gg/?q=password+64+strong) — búsqueda en DuckDuckGo que genera una contraseña fuerte de 64 caracteres.

En Halltic **siempre** usamos ese enlace para generar contraseñas fuertes, incluso en dev. Genera al menos 3 (Odoo admin, PostgreSQL, y backup passphrase si aplica) y guárdalas en tu gestor de contraseñas **antes** de lanzar `copier copy`.

**Crear proyecto nuevo:**
```bash
copier copy gh:Halltic/doodba-copier-template ~/proyectos/cliente-odoo --trust
```

- `--trust` es obligatorio: el template ejecuta tareas post-copia (`invoke after-update` + `invoke develop`) que inicializan Git y pre-commit.
- Las respuestas se guardan en `.copier-answers.yml` dentro del proyecto.

**Actualizar proyecto existente** (cuando el template se actualiza):
```bash
cd ~/proyectos/cliente-odoo
copier update --trust
```
Copier recuerda tus respuestas anteriores. Pulsa Enter para mantenerlas o cambia lo que necesites.

---

## 3. Perfiles de proyecto Halltic

Antes de responder las preguntas, identifica en qué perfil cae tu proyecto.

### Perfil A — Proyecto cliente nuevo (el más común)

> Cliente que contrata a Halltic para implantar Odoo. Módulos privados, sin website público.

| Pregunta clave | Respuesta |
|---|---|
| `odoo_version` | La que pida el cliente (17.0 o 18.0) |
| `project_name` | `nombre-cliente` (ej: `acme-erp`) |
| `project_license` | `OPL-1.0` (módulos propietarios) |
| `project_author` | `Halltic Tech S.L.` (ya es el default del fork) |
| `domains_prod` | Los dominios del cliente (los configura el senior) |
| `postgres_version` | `17` (última estable) |
| `postgres_dbname` | `prod` (default, convención Halltic) |
| `odoo_listdb` (prod) | `No` |
| `backup_dst` | Lo configura el senior/devops |
| `smtp_relay_host` | Lo configura el senior |
| Contraseñas | Todas con [ddg.gg/?q=password+64+strong](https://ddg.gg/?q=password+64+strong) |
| Todo lo demás | Valores por defecto o vacío |

### Perfil B — Proyecto cliente con ecommerce/website

> Igual que A, pero con website público. Afecta a crawlers y dominios.

| Diferencia vs Perfil A | Respuesta |
|---|---|
| `paths_without_crawlers` | `[/web]` (solo el backend, NO `/website/info`) |
| `paths_with_crawlers` | `[/web/image/website]` |
| `domains_prod` | Incluye dominios con `redirect_to` (www → sin www o viceversa) |

### Perfil C — Módulo interno / POC / formación

> Para probar cosas, aprender, o desarrollar un módulo aislado.

| Pregunta clave | Respuesta |
|---|---|
| `odoo_version` | La que estés estudiando |
| `project_name` | Descriptivo: `poc-facturacion`, `formacion-17` |
| `project_license` | `no_license` |
| `domains_prod` | `{}` (no vas a producción) |
| `backup_dst` | Vacío |
| `smtp_relay_host` | Vacío (emails van a MailHog) |

---

## 4. Las preguntas de Copier

El cuestionario tiene **42 preguntas en total**, pero muchas son condicionales (solo aparecen si has respondido algo concreto antes). En una ejecución típica para dev local verás unas 26-28.

### 🔴 Críticas (difíciles de cambiar después)

---

#### Q1. `odoo_version` — Versión de Odoo

> "On which odoo version is it based?"

**Valores:** 7.0 – 19.0 (default: `19.0`)

**Qué hace:** Define la imagen Docker base, los puertos de desarrollo, y qué linters se activan.

**Dónde aterriza:**
- `common.yaml` → imagen base `tecnativa/doodba:{version}`
- Puertos dev → `localhost:{MAJOR}069` (Odoo 17 = `localhost:17069`, Odoo 18 = `localhost:18069`)
- MailHog dev → `localhost:{MAJOR}025`
- wdb debugger → `localhost:{MAJOR}984`

| Situación | Pon |
|---|---|
| Proyecto cliente nuevo sin preferencia | `18.0` (última estable probada) |
| Cliente ya tiene Odoo 17 en producción | `17.0` |
| Migración desde versión anterior | La versión destino |
| POC o formación | La que necesites practicar |

**⚠️ Error común:** Cambiar la versión después de crear el proyecto requiere reconstruir todo (BD incompatible entre versiones mayores).

---

#### Q2. `project_name` — Nombre del proyecto

> "What's your project name?"

**Default:** `myproject-odoo`  
**Restricción:** Solo `A-Za-z0-9-_`. Sin puntos, sin espacios, sin ñ.

**Dónde aterriza:** Contenedores `{project_name}-odoo-1`, redes Docker, labels Traefik, workspace VSCode.

| Situación | Ejemplo |
|---|---|
| Proyecto cliente | `acme-erp`, `ferreteria-lopez` |
| Proyecto interno Halltic | `halltic-interno`, `halltic-rrhh` |
| POC / formación | `poc-website`, `formacion-18` |

**⚠️ Error común:** Usar nombres genéricos como `test` o `odoo`. Si tienes varios proyectos en la misma máquina, los contenedores colisionan.

---

#### Q3. `postgres_version` — Versión de PostgreSQL

> "Which PostgreSQL version do you want to deploy?"

**Default:** `17`  
**Valores:** `""` (servidor externo), `9.6`, `10`–`17`

**Dónde aterriza:** `common.yaml` → `postgres:{version}-alpine`

**⚠️ Validación automática de compatibilidad odoo↔postgres:**

| Odoo | PG mínimo | PG máximo |
|---|---|---|
| ≥ 16.0 | 12 | (sin límite) |
| ≥ 14.0 | 10 | (sin límite) |
| ≤ 14.0 | — | 16 |
| ≤ 12.0 | — | 13 |

| Situación | Pon |
|---|---|
| Proyecto nuevo | `17` |
| Cliente con BD existente en PG 15 | `15` (misma versión que prod) |
| Servidor externo (no Docker) | `""` (primera opción) |

**⚠️** PostgreSQL 9.6 no tiene soporte de backups en este template.

---

#### Q4. `domains_prod` — Dominios de producción

> "Configure the production domains for this project."

**Default:** `{}` (vacío) · **Tipo:** YAML multiline

**Dónde aterriza:** `prod.yaml` → labels de Traefik en el servicio `odoo`

**Como junior, normalmente pondrás `{}`** (vacío). El senior/devops lo configura al desplegar. Pero debes entender la estructura:

```yaml
# Dominio principal + redirección
- hosts:
    - www.acme.com
    - shop.acme.com
  cert_resolver: letsencrypt    # HTTPS automático

# Redirección permanente
- hosts:
    - acme.com
  redirect_to: www.acme.com
  redirect_permanent: true       # 301 en vez de 302

# Dominio VPN con cert autofirmado
- hosts:
    - vpn.acme.com
  cert_resolver: true
```

**Claves disponibles por dominio:**

| Clave | Tipo | Default | Qué hace |
|---|---|---|---|
| `hosts` | lista strings | (obligatorio) | Nombres de dominio |
| `path_prefixes` | lista strings | `[]` | Solo enruta esos paths (ej: `[/shop]`) |
| `redirect_to` | string | `null` | Redirige a otro dominio |
| `redirect_permanent` | bool | `false` | 301 en vez de 302 |
| `cert_resolver` | string/bool | `"letsencrypt"` | `true`=autofirmado, `false`=sin TLS |
| `entrypoints` | lista strings | `[]` | Entrypoints específicos de Traefik |

💡 El primer host del primer dominio sin `path_prefixes` se considera el dominio principal.

---

#### Q5. `backup_dst` — Destino de backups

> "Where should the backups be stored?"

**Default:** `""` (vacío) · **Condición:** Solo aparece si `postgres_version` ≥ 10

**Dónde aterriza:** `prod.yaml` → servicio `backup` (Duplicity)

**Como junior: déjalo vacío.** El senior lo configura. Vacío = **sin backups automáticos** (inaceptable en producción).

Formatos: `boto3+s3://bucket/path`, `sftp://user@host/path`, o cualquier URL soportada por [Duplicity](http://duplicity.nongnu.org/vers8/duplicity.1.html#sect7).

**Si rellenas backup_dst**, Copier te preguntará estas preguntas adicionales:

| Pregunta | Default | Qué hace |
|---|---|---|
| `backup_image_version` | `latest` | Versión de `docker-duplicity-postgres` |
| `backup_email_from` | `""` | Remitente de reportes de backup |
| `backup_email_to` | `""` | Destinatario de reportes |
| `backup_deletion` | `false` | Borrar backups antiguos vía cron (activar si NO usas S3 lifecycle rules) |
| `backup_tz` | `UTC` | Timezone para el cron de backups |
| `backup_passphrase` | `example-backup-passphrase` | 🔐 Clave para encriptar backups. **Necesaria para restaurar.** Genera con [ddg.gg](https://ddg.gg/?q=password+64+strong) |

**Si usas S3** (`s3:` en la URL), además te pide:

| Pregunta | Qué hace |
|---|---|
| `backup_aws_access_key_id` | 🔐 Access key de AWS/S3 |
| `backup_aws_secret_access_key` | 🔐 Secret key de AWS/S3 |

---

### 🟡 Importantes (afectan al flujo, ajustables después)

---

#### Q6. `odoo_proxy` — Reverse proxy

> "Which proxy will you use to deploy odoo?"

**Default:** `traefik`

| Opción | Valor interno | Cuándo usarla |
|---|---|---|
| **Traefik** | `traefik` | Siempre en Halltic |
| Other proxy | `other` | Si hay nginx/Apache ya configurado |
| No proxy | `""` | ⚠️ Solo dev local aislado. Peligroso en producción |

En **dev** no afecta: siempre accedes por `localhost:{MAJOR}069`.

---

#### Q7. `traefik_version` — Versión de Traefik

> "Indicate Traefik version (v2 recommended)"

**Condición:** Solo aparece si `odoo_proxy` = `traefik`  
**Default:** `v2.4` (valor interno: `2`)

| Opción | Valor | Cuándo |
|---|---|---|
| v1.7 | `1` | ❌ Deprecado, no usar |
| **v2.4** | `2` | ✅ Default recomendado |
| v3.0 | `3` | Si el servidor ya tiene Traefik 3 |

**Dónde aterriza:** Selecciona la plantilla Jinja `_traefik{1,2,3}_labels.yml.jinja`. La sintaxis de labels es radicalmente diferente entre versiones.

**⚠️** Pregunta al senior qué versión de Traefik corre en staging/prod antes de elegir.

---

#### Q8. `odoo_admin_password` — Contraseña maestra de Odoo

> "What will be your odoo admin password?"
> 💡 To auto-generate strong passwords, see https://ddg.gg/?q=password+64+strong

**Default:** `example-admin-password` (¡cámbialo siempre!) · **Tipo:** secret

**Qué hace:** Protege `/web/database/manager` (crear, borrar, duplicar BDs).

**Dónde aterriza:** `.docker/odoo.env` → variable `ADMIN_PASSWD`

**En Halltic:** siempre generada desde [ddg.gg/?q=password+64+strong](https://ddg.gg/?q=password+64+strong).

**⚠️ Crítico:** Quien tenga esta contraseña puede **borrar la base de datos**. Nunca la commitees a Git (`.docker/odoo.env` está en `.gitignore`).

---

#### Q9. `odoo_listdb` / Q10. `odoo_listdb_staging` — Listar BDs

| Pregunta | Default | Recomendación Halltic |
|---|---|---|
| `odoo_listdb` (prod) | `false` | **`false` siempre** |
| `odoo_listdb_staging` | hereda de `odoo_listdb` | `true` (facilita testing) |

---

#### Q11. `postgres_dbname` — Nombre de la BD

> **Default:** `prod`

💡 Usar `prod` en todos los entornos facilita restaurar backups de producción en dev sin renombrar la BD. Es la **convención Halltic**.

---

#### Q12. `postgres_password` — Contraseña de PostgreSQL

> **Default:** `example-db-password` (¡cámbialo siempre!) · **Tipo:** secret

**En Halltic:** siempre desde [ddg.gg/?q=password+64+strong](https://ddg.gg/?q=password+64+strong). Se guarda en `.docker/db-access.env` (gitignored).

---

#### Q13. `odoo_dbfilter` — Filtro de BD

> **Default:** `^{postgres_dbname}` (dinámico, ej: `^prod`)

Regex que restringe qué BDs son visibles en producción.

| Situación | Valor |
|---|---|
| BD se llama `prod` | `^prod$` |
| BD se llama `acme` | `^acme$` |
| Dev/POC | Default (`^prod`) |

**⚠️ Error común:** El default es `^prod` sin `$`. Eso matchea `prod`, `prod2`, `produccion`... En producción, cierra siempre con `$`.

---

#### Q14. `project_license` — Licencia

> **Default:** `BSL-1.0`

| Tipo de proyecto | Licencia |
|---|---|
| Módulos para cliente | `OPL-1.0` |
| Módulos basados en Odoo Enterprise | `OEEL-1.0` |
| Contribución OCA | `LGPL-3.0-or-later` o `AGPL-3.0-or-later` |
| POC / interno | `no_license` |

---

### 🟢 Opcionales / Avanzadas

---

#### Q15. `odoo_initial_lang` — Idioma inicial

> **Default:** `es_ES` (ya customizado en el fork Halltic)

En Halltic: siempre `es_ES`. Carga traducciones al crear la BD.

---

#### Q16. `odoo_oci_image` — Registro Docker

> **Default:** `""` (vacío)

**Como junior: déjalo vacío.** Si el proyecto usa CI/CD, el senior lo configura.

---

#### Q17. `project_author` — Autor

> **Default en fork Halltic:** `Halltic Tech S.L.`

Si tu módulo tiene `"author": "Pablo"`, pylint dará warning. Debe ser `"author": "Halltic Tech S.L."`.

---

#### Q18. `gitlab_url` — URL de GitLab

> **Default:** `""`. **Como junior: déjalo vacío.**

---

#### Q19. `domains_test` — Dominios de staging

Mismo formato que `domains_prod`. Ejemplo:
```yaml
- hosts:
    - acme.staging.halltic.com
  cert_resolver: true  # autofirmado para staging
```

**Como junior:** Pregunta al senior qué subdominio usar.

---

#### Q20. `paths_without_crawlers` — Rutas bloqueadas para bots

> **Default:** `[/web, /website/info]`

| Tipo de proyecto | Valor |
|---|---|
| Sin website público | Default `[/web, /website/info]` |
| Con ecommerce | `[/web]` (NO bloquees `/website/info` si quieres SEO) |

---

#### Q21. `paths_with_crawlers` — Excepciones

> **Default:** `[/web/image/website]` — el default está bien.

---

#### Q22. `cidr_whitelist` — Whitelist de IPs

> **Default:** `null`. Solo Traefik 2+. **Como junior: déjalo vacío.**

---

#### Q23. `postgres_username`

> **Default:** `odoo`. **Siempre `odoo`.** No hay razón para cambiarlo.

---

#### Q24. `postgres_exposed` — Exponer PostgreSQL fuera de Docker

> **Default:** `false`

| Entorno | Valor |
|---|---|
| Dev (quieres pgAdmin/DBeaver) | `true` |
| Staging/Prod | **`false` siempre** |

**Si pones `true`** y Traefik ≠ v3, aparecen preguntas adicionales: `postgres_exposed_port` (default `5432`) y `postgres_cidr_whitelist`.

---

#### Q25. `smtp_default_from`

> **Default:** `""`. **Como junior: déjalo vacío.**

---

#### Q26. `smtp_relay_host` — Servidor SMTP

> **Default:** `""`. ⚠️ Si lo dejas vacío, **todas las demás preguntas SMTP se saltan.**

**Como junior: déjalo vacío.** En dev los emails van a MailHog (`localhost:{MAJOR}025`).

**Si se rellena** (lo hace el senior), Copier pregunta adicionalmente:

| Pregunta | Default | Nota |
|---|---|---|
| `smtp_relay_port` | `587` | ⚠️ **Nunca usar 465** |
| `smtp_relay_user` | `""` | Debe poder hacer mail spoofing |
| `smtp_relay_password` | `example-smtp-password` | 🔐 |
| `smtp_relay_version` | `13` | Versión de docker-mailserver |
| `smtp_canonical_default` | `""` | Dominio canónico para SPF/DKIM/DMARC |
| `smtp_canonical_domains` | `[]` | Dominios adicionales autorizados |

---

## 5. Archivos generados tras Copier

```
tu-proyecto/
├── .copier-answers.yml          ← Tus respuestas (versionado en Git)
├── .docker/
│   ├── odoo.env                 ← ADMIN_PASSWD, config Odoo (🔒 gitignored)
│   ├── db-access.env            ← PGUSER, PGPASSWORD, PGDATABASE (🔒 gitignored)
│   ├── db-creation.env          ← POSTGRES_PASSWORD para crear contenedor (🔒 gitignored)
│   └── backup.env               ← AWS keys, passphrase (🔒 gitignored, si backup_dst)
├── .vscode/                     ← Config de VSCode para debug
├── common.yaml                  ← Docker Compose base (imagen, volúmenes, BD)
├── devel.yaml                   ← Docker Compose dev (hot-reload, wdb, MailHog)
├── test.yaml                    ← Docker Compose staging (Traefik, sin debug)
├── prod.yaml                    ← Docker Compose prod (Traefik, backups, SMTP)
├── setup-devel.yaml             ← Compose auxiliar para git-aggregate
├── odoo/
│   ├── custom/
│   │   ├── dependencies/        ← apt.txt, pip.txt, gem.txt (deps extra)
│   │   └── src/
│   │       ├── private/         ← 🎯 TUS MÓDULOS VAN AQUÍ
│   │       ├── repos.yaml       ← Repos externos (OCA, etc.) a agregar
│   │       └── addons.yaml      ← Qué addons activar de cada repo
│   └── auto/                    ← Generado automáticamente (no tocar)
├── tasks.py                     ← Tareas invoke (toda la sección 6)
├── .pre-commit-config.yaml      ← Hooks de linting
├── LICENSE                      ← Según project_license elegida
└── README.md                    ← Generado con datos del proyecto
```

**Archivos clave:**

| Archivo | Para qué | ¿Lo editas? |
|---|---|---|
| `odoo/custom/src/private/` | Tus módulos Odoo | ✅ Siempre |
| `odoo/custom/src/repos.yaml` | Repos OCA/terceros | ✅ Cuando añades dependencias |
| `odoo/custom/src/addons.yaml` | Activar addons de esos repos | ✅ Junto con repos.yaml |
| `odoo/custom/dependencies/pip.txt` | Dependencias Python extra | ✅ Si tu módulo necesita librerías |
| `.docker/odoo.env` | Variables de Odoo | ⚠️ Solo contraseñas y config local |
| `.docker/db-access.env` | Acceso a PostgreSQL | ⚠️ Solo contraseñas |
| `common.yaml` / `devel.yaml` | Docker Compose | 🚫 Raramente (Copier los gestiona) |
| `.copier-answers.yml` | Respuestas de Copier | 🚫 No editar a mano |

💡 Los archivos en `odoo/custom/dependencies/`, `odoo/custom/src/private/`, `repos.yaml`, `addons.yaml` y `odoo/custom/ssh/` **no se sobreescriben** al hacer `copier update` (están en `_skip_if_exists`).

---

## 5b. Formato de `repos.yaml` y `addons.yaml`

Estos dos archivos se mencionan en toda la guía pero su sintaxis nunca se ha explicado. Son los archivos que controlan **qué código externo descarga y activa** tu proyecto.

---

### `repos.yaml` — Qué repos descargar

**Ubicación:** `odoo/custom/src/repos.yaml`

Este archivo es leído por `gitagregate` (vía `inv git-aggregate`). Cada entrada le dice: "descarga este repo Git, en esta carpeta, con estas opciones".

**Sintaxis básica:**

```yaml
# Formato: <ruta_destino>:
#   Donde la ruta es relativa a odoo/custom/src/

./OCA/account-financial-tools:
  defaults: &OCA       # ancla YAML reutilizable (patrón Halltic)
    remotes:
      origin: https://github.com/OCA/account-financial-tools.git
    target: origin 17.0
    merges:
      - origin 17.0   # rama base

./OCA/server-tools:
  defaults: &OCA_server
    remotes:
      origin: https://github.com/OCA/server-tools.git
    target: origin 17.0
    merges:
      - origin 17.0
```

**Estructura de cada entrada:**

| Campo | Obligatorio | Qué hace |
|---|---|---|
| `remotes` | ✅ | Nombre(s) y URL(s) del remote Git |
| `target` | ✅ | `<remote> <rama>` — la rama que se checkoutea como HEAD |
| `merges` | ✅ | Lista de ramas/PRs a merge sobre `target` |
| `fetch` | No | Refs adicionales a fetchear (útil para PRs de GitHub) |
| `depth` | No | Profundidad de clone (para acelerar, `depth: 1` en prod) |

---

#### Patrón Halltic: merge de un PR de OCA sin esperar a que se mergee upstream

Cuando un PR de OCA arregla un bug que necesitas ahora mismo:

```yaml
./OCA/account-financial-tools:
  defaults: &OCA
    remotes:
      origin: https://github.com/OCA/account-financial-tools.git
    target: origin 17.0
    merges:
      - origin 17.0          # rama base (siempre va primero)
      - origin refs/pull/1234/head  # PR #1234 (fetch + merge automático)
    fetch:
      - refs/pull/1234/head  # obligatorio para que git pueda ver el PR
```

> **⚠️ Después de añadir o cambiar un repo:** ejecuta `inv git-aggregate` y luego `inv img-build`. Sin `img-build` la imagen no incluirá el nuevo código.

> **💡 `inv closed-prs`** comprueba periódicamente si los PRs que tienes en merges ya se mergearon al upstream, para que puedas limpiarlos.

---

#### Ejemplo completo: proyecto Halltic con 2 repos OCA

```yaml
# odoo/custom/src/repos.yaml

# Repo privado de Halltic (módulos propios)
./private:
  {}  # se gestiona vía Git del proyecto, no gitagregate

# OCA: herramientas de servidor
./OCA/server-tools:
  defaults:
    remotes:
      origin: https://github.com/OCA/server-tools.git
    target: origin 17.0
    merges:
      - origin 17.0

# OCA: contabilidad, con un PR pendiente
./OCA/account-financial-tools:
  defaults:
    remotes:
      origin: https://github.com/OCA/account-financial-tools.git
    target: origin 17.0
    merges:
      - origin 17.0
      - origin refs/pull/9876/head   # fix: importación SEPA
    fetch:
      - refs/pull/9876/head
```

---

### `addons.yaml` — Qué addons activar

**Ubicación:** `odoo/custom/src/addons.yaml`

Este archivo controla **qué módulos** de los repos descargados van a ser incluidos en la imagen Docker. Sin esta configuración, tener el código en disco no es suficiente para que Odoo lo cargue.

**Sintaxis:**

```yaml
# Comentario: <nombre_del_addon>: <opciones>
# Opciones más comunes:
#   true  → incluir el addon
#   false → excluir explícitamente (útil para anular defaults)
#   auto  → incluir solo si sus dependencias están instaladas (default implícito)

# Addons del repo OCA/server-tools
auto_backup: true              # incluir este addon explícitamente
base_setup_default: true
base_technical_user: false     # excluir aunque esté en el repo

# Addons del repo OCA/account-financial-tools
account_bank_statement_import_sepa_direct_debit: true
account_payment_order: true

# Addons privados (en odoo/custom/src/private/)
mi_modulo_halltic: true
mi_modulo_facturacion: true
```

**Valores posibles:**

| Valor | Significa |
|---|---|
| `true` | Incluir siempre en la imagen |
| `false` | Excluir explícitamente (no se copiará a `odoo/auto/addons/`) |
| `auto` | Incluir si sus dependencias están disponibles (comportamiento por defecto) |

> **⚠️ Después de cambiar `addons.yaml`:** siempre ejecuta `inv img-build`. El addon no estará disponible en Odoo hasta que se reconstruya la imagen.

> **💡 No es necesario listar TODOS los addons.** Si un repo tiene 100 módulos y solo quieres 3, lista solo esos 3 con `true`. El resto queda excluido.

---

#### Flujo completo: añadir un nuevo addon OCA

```bash
# 1. Añadir el repo a repos.yaml (si no está)
#    Editar: odoo/custom/src/repos.yaml

# 2. Listar el addon en addons.yaml
#    Editar: odoo/custom/src/addons.yaml
#    Añadir:  nombre_addon: true

# 3. Descargar el código del repo
inv git-aggregate

# 4. Reconstruir imagen con el nuevo addon
inv img-build

# 5. Levantar y luego instalar en Odoo
inv start
inv install --modules nombre_addon
inv start   # volver al modo normal (install para odoo)
```

---

## 6. Comandos Invoke — referencia completa

`invoke` (o `inv`) es la CLI que envuelve todas las operaciones del proyecto. Cada comando es una tarea definida en `tasks.py`. Internamente ejecutan `docker compose` con las opciones correctas.

El binario `invoke` debe llamarse exactamente `invoke` (no `invoke3` ni similares).

### ¿Cómo funciona la imagen Doodba internamente?

Antes de ver los comandos, conviene entender por qué el orden `git-aggregate → img-build` es obligatorio y no opcional.

La imagen base `tecnativa/doodba:{version}` usa **instrucciones `ONBUILD`** en su Dockerfile. Esto significa que el `Dockerfile` de tu proyecto casi no tiene contenido propio, pero cuando Docker hace el `build`, automáticamente se ejecutan triggers heredados de la imagen base que:

1. **Copian** `odoo/custom/` al interior de la imagen (incluyendo los repos que clonó `git-aggregate`)
2. **Instalan** las dependencias `apt`, `pip` y `gem` de `odoo/custom/dependencies/`
3. **Registran** los addons en `odoo/auto/addons/` según `addons.yaml`

Esto tiene una consecuencia directa para el día a día:

```
┌─────────────────────────────────────────────────────────────┐
│  inv git-aggregate   →   código llega a odoo/custom/src/   │
│  inv img-build       →   Docker ONBUILD copia ese código   │
│                           dentro de la imagen               │
│  (sin img-build, la imagen ignora los cambios en disco)    │
└─────────────────────────────────────────────────────────────┘
```

> **Error #1 de juniors:** hacer `git-aggregate` y luego `start` sin `img-build`. La imagen es la de la última build, sin el código nuevo. Odoo arranca pero los addons nuevos no existen.

---

### SETUP INICIAL (una vez al crear el proyecto)

#### `inv develop`
**Qué hace:** Prepara el entorno de desarrollo desde cero.
```bash
inv develop
```
**Internamente ejecuta:**
1. Crea `odoo/auto/addons/` con permisos 777 (para compatibilidad con Podman)
2. `git init` en la raíz del proyecto
3. Crea el symlink `docker-compose.yml → devel.yaml`
4. Genera el archivo `.code-workspace` para VSCode
5. Instala los hooks de `pre-commit`

**Cuándo:** Después del `copier copy`. Se ejecuta automáticamente con `--trust`, pero puedes relanzarlo si algo falla.

**Nota:** Este comando es prerequisito de `git-aggregate` y `closed-prs` — Invoke lo ejecuta automáticamente antes si no se ha corrido.

---

#### `inv git-aggregate`
**Qué hace:** Descarga todos los repos Git externos definidos en `repos.yaml`.
```bash
inv git-aggregate
```
**Internamente ejecuta:**
1. `docker compose --file setup-devel.yaml run --rm odoo` → contenedor temporal que ejecuta `gitaggregate`
2. Clona/actualiza cada repo listado en `odoo/custom/src/repos.yaml`
3. Aplica merges de PRs específicos si están definidos
4. Regenera el `.code-workspace`
5. Instala/desinstala `pre-commit` en cada subrepo según tenga configuración

**Cuándo:**
- Primera vez, después de `inv develop`
- Cada vez que modificas `repos.yaml` (añades repo, cambias merge, etc.)
- Periódicamente para actualizar repos OCA/externos

**Nota:** Respeta `UID/GID` del host para que los archivos clonados tengan permisos correctos.

---

#### `inv img-build`
**Qué hace:** Construye las imágenes Docker del proyecto.
```bash
inv img-build            # Construir y descargar imágenes base actualizadas
inv img-build --no-pull  # Construir sin descargar actualizaciones de imágenes base
```
**Internamente ejecuta:**
- `docker compose build --pull`

**Cuándo:**
- Después de `inv git-aggregate` (el código descargado se mete en la imagen vía ONBUILD)
- Después de modificar `odoo/custom/dependencies/pip.txt` o `apt.txt`
- Después de cambiar `addons.yaml`

**⚠️** Sin este paso, la imagen Docker no contiene tu código. Es el paso que más se olvida.

---

#### `inv img-pull`
**Qué hace:** Descarga imágenes Docker pre-construidas (desde un registro).
```bash
inv img-pull
```
**Internamente ejecuta:**
- `docker compose pull`

**Cuándo:** Solo si el proyecto usa `odoo_oci_image` con un registro Docker y las imágenes se construyen en CI/CD. En la mayoría de proyectos Halltic, usarás `inv img-build` en su lugar.

---

### DÍA A DÍA EN DESARROLLO

#### `inv start`
**Qué hace:** Levanta todos los contenedores del entorno.
```bash
inv start                     # Levantar en background (detach)
inv start --no-detach         # En primer plano (ves logs en directo)
inv start --debugpy           # Con debugpy para VSCode (desactiva hot-reload)
inv start --port-prefix 17    # Usar puertos 17069, 17025... (útil si hay conflicto)
```
**Internamente ejecuta:**
1. `docker compose up -d`
2. Si `--debugpy`: desactiva hot-reload y activa debugger en puerto `{VERSION}899`
3. Si los contenedores ya existían sin cambios, hace `restart` automático
4. Espera unos segundos para que los servicios arranquen

**Cuándo:** Cada vez que empiezas a trabajar o después de un `inv stop`.

---

#### `inv stop`
**Qué hace:** Para todos los contenedores.
```bash
inv stop                # Para contenedores, mantiene datos
inv stop --purge        # Para Y BORRA contenedores, redes, imágenes locales y volúmenes
```
**Internamente ejecuta:**
- `docker compose down --remove-orphans`
- Con `--purge`: añade `--rmi local --volumes` → **CUIDADO: borra la base de datos**

**Cuándo:**
- `inv stop` → al terminar de trabajar
- `inv stop --purge` → cuando quieres empezar completamente de cero

---

#### `inv restart`
**Qué hace:** Reinicia los contenedores de Odoo rápidamente.
```bash
inv restart              # Reinicio rápido (timeout 0)
inv restart --no-quick   # Reinicio graceful (espera cierre limpio)
```
**Internamente ejecuta:**
- `docker compose restart -t0 odoo odoo_proxy`

**Cuándo:**
- Cambios Python que el hot-reload no detectó
- Después de instalar un módulo
- Odoo se queda colgado

**Nota:** Solo reinicia `odoo` y `odoo_proxy`, no la BD ni otros servicios.

---

#### `inv logs`
**Qué hace:** Muestra los logs de los contenedores.
```bash
inv logs                          # Últimos 10 + seguimiento en tiempo real
inv logs --tail 50                # Últimos 50
inv logs --no-follow              # Solo muestra, no sigue
inv logs --container odoo         # Solo odoo
inv logs --container odoo,db      # odoo y db
```
**Cuándo:** Para depurar errores, ver tracebacks, verificar que un módulo se instaló.

---

### GESTIÓN DE MÓDULOS

#### `inv scaffold`
**Qué hace:** Crea la estructura base de un módulo nuevo de Odoo.
```bash
inv scaffold mi_modulo                                            # En directorio actual
inv scaffold mi_modulo --path odoo/custom/src/private             # En path específico
```
**Genera:**
```
mi_modulo/
├── __init__.py
├── __manifest__.py
├── controllers/
├── demo/
├── models/
├── security/
└── views/
```

**⚠️ Restricción:** El path debe estar **dentro del directorio del proyecto**. Si intentas crear un módulo fuera, dará error.

**Cuándo:** Al empezar un módulo nuevo.

---

#### `inv install`
**Qué hace:** Instala módulos de Odoo en la base de datos.
```bash
inv install --modules mi_modulo              # Un módulo específico
inv install --modules mod1,mod2,mod3         # Varios módulos
inv install --cur-file ./ruta/al/archivo.py  # El módulo del archivo actual
inv install --private                         # TODOS los módulos privados
inv install --core                            # Todos los core de Odoo
inv install --extra                           # Todos los extra (OCA, etc.)
inv install --enterprise                      # Todos los enterprise
```
**Internamente ejecuta:**
1. `docker compose stop odoo`
2. `docker compose run --rm odoo addons init -w {módulos}`
3. **No reinicia automáticamente** — debes hacer `inv start` después

**Nota:** En VSCode hay un botón en la statusbar "Install module" que ejecuta esto sobre el archivo abierto.

---

#### `inv uninstall`
**Qué hace:** Desinstala módulos de Odoo.
```bash
inv uninstall --modules mi_modulo
```
**Internamente ejecuta:**
- `docker compose run --rm odoo click-odoo-uninstall -m {módulo}`

---

### TESTING

#### `inv test`
**Qué hace:** Ejecuta los tests de Odoo para módulos específicos.
```bash
inv test --modules mi_modulo                  # Testear un módulo
inv test --cur-file ./ruta/al/archivo.py      # Testear módulo del archivo actual
inv test --private                             # Todos los módulos privados
inv test --modules mod1,mod2 --skip mod2       # Testear mod1, saltar mod2
inv test --modules mi_modulo --debugpy         # Con debugger VSCode
inv test --modules mi_modulo --mode update     # Modo update (en vez de init)
inv test --modules mi_modulo --db-filter ""    # Sin filtro de BD
```
**Internamente ejecuta:**
1. Si `--debugpy`: levanta contenedor con debugpy habilitado
2. Si no: `docker compose run --rm odoo odoo --test-enable --stop-after-init --workers=0 -i {módulos}`
3. Desde Odoo 12+: añade `--test-tags` para limitar tests a los módulos explícitos

**⚠️ Importante:** Después de `inv test`, Odoo queda parado. Debes ejecutar `inv start` para volver al modo normal.

---

### BASE DE DATOS

#### `inv resetdb`
**Qué hace:** Destruye y recrea la base de datos desde cero.
```bash
inv resetdb                                    # BD "devel" con módulo "base"
inv resetdb --dbname mi_bd                     # BD específica
inv resetdb --modules mi_modulo                # Instala módulos específicos
inv resetdb --private                          # Instala todos los módulos privados
inv resetdb --dependencies --modules mi_modulo # Solo dependencias del módulo
inv resetdb --no-populate                      # No ejecutar preparedb después
```
**Internamente ejecuta:**
1. `docker compose stop odoo`
2. `click-odoo-dropdb devel` → borra la BD
3. `click-odoo-initdb -n devel -m {módulos}` → crea BD nueva con módulos
4. `preparedb` → configura la BD con valores útiles para desarrollo

> **⚠️ Odoo 19+:** Usa el CLI nativo de Odoo en vez de `click-odoo-initdb` por cambios internos en el Registry. Si tu proyecto es Odoo 19 o superior, el flujo de inicialización de base de datos cambia — confirma con el senior antes de ejecutar `resetdb`.

**Cuándo:**
- BD corrupta o llena de basura
- Entorno limpio para probar algo
- Cambios grandes en modelos

---

#### `inv preparedb`
**Qué hace:** Ejecuta el script `preparedb` dentro del contenedor (parámetros del sistema, configuración base).
```bash
inv preparedb
```
Solo disponible desde Odoo 11+. Se ejecuta automáticamente después de `inv resetdb`.

---

#### `inv snapshot`
**Qué hace:** Crea una copia de la BD actual.
```bash
inv snapshot                                         # Snapshot de "devel" con timestamp
inv snapshot --source-db devel                       # BD origen
inv snapshot --destination-db mi_backup_manual       # Nombre personalizado
```
**Internamente:** Para odoo y db → `click-odoo-copydb devel devel-2026_02_21-14_30` → reinicia si estaban activos.

**Cuándo:** Antes de hacer algo arriesgado (migración, borrado masivo, etc.)

---

#### `inv restore-snapshot`
**Qué hace:** Restaura un snapshot previo.
```bash
inv restore-snapshot                                   # Restaura el último automáticamente
inv restore-snapshot --snapshot-name devel-2026_02_21   # Uno específico
inv restore-snapshot --destination-db devel             # A qué BD restaurar
```
**Internamente:** Si no se especifica nombre, busca el snapshot más reciente por fecha → dropdb → copydb → reinicia.

---

### CALIDAD DE CÓDIGO

#### `inv lint`
**Qué hace:** Ejecuta todos los linters y formatters configurados.
```bash
inv lint              # Lint normal
inv lint --verbose    # Con output detallado
```
**Internamente:** `pre-commit run --show-diff-on-failure --all-files --color=always`

**Qué revisa:** Prettier (YAML, XML), Pylint, Ruff, ESLint, y todos los hooks OCA.

**Cuándo:** Se ejecuta automáticamente en cada `git commit` gracias a pre-commit. Manualmente para verificar antes de push.

---

#### `inv updatepot`
**Qué hace:** Actualiza archivos de traducción (.pot/.po) de un módulo.
```bash
inv updatepot --module mi_modulo              # Un módulo
inv updatepot --all                            # Todos
inv updatepot --repo server-tools              # Todos los de un repo
inv updatepot --module mi_modulo --no-msgmerge # Sin merge a .po existentes
```
**Internamente:** Para Odoo → `click-odoo-makepot` → limpia archivos temporales y fechas → ejecuta pre-commit sobre archivos modificados.

**Cuándo:** Después de añadir/modificar strings traducibles (`_("...")`) en tu módulo.

---

### UTILIDADES

#### `inv write-code-workspace-file`
**Qué hace:** Regenera el archivo `.code-workspace` de VSCode.
```bash
inv write-code-workspace-file
inv write-code-workspace-file --cw-path doodba.custom.code-workspace
```
**Qué configura:** Carpetas del workspace, Python (linting, paths, formatter), debug (debugpy, Firefox, Chrome), tasks VSCode.

Se ejecuta automáticamente con `inv develop` y `inv git-aggregate`.

---

#### `inv after-update`
**Qué hace:** Acciones post-actualización del template Copier (permisos, limpieza).

Se ejecuta automáticamente después de `copier copy/update`. Nunca lo ejecutas manualmente.

---

#### `inv closed-prs`
**Qué hace:** Comprueba si algún PR referenciado en `repos.yaml` ha sido cerrado (mergeado o rechazado).
```bash
inv closed-prs
```
**Cuándo:** Periódicamente, para limpiar merges de PRs que ya están en upstream.

---

## 7. Flujos de trabajo

### 7.1 Flujo inicial completo (después de Copier)

```
copier copy gh:Halltic/doodba-copier-template ~/proyectos/cliente --trust
  └── (automático) inv after-update + inv develop

cd ~/proyectos/cliente

inv git-aggregate      # 1. Descargar código (repos.yaml → odoo/custom/src/)
inv img-build          # 2. Construir imagen Docker con el código descargado
inv resetdb            # 3. Crear base de datos
inv start              # 4. Arrancar entorno
```

**¿Por qué ese orden?** Es una cadena de dependencias:
- `git-aggregate` descarga el código → sin él, la imagen se construye vacía
- `img-build` mete el código en la imagen Docker (triggers ONBUILD del Dockerfile de Doodba)
- `resetdb` ejecuta Odoo dentro de la imagen para crear tablas en PostgreSQL
- `start` levanta todo: ya tiene imagen, código y base de datos

---

### 7.2 ¿Cuándo repetir cada paso?

| Cambio realizado | Repetir desde |
|---|---|
| Modificas `repos.yaml` (nuevo repo/merge) | `git-aggregate` → `img-build` → `resetdb` → `start` |
| Modificas `addons.yaml` (activar addon existente) | `img-build` → `resetdb` → `start` |
| Modificas código Python en `private/` | Solo `inv restart` (hot-reload en dev) |
| Modificas vistas XML en `private/` | Actualizar módulo desde Odoo o `inv install` |
| Cambias dependencias pip/apt | `img-build` → `start` |
| BD corrupta o quieres empezar limpio | `resetdb` → `start` |
| Cambias `copier.yml` o actualizas template | `copier update --trust` |

---

### 7.3 Flujo típico de trabajo diario

```
1. inv start                          # Levantar entorno
2. (editar código)                    # Hot-reload recarga automáticamente
3. inv install --modules mi_modulo    # Si es un módulo nuevo
   inv start                          # Volver a levantar (install para odoo)
4. inv restart                        # Si hot-reload no detectó cambios
5. inv test --modules mi_modulo       # Verificar que funciona
   inv start                          # Volver al modo normal (test para odoo)
6. git add . && git commit            # Pre-commit ejecuta linters
7. inv stop                           # Al terminar el día
```

---

### 7.4 Flujo de emergencia (algo se rompió)

```
1. inv logs --container odoo           # Ver qué error hay
2. inv restart                         # Intentar reiniciar
3. inv snapshot                        # Si funciona, guardar estado antes de tocar
4. inv resetdb                         # Nuclear: empezar de cero si nada funciona
```

---

### 7.5 Flujo para nuevo repo externo

```
1. (editar repos.yaml y addons.yaml)
2. inv git-aggregate                   # Descarga el repo
3. inv img-build                       # Reconstruir imagen con el nuevo código
4. inv start                           # Levantar
5. inv install --modules nuevo_addon   # Instalar el módulo
   inv start                           # Volver a levantar
```

---

## 8. Resumen: qué cambia entre entornos

| Configuración | Dev (`devel.yaml`) | Staging (`test.yaml`) | Prod (`prod.yaml`) |
|---|---|---|---|
| Acceso | `localhost:{MAJOR}069` | Dominio staging | Dominio real |
| HTTPS | No | Sí (Traefik, cert autofirmado) | Sí (Let's Encrypt) |
| SMTP | MailHog (fake) | MailHog (fake) | SMTP real del cliente |
| Backups | No | No | Sí (Duplicity) |
| Listar BDs | Siempre | Configurable | **No** |
| PostgreSQL expuesto | Opcional | No | **No** |
| Contraseñas | Fuertes (`ddg.gg`) | Fuertes | **Fuertes** |
| Debug (wdb/debugpy) | Sí | No | No |
| Hot-reload (`--dev`) | Sí | No | No |
| Demo data | Sí | No (`WITHOUT_DEMO=all`) | No |
| Pgweb | Sí (`{MAJOR}081`) | No | No |

---

## 9. Errores frecuentes de juniors

| Síntoma | Causa probable | Solución |
|---|---|---|
| `localhost:17069` no responde | Contenedor no arrancó | `inv logs --container odoo` y busca el error |
| "Database not found" | BD no creada o `db_filter` mal puesto | Accede a `/web/database/manager` con la contraseña maestra |
| Módulo no aparece en Apps | No actualizaste la lista | Ajustes → Actualizar lista de aplicaciones |
| Pylint: "wrong author" | `__manifest__.py` no tiene `"author": "Halltic Tech S.L."` | Añade el autor correcto |
| `pre-commit` falla al hacer commit | YAML mal formateado (normal tras copier) | `pre-commit run --all-files` y commitea los fixes |
| Conflicto de puertos | Otro proyecto usa el mismo puerto | `inv stop` en el otro proyecto, o `inv start --port-prefix XX` |
| `invoke develop` falla | Dependencias de sistema faltantes | Revisa que tienes `python3-venv` instalado |
| "password authentication failed for user odoo" | Password de PG no coincide entre `.docker/db-access.env` y `.docker/db-creation.env` | Verifica que ambos tienen la misma contraseña. Si cambiaste la pass después de crear el contenedor: `inv stop --purge` y vuelve a levantar |
| "The postgres version is too low/high" | Incompatibilidad odoo↔postgres | Consulta la tabla de compatibilidad en Q3 |
| Imagen vacía (no se ven addons) | Faltó `inv img-build` después de `git-aggregate` | `inv img-build` → `inv start` |
| `inv test` y luego Odoo no arranca | `test` para Odoo al terminar | `inv start` para volver al modo normal |
| `inv scaffold` da error de path | Path fuera del directorio del proyecto | El path debe ser relativo al proyecto o estar dentro de él |

---

## 10. Integración con VSCode

Doodba genera automáticamente un archivo `.code-workspace` cuando ejecutas `inv develop` o `inv git-aggregate`. Este workspace configura todo lo necesario para trabajar cómodamente con VSCode.

### 10.1 Abrir el workspace

```bash
# Desde la raíz del proyecto
code doodba.code-workspace
# o si se llama distinto:
code *.code-workspace
```

> **⚠️ Siempre abre el `.code-workspace`, NO la carpeta del proyecto directamente.** El workspace configura Python path, linting y debug. Sin él, pylint no encontrará los imports de Odoo.

---

### 10.2 Tasks de la statusbar (barra de estado)

Al abrir el workspace verás botones en la barra inferior de VSCode. Estos botones ejecutan tareas Invoke directamente sin abrir la terminal:

| Botón en statusbar | Comando equivalente | Cuándo usarlo |
|---|---|---|
| `▶ Start Odoo` | `inv start` | Levantar el entorno |
| `⏹ Stop Odoo` | `inv stop` | Parar el entorno |
| `🔄 Restart Odoo` | `inv restart` | Reiniciar tras cambios |
| `📦 Install module` | `inv install --cur-file ${file}` | Instalar el módulo del archivo abierto |
| `🧪 Test module` | `inv test --cur-file ${file}` | Testear el módulo del archivo abierto |
| `📜 Logs` | `inv logs` | Ver logs en directo |

> **💡 Consejo:** Abre el archivo Python de tu módulo antes de pulsar "Install module" o "Test module". VSCode detecta automáticamente a qué módulo pertenece el archivo.

Si no ves los botones, instala la extensión **[Task Buttons](https://marketplace.visualstudio.com/items?itemName=spmeesseman.vscode-taskbuttons)** o usa `Ctrl+Shift+P → Run Task`.

---

### 10.3 Debug con debugpy (breakpoints en Python)

Doodba incluye soporte nativo para `debugpy`, el debugger de Python que VSCode usa internamente.

#### Paso 1: Levantar Odoo en modo debug

```bash
# Opción A: arrancar directamente con debugger
inv start --debugpy

# Opción B: si Odoo ya está arrancado, para y relanza
inv stop
inv start --debugpy
```

> **⚠️ `--debugpy` desactiva el hot-reload** (no pueden coexistir). Úsalo solo cuando necesites hacer debug con breakpoints.

#### Paso 2: Hacer attach desde VSCode

1. Abre el panel **Run and Debug** (`Ctrl+Shift+D`)
2. En el desplegable, selecciona **`Attach to Odoo`** (ya está preconfigurado en el workspace)
3. Pulsa ▶ (Play) o `F5`
4. VSCode conecta al proceso Odoo dentro del contenedor

Si la configuración no aparece en el desplegable, verifica que abriste el `.code-workspace` y no la carpeta directamente.

#### Paso 3: Poner breakpoints

1. Abre el archivo Python donde quieres parar (ej: `models/sale_order.py`)
2. Haz clic en el margen izquierdo junto al número de línea → aparece el punto rojo 🔴
3. Ejecuta la acción en el navegador que dispara ese código (ej: confirmar una venta)
4. VSCode para la ejecución en el breakpoint → puedes inspeccionar variables, call stack, etc.

#### Puertos de debugpy por versión

| Versión Odoo | Puerto debugpy |
|---|---|
| 17.0 | `17899` |
| 18.0 | `18899` |
| Genérico | `{MAJOR}899` |

---

### 10.4 Debug de tests con debugpy

```bash
# Ejecutar tests con debugger habilitado
inv test --modules mi_modulo --debugpy
```

Después, haz attach desde VSCode igual que en el paso 2 de arriba. Puedes poner breakpoints en los propios tests (`test_mymodule.py`) y en el código que prueban.

---

### 10.5 Estructura del workspace generado

El archivo `.code-workspace` incluye automáticamente:

- **Carpetas:** raíz del proyecto + cada repo clonado por `git-aggregate`
- **Python path:** apunta a `odoo/custom/src/` para que pylint encuentre imports
- **Configuración debugpy:** `Attach to Odoo` preconfigurado con el puerto correcto
- **Tasks:** los botones de la statusbar descritos arriba
- **Extensions recomendadas:** Python, Pylance, EditorConfig, etc.

Se regenera automáticamente con `inv git-aggregate` (para añadir los repos nuevos) y con `inv write-code-workspace-file`.

---

## 11. Cheatsheet — Referencia rápida

Tabla de referencia para tener a mano. Cubre el 90% del trabajo diario.

### Setup (una vez)

| Comando | Cuándo |
|---|---|
| `copier copy gh:Halltic/doodba-copier-template ./mi-proyecto --trust` | Crear proyecto nuevo |
| `copier update --trust` | Actualizar template (cuando hay nuevas versiones) |
| `inv develop` | Inicializar entorno (git, symlinks, pre-commit) |
| `inv git-aggregate` | Descargar repos externos (repos.yaml) |
| `inv img-build` | Construir imagen Docker con el código |
| `inv resetdb` | Crear base de datos desde cero |

### Día a día

| Comando | Cuándo |
|---|---|
| `inv start` | Levantar el entorno cada mañana |
| `inv start --debugpy` | Levantar con soporte de breakpoints VSCode |
| `inv stop` | Parar el entorno al terminar |
| `inv restart` | Reiniciar Odoo rápido (cambios Python) |
| `inv logs` | Ver logs en tiempo real |
| `inv logs --container odoo` | Solo logs de Odoo |

### Módulos

| Comando | Cuándo |
|---|---|
| `inv scaffold mi_modulo` | Crear estructura de módulo nuevo |
| `inv install --modules mi_modulo` | Instalar módulo en la BD |
| `inv install --cur-file ./models/mi_model.py` | Instalar módulo del archivo actual |
| `inv uninstall --modules mi_modulo` | Desinstalar módulo |
| `inv test --modules mi_modulo` | Ejecutar tests del módulo |
| `inv updatepot --module mi_modulo` | Actualizar traducciones |

### Base de datos

| Comando | Cuándo |
|---|---|
| `inv resetdb` | Recrear BD (borra todo y empieza) |
| `inv snapshot` | Guardar estado actual de la BD |
| `inv restore-snapshot` | Restaurar último snapshot |
| `inv preparedb` | Configurar parámetros del sistema |

### Calidad y repos

| Comando | Cuándo |
|---|---|
| `inv lint` | Ejecutar todos los linters manualmente |
| `inv git-aggregate` | Actualizar repos externos |
| `inv closed-prs` | Verificar si los PRs en repos.yaml ya se mergearon |

### Reglas del juego

```
Modificas repos.yaml      → git-aggregate → img-build → resetdb → start
Modificas addons.yaml     → img-build → resetdb → start
Modificas código Python   → restart (hot-reload automático en dev)
Modificas dependencias    → img-build → start
BD corrupta               → resetdb → start
```
