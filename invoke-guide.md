# Guía Completa de Comandos Invoke — doodba-copier-template

`invoke` (o `inv`) es la CLI que envuelve todas las operaciones del proyecto. Cada comando es una tarea definida en `tasks.py`. Internamente ejecutan `docker compose` con las opciones correctas.

---

## Comandos por fase de trabajo

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

**Nota:** Este comando es prerequisito de `git-aggregate`, `closed-prs` y otros — Invoke lo ejecuta automáticamente antes si no se ha corrido.

---

#### `inv git-aggregate`
**Qué hace:** Descarga todos los repos Git externos definidos en `repos.yaml`.
```bash
inv git-aggregate
```
**Internamente ejecuta:**
1. `docker compose --file setup-devel.yaml run --rm odoo` → arranca un contenedor temporal que ejecuta `gitaggregate`
2. Clona/actualiza cada repo listado en `odoo/custom/src/repos.yaml`
3. Aplica merges de PRs específicos si están definidos
4. Regenera el `.code-workspace`
5. Instala/desinstala `pre-commit` en cada subrepo según tenga o no configuración

**Cuándo:**
- Primera vez, después de `inv develop`
- Cada vez que modificas `repos.yaml` (añades un repo nuevo, cambias un merge, etc.)
- Periódicamente para actualizar repos OCA/externos a sus últimos commits

**Nota:** Respeta `UID/GID` del host para que los archivos clonados tengan los permisos correctos.

---

### DÍA A DÍA EN DESARROLLO

#### `inv start`
**Qué hace:** Levanta todos los contenedores del entorno.
```bash
inv start                    # Levantar en background (detach)
inv start --no-detach        # Levantar en primer plano (ves logs en directo)
inv start --debugpy           # Levantar con debugpy habilitado para VSCode
inv start --port-prefix 17    # Usar puertos 17069, 17025... en vez de los por defecto
```
**Internamente ejecuta:**
1. `docker compose up -d`
2. Si `--debugpy`: desactiva hot-reload (incompatible) y activa el debugger en puerto {VERSION}899
3. Si los contenedores ya existían sin cambios, hace un `restart` automático
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
- `inv stop --purge` → cuando quieres empezar completamente de cero (nuevo entorno limpio)

---

#### `inv restart`
**Qué hace:** Reinicia los contenedores de Odoo rápidamente.
```bash
inv restart              # Reinicio rápido (timeout 0)
inv restart --no-quick   # Reinicio graceful (espera a que Odoo cierre limpiamente)
```
**Internamente ejecuta:**
- `docker compose restart -t0 odoo odoo_proxy`

**Cuándo:**
- Después de cambios en Python que no se recargaron con hot-reload
- Después de instalar un módulo
- Cuando Odoo se queda colgado

**Nota:** Solo reinicia `odoo` y `odoo_proxy`, no la base de datos ni otros servicios.

---

#### `inv logs`
**Qué hace:** Muestra los logs de los contenedores.
```bash
inv logs                          # Últimos 10 logs + seguimiento en tiempo real
inv logs --tail 50                # Últimos 50 logs
inv logs --no-follow              # Solo muestra logs, no se queda esperando
inv logs --container odoo         # Solo logs del contenedor odoo
inv logs --container odoo,db      # Logs de odoo y db
```
**Internamente ejecuta:**
- `docker compose logs -f --tail 10`

**Cuándo:** Para depurar errores, ver tracebacks de Odoo, verificar que un módulo se instaló correctamente.

---

### GESTIÓN DE MÓDULOS

#### `inv install`
**Qué hace:** Instala módulos de Odoo en la base de datos.
```bash
inv install --modules mi_modulo              # Instalar un módulo específico
inv install --modules mod1,mod2,mod3         # Instalar varios módulos
inv install --cur-file ./odoo/custom/src/private/mi_modulo/models/ticket.py
                                              # Instalar el módulo del archivo actual
inv install --private                         # Instalar TODOS los módulos privados
inv install --core                            # Instalar todos los módulos core de Odoo
inv install --extra                           # Instalar todos los módulos extra (OCA, etc.)
inv install --enterprise                      # Instalar todos los módulos enterprise
```
**Internamente ejecuta:**
1. `docker compose stop odoo` → para Odoo
2. `docker compose run --rm odoo addons init -w {módulos}` → instala los módulos
3. No reinicia automáticamente — debes hacer `inv start` después

**Cuándo:**
- Primera vez que quieres usar un módulo nuevo
- Después de añadir un módulo a `addons.yaml`

**Nota:** En VSCode hay un botón en la statusbar "Install module" que ejecuta esto automáticamente sobre el archivo abierto.

---

#### `inv uninstall`
**Qué hace:** Desinstala módulos de Odoo.
```bash
inv uninstall --modules mi_modulo
```
**Internamente ejecuta:**
- `docker compose run --rm odoo click-odoo-uninstall -m {módulo}`

**Cuándo:** Cuando un módulo ya no se necesita en la BD de desarrollo.

---

### TESTING

#### `inv test`
**Qué hace:** Ejecuta los tests de Odoo para módulos específicos.
```bash
inv test --modules mi_modulo                  # Testear un módulo
inv test --cur-file ./ruta/al/archivo.py      # Testear el módulo del archivo actual
inv test --private                             # Testear todos los módulos privados
inv test --modules mod1,mod2 --skip mod2       # Testear mod1, saltar mod2
inv test --modules mi_modulo --debugpy         # Testear con debugger de VSCode
inv test --modules mi_modulo --mode update     # Testear en modo update (en vez de init)
inv test --modules mi_modulo --db-filter ""    # Sin filtro de BD
```
**Internamente ejecuta:**
1. Si `--debugpy`: levanta contenedor con debugpy habilitado
2. Si no: `docker compose run --rm odoo odoo --test-enable --stop-after-init --workers=0 -i {módulos}`
3. Desde Odoo 12+: añade `--test-tags` para limitar tests a los módulos explícitos

**Cuándo:**
- Antes de hacer commit (verificar que no rompes nada)
- En desarrollo TDD
- CI/CD ejecuta esto automáticamente

**Nota importante:** Después de `inv test`, Odoo queda parado. Debes ejecutar `inv start` para volver al modo normal.

---

### BASE DE DATOS

#### `inv resetdb`
**Qué hace:** Destruye y recrea la base de datos desde cero.
```bash
inv resetdb                                    # Resetea BD "devel" con módulo "base"
inv resetdb --dbname mi_bd                     # Resetea una BD específica
inv resetdb --modules mi_modulo                # Resetea e instala módulos específicos
inv resetdb --private                          # Resetea e instala todos los módulos privados
inv resetdb --dependencies --modules mi_modulo # Instala solo las dependencias del módulo
inv resetdb --no-populate                      # No ejecuta preparedb después
```
**Internamente ejecuta:**
1. `docker compose stop odoo`
2. `click-odoo-dropdb devel` → borra la BD
3. `click-odoo-initdb -n devel -m {módulos}` → crea BD nueva con módulos
4. `preparedb` → configura la BD con valores útiles para desarrollo

**Cuándo:**
- La BD de desarrollo está corrupta o llena de basura
- Quieres un entorno limpio para probar algo
- Después de cambios grandes en modelos (nuevas dependencias, reestructuración)

**Nota:** Para Odoo 19+, usa el CLI nativo de Odoo en vez de `click-odoo-initdb` por cambios internos en el Registry.

---

#### `inv preparedb`
**Qué hace:** Ejecuta el script `preparedb` dentro del contenedor.
```bash
inv preparedb
```
**Qué configura:** Valores útiles para desarrollo (parámetros del sistema, configuración base). Solo disponible desde Odoo 11+.

**Cuándo:** Se ejecuta automáticamente después de `inv resetdb`. Rara vez lo necesitas manualmente.

---

#### `inv snapshot`
**Qué hace:** Crea una copia de la BD actual (snapshot).
```bash
inv snapshot                                         # Snapshot de "devel" con timestamp
inv snapshot --source-db devel                       # Especificar BD origen
inv snapshot --destination-db mi_backup_manual       # Nombre personalizado
```
**Internamente ejecuta:**
1. Para `odoo` y `db`
2. `click-odoo-copydb devel devel-2026_02_18-14_30`
3. Reinicia servicios si estaban activos

**Cuándo:**
- Antes de hacer algo arriesgado (migración, borrado masivo, etc.)
- Para guardar un estado "bueno" al que volver

---

#### `inv restore-snapshot`
**Qué hace:** Restaura un snapshot previo.
```bash
inv restore-snapshot                                   # Restaura el último snapshot automáticamente
inv restore-snapshot --snapshot-name devel-2026_02_18   # Restaura uno específico
inv restore-snapshot --destination-db devel             # A qué BD restaurar
```
**Internamente ejecuta:**
1. Para `odoo` y `db`
2. Si no se especifica nombre: busca el snapshot más reciente por fecha
3. `click-odoo-dropdb devel` → borra BD actual
4. `click-odoo-copydb {snapshot} devel` → restaura
5. Reinicia servicios si estaban activos

**Cuándo:** Después de romper algo, para volver al estado del snapshot.

---

### CALIDAD DE CÓDIGO

#### `inv lint`
**Qué hace:** Ejecuta todos los linters y formatters configurados en pre-commit.
```bash
inv lint              # Lint normal
inv lint --verbose    # Con output detallado
```
**Internamente ejecuta:**
- `pre-commit run --show-diff-on-failure --all-files --color=always`

**Qué revisa:** Prettier (YAML, XML), Pylint, Ruff, ESLint, y todos los hooks OCA.

**Cuándo:**
- Antes de hacer push (se ejecuta automáticamente en cada `git commit` gracias a pre-commit)
- Para verificar que todo el código cumple los estándares

---

#### `inv updatepot`
**Qué hace:** Actualiza los archivos de traducción (.pot/.po) de un módulo.
```bash
inv updatepot --module mi_modulo              # Actualizar traducciones de un módulo
inv updatepot --all                            # Todos los módulos
inv updatepot --repo server-tools              # Todos los módulos de un repo
inv updatepot --module mi_modulo --no-msgmerge # Sin merge automático a .po existentes
```
**Internamente ejecuta:**
1. Para Odoo
2. `click-odoo-makepot` → genera/actualiza .pot
3. Limpia archivos temporales y fechas de los .po
4. Ejecuta pre-commit sobre los archivos modificados

**Cuándo:** Después de añadir/modificar strings traducibles en tu módulo.

---

### SCAFFOLDING

#### `inv scaffold`
**Qué hace:** Crea la estructura base de un módulo nuevo de Odoo.
```bash
inv scaffold mi_modulo                                           # En directorio actual
inv scaffold mi_modulo --path odoo/custom/src/private             # En path específico
```
**Internamente ejecuta:**
- `docker compose run --rm odoo odoo scaffold {nombre} {path}`

**Qué genera:**
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

**Cuándo:** Al empezar un módulo nuevo. Es lo que usaste para crear `modulo_pablo_ticket`.

---

### UTILIDADES

#### `inv write-code-workspace-file`
**Qué hace:** Regenera el archivo `.code-workspace` de VSCode.
```bash
inv write-code-workspace-file
inv write-code-workspace-file --cw-path doodba.custom.code-workspace
```
**Qué configura:**
- Carpetas del workspace (una por cada subrepo en `src/`)
- Configuración de Python (linting, paths, formatter)
- Configuraciones de debug (debugpy, Firefox, Chrome)
- Tasks de VSCode (Start, Stop, Restart, Install, Test, Logs)

**Cuándo:** Se ejecuta automáticamente con `inv develop` y `inv git-aggregate`. Solo manualmente si añades repos sin usar `git-aggregate`.

---

#### `inv after-update`
**Qué hace:** Acciones post-actualización del template Copier.
```bash
inv after-update
```
**Internamente:** Ajusta permisos de scripts de build según la versión de Odoo. Limpia archivos obsoletos.

**Cuándo:** Se ejecuta automáticamente después de `copier copy/update`. Nunca lo ejecutas manualmente.

---

#### `inv closed-prs`
**Qué hace:** Comprueba si algún PR referenciado en `repos.yaml` ha sido cerrado.
```bash
inv closed-prs
```
**Cuándo:** Periódicamente, para detectar PRs mergeados o rechazados que ya no necesitas en tu configuración de merges.

---

## Flujo inicial completo (después de Copier)

```
copier copy gh:Halltic/doodba-copier-template /opt/odoo/proyecto --trust
  └── (automático) inv after-update + inv develop

cd /opt/odoo/proyecto
# Abrir el workspace .code-workspace generado en VSCode

inv git-aggregate      # 1. Descargar código (repos.yaml → odoo/custom/src/)
inv img-build          # 2. Construir imagen Docker con el código descargado
inv resetdb            # 3. Crear base de datos (necesita la imagen construida)
inv start              # 4. Arrancar entorno (necesita imagen + BD)
```

**¿Por qué ese orden?** Es una cadena de dependencias:
- `git-aggregate` descarga el código → sin él, la imagen se construye vacía
- `img-build` mete el código en la imagen Docker (triggers ONBUILD del Dockerfile)
- `resetdb` ejecuta Odoo dentro de la imagen para crear tablas en PostgreSQL
- `start` levanta todo: ya tiene imagen, código y base de datos

**¿Cuándo repetir cada paso?**
| Cambio realizado | Repetir desde |
|---|---|
| Modificas `repos.yaml` (nuevo repo/merge) | `git-aggregate` → `img-build` → `resetdb` → `start` |
| Modificas `addons.yaml` (activar addon existente) | `img-build` → `resetdb` → `start` |
| Modificas código Python en `private/` | Solo `inv restart` (hot-reload en dev) |
| Modificas vistas XML en `private/` | Actualizar módulo desde Odoo o `inv install` |
| Cambias dependencias pip/apt | `img-build` → `start` |
| BD corrupta o quieres empezar limpio | `resetdb` → `start` |

---

## Flujo típico de trabajo diario

```
1. inv start                          # Levantar entorno
2. (editar código)                    # Hot-reload recarga automáticamente
3. inv install --modules mi_modulo    # Si es un módulo nuevo
4. inv restart                        # Si hot-reload no detectó los cambios
5. inv test --modules mi_modulo       # Verificar que funciona
6. inv start                          # Volver al modo normal después de test
7. git add . && git commit            # Pre-commit ejecuta linters automáticamente
8. inv stop                           # Al terminar el día
```

## Flujo de emergencia (algo se rompió)

```
1. inv logs --container odoo           # Ver qué error hay
2. inv restart                         # Intentar reiniciar
3. inv snapshot                        # Si funciona, guardar estado antes de tocar
4. inv resetdb                         # Nuclear: empezar de cero si nada funciona
```

## Flujo para nuevo repo externo

```
1. (editar repos.yaml y addons.yaml)
2. inv git-aggregate                   # Descarga el repo
3. inv start                           # Levanta con el nuevo código
4. inv install --modules nuevo_addon   # Instala el módulo
```
