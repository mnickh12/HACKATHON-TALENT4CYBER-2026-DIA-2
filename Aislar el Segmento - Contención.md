# Aislar el Segmento: Contención — PIR-16

## 1. Datos del reto

| Campo | Valor |
|-------|-------|
| **Nombre** | ORDOP. Contención del segmento comprometido |
| **Categoría** | Contención |
| **Puntos** | 20 pts |
| **Objetivo** | Aislar la baliza comprometida sin interrumpir los servicios críticos |
| **Formato de entrega** | `T4C{...}` |
| **Flag** | `T4C{s3gm3nt0_0T_41sl4d0_0k}` |

---

## 2. Contexto del reto

El enunciado plantea una situación de ciberdefensa sobre la red de señalización portuaria del Estrecho. Un implante activo en una baliza de navegación obliga a aplicar **microsegmentación** para aislar la amenaza **sin interrumpir** los servicios críticos que dependen del mismo entorno de red:

- Los **ferris de pasaje** que cruzan cada media hora.
- Las **balizas de orientación nocturna** para buques que entran por el canal.

Se debían cumplir **seis condiciones** simultáneas (tres de contención y tres de continuidad operativa) y superar una comprobación automática que devuelve el flag en formato `T4C{...}`.

---

## 3. Reconocimiento inicial

El enunciado indicaba que el conjunto de reglas se enviaba a `/reto16/api/reglas`. Los primeros intentos con `GET` devolvieron:

- **405 Method Not Allowed** → el endpoint solo acepta `POST`.
- **400 Bad Request** → el cuerpo debía ser una **lista JSON** de reglas.

El error del servidor fue clave:

    "se espera una lista de como mucho 50 reglas"

Posteriormente, un `POST` con `[]` permitió descubrir el formato exacto de cada regla. La consulta a `/reto16/api/red` devolvió la topología completa de la red.

### 3.1. Segmentos identificados

| Segmento | Red | Descripción |
|----------|-----|-------------|
| **CORP** | `10.10.0.0/24` | Gestión portuaria (TI) |
| **OT-BALIZA** | `10.30.9.0/24` | Balizas de navegación (contiene la comprometida) |
| **OT-SENAL** | `10.30.7.0/24` | Señalización portuaria (soporta tráfico civil) |
| **CIVIL** | `10.40.1.0/24` | Consola de tráfico civil (solo lectura) |

### 3.2. Hosts relevantes

- `10.30.9.14` → **baliza ES-TAR-04 — COMPROMETIDA (implante activo)**
- `10.30.9.15 / .16 / .17` → balizas sanas
- `10.30.7.20` → controlador de señalización (del que depende el tráfico civil)
- `10.30.7.44` → panel de disyuntores OT
- `10.30.7.61` → repetidor VHF canal 16
- `10.30.7.80` → gateway NMEA de señalización
- `10.10.0.5` → salto-mantenimiento (bastión forense)
- `10.10.0.31` → estación de gestión portuaria
- `10.40.1.10` → consola de tráfico civil
- `10.40.1.22` → radar costero del Estrecho

### 3.3. Formato de regla

    {
      "origen": "CIDR o IP/32",
      "destino": "CIDR o IP/32",
      "accion": "permitir | denegar"
    }

> **Regla de oro:** *"la primera regla que encaja decide; sin coincidencias, permitido"*. El firewall tiene política por defecto `permit`, por lo que el aislamiento debe hacerse con reglas `denegar` explícitas y **el orden importa**.

---

## 4. Lógica de la contención

Del enunciado se deducen las seis condiciones a cumplir:

**Condiciones de contención (aislar la amenaza):**

1. La baliza comprometida no puede alcanzar el controlador de señalización.
2. La baliza comprometida no puede alcanzar el panel de disyuntores.
3. La baliza comprometida no puede alcanzar la red de gestión portuaria.

**Condiciones de continuidad (mantener servicios):**

4. El tráfico civil debe seguir recibiendo señalización (ferris).
5. El bastión forense debe poder acceder a la baliza comprometida para preservar evidencia.
6. Las balizas sanas deben poder comunicarse con la señalización.

---

## 5. Reglas aplicadas

El `POST` final enviado a `/reto16/api/reglas` fue:

    fetch('/reto16/api/reglas', {
      method: 'POST',
      headers: {'Content-Type': 'application/json'},
      credentials: 'include',
      body: JSON.stringify([
        // 1. Permite al bastión forense acceder a la baliza comprometida
        {"origen": "10.10.0.5/32", "destino": "10.30.9.14/32", "accion": "permitir"},

        // 2. La baliza comprometida no puede salir a ningún destino
        {"origen": "10.30.9.14/32", "destino": "0.0.0.0/0", "accion": "denegar"},

        // 3. Nadie puede entrar en la baliza comprometida (salvo el bastión, por orden)
        {"origen": "0.0.0.0/0", "destino": "10.30.9.14/32", "accion": "denegar"},

        // 4. Permitir explícitamente que las balizas sanas sigan comunicándose
        // con el controlador de señalización (ferris / tráfico civil)
        {"origen": "10.30.9.0/24", "destino": "10.30.7.20/32", "accion": "permitir"}
      ])
    })

**Puntos clave del diseño:**

- El **orden** de las reglas respeta la semántica "first match wins": el `permitir` al bastión precede al `denegar` general hacia la baliza.
- La baliza comprometida queda **bidireccionalmente aislada** (no sale ni recibe tráfico no autorizado).
- El **tráfico civil** y las **balizas sanas** conservan su operación.
- No se emplea un `denegar todo` global, que habría aislado la amenaza pero roto la continuidad operativa.

---

## 6. Verificación automática

El servidor devolvió `Status: 200` con el siguiente resultado:

    {
      "aislado": true,
      "condiciones": [
        {"cumple": true, "destino": "10.30.7.20", "esperado": "denegado", "motivo": "la baliza comprometida no debe alcanzar la señalización portuaria"},
        {"cumple": true, "destino": "10.30.7.44", "esperado": "denegado", "motivo": "la baliza comprometida no debe alcanzar el panel de disyuntores"},
        {"cumple": true, "destino": "10.10.0.31", "esperado": "denegado", "motivo": "la baliza comprometida no debe alcanzar la red de gestión"},
        {"cumple": true, "destino": "10.30.7.20", "esperado": "permitido", "motivo": "el tráfico civil debe seguir recibiendo señalización"},
        {"cumple": true, "destino": "10.30.9.14", "esperado": "permitido", "motivo": "el bastión forense debe conservar acceso para preservar la evidencia"},
        {"cumple": true, "destino": "10.30.7.20", "esperado": "permitido", "motivo": "la baliza sana no debe quedar aislada"}
      ],
      "estado": "reglas aplicadas al entorno comprometido",
      "flag": "T4C{s3gm3nt0_0T_41sl4d0_0k}"
    }

Las **seis condiciones** se cumplen simultáneamente, el entorno queda marcado como aislado y el servidor entrega el flag.

---

## 7. Justificación (parte de decisión)

**Necesidad militar.** Existe un implante activo en la baliza ES-TAR-04 (`10.30.9.14`) desde el que un actor hostil podría pivotar hacia el controlador de señalización y la red de gestión portuaria, comprometiendo la seguridad del canal.

**Proporcionalidad.** Se aplica una segmentación quirúrgica: se deniega el tráfico saliente y entrante de la baliza comprometida, pero se preserva el flujo del bastión forense (para preservar evidencia) y el de la consola de tráfico civil (del que dependen los ferris).

**Distinción.** Se separa el segmento comprometido (OT-BALIZA) de los servicios civiles: las balizas sanas y la señalización portuaria mantienen su operación, de modo que las balizas de orientación nocturna y el control del Estrecho no se ven afectados.

El resultado es una contención efectiva con el **mínimo daño colateral** al tráfico civil, sin recurrir a una denegación indiscriminada.

---

## 8. Conclusión

El reto se resolvió mediante un proceso de **reconocimiento de la API** (descubrimiento del formato de regla y de la topología vía `/api/red`), **análisis de la lógica de contención** (aislar sin romper servicios críticos) y **aplicación ordenada de cuatro reglas de microsegmentación**. La comprobación automática confirmó el cumplimiento de las seis condiciones y devolvió el flag:

    T4C{s3gm3nt0_0T_41sl4d0_0k}

La parte técnica quedó cerrada con este resultado. La justificación escrita se entrega por separado en el formulario correspondiente, aplicando los principios de necesidad militar, proporcionalidad y distinción.

---

## 9. Lecciones aprendidas

- **La microsegmentación es clave** para aislar una amenaza sin romper servicios críticos.
- **El orden de las reglas importa** en firewalls con semántica "first match wins".
- **La política por defecto `permit`** obliga a usar reglas `denegar` explícitas.
- **El reconocimiento de la API** (`/api/red`) revela la topología y los hosts relevantes.
- **La proporcionalidad y la distinción** son principios fundamentales en la respuesta a incidentes.
- **No se debe usar un `denegar todo` global** si rompe la continuidad operativa.
