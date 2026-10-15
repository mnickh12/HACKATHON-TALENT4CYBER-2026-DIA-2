# Cadena de Custodia — Forense integrador

## 1. Datos del reto

| Campo | Valor |
|-------|-------|
| **Nombre** | Cadena de Custodia |
| **Categoría** | Forense integrador |
| **Dificultad** | Difícil |
| **Objetivo** | Entregar un timeline consolidado de las dos jornadas, en orden cronológico, que integre la evidencia de los retos 4, 5, 7, 12 y 13 |
| **Extensión máxima** | 800 palabras |
| **Formato de entrega** | Sin flag. Se entrega por formulario y se puntúa por rúbrica |

---

## 2. Enunciado de la actividad

### El mando pide la reconstrucción completa

No basta con haber resuelto los retos: hay que poder **demostrar cómo se llegó a cada conclusión**. Un hallazgo sin trazabilidad no sirve para respaldar una decisión, y menos aún una atribución.

**Misión:** entregar un timeline consolidado de las dos jornadas, en orden cronológico, que integre la evidencia de los retos 4, 5, 7, 12 y 13.

### Requisitos

1. Cada entrada indica el **momento**, el **hecho** y el **artefacto concreto** que lo respalda: fichero, marca temporal, hash, ruta o identificador.
2. Distingue lo **observado** de lo **inferido** y marca lo que **no está confirmado**.
3. Señala al menos **una correlación entre el Día 1 y el Día 2**.

**Máximo 800 palabras.** Lo que pase del límite no se corrige.

**Formato de entrega:** Sin flag. Se entrega por formulario y se puntúa íntegramente por rúbrica.

---

## 3. Desarrollo — fuentes utilizadas por reto

### Reto 4 — Correo con Carga (PIR-04, Forense)

Correo con remitente falsificado (`sistemas@mli-iberia.example`) que en realidad se originó en `transportes-delta.example` desde la IP `185.203.117.44`, acreditado por las cabeceras `Received` (leídas de abajo arriba) y por `spf=fail` / `dkim=fail` / `dmarc=fail` en la `Authentication-Results` del MX receptor (la única fiable).

El adjunto `.docm` contiene una macro VBA (`AutoOpen` / `Document_Open`) que ensambla seis fragmentos de texto, los invierte (`StrReverse`) y decodifica en base64 una configuración de segundo estadio, escrita en `%TEMP%\svc_upd.dat`.

Esa configuración incluye `canal=cdn.transportes-delta.example`, `intervalo=900`, `identidad=MLI-ALG-0774` y `mutex=Global\NavalSyncUpdater`.

### Reto 5 — Persistencia Silenciosa (PIR-05, Reversing)

Análisis estático de `persistencia.elf` (recuperado de `/usr/lib/naval/`). Binario ELF64, sin símbolos, que calcula un hash FNV-64 del hostname `srv-planificacion-01` y descifra un blob en `.rodata` (offset `0x2020`). El texto en claro apunta a la unidad de persistencia `/etc/systemd/system/naval-syncd.timer`.

**Advertencia:** el propio informe indica que la salida de `systemctl list-timers` pudo estar manipulada por el implante. La ruta se trata como **hipótesis reforzada** por el análisis del binario, no como hecho verificado de forma independiente.

### Reto 7 — Escalada al Núcleo (PIR-07, Pwn)

Escalada a root en `srv-planificacion-01` mediante **PATH hijacking** del binario SUID `naval-report`. Este llama por `system()` a `naval-collect` (no SUID), que a su vez invoca `date` sin ruta absoluta. Al anteponer `/tmp` al PATH antes de ejecutar el binario SUID, un `/tmp/date` falso se ejecuta con `euid=0`.

**Artefacto:** `/tmp/resultado.txt` con `uid=0(root)` y copia de `flag.out`.

**No confirmado:** los timestamps de fichero visibles en el host (`Oct 6`, `Ago 13`) no son del incidente real, sino del entorno del laboratorio, y se excluyen de la cronología.

### Reto 12 — Implante en el Faro (PIR-12, Reversing)

Análisis estático de `implante.elf` (baliza ES-TAR-04, Punta Carnero). Se determina la condición algebraica de activación (30 ecuaciones sobre bytes de entrada) y se reconstruye el código válido `12250106...fcb2`.

**Artefacto:** tablas `T` (`0x4f158`) y `C` (`0x4f178`) en `.rodata`.

**No confirmado:** no se ejecutó el binario (sin `qemu-arm`), por lo que se conoce el mecanismo de disparo pero no si llegó a activarse realmente ni en qué momento.

### Reto 13 — Cable Roto (PIR-13, Forense)

Sesión SSH `a91f4c7e` en el sistema de gestión del cable ES-MA-01, usuario `mant-optico`, desde IP externa `185.203.117.44` (la misma del correo de phishing del Día 1), a las `03:17:44Z`.

Cambios en esa sesión: reducción de `amplificador_edfa_ganancia_db` de 18,0 a 6,5 a las `03:19:12Z`; redirección de `traps_snmp_destino` de `10.12.4.9` a `0.0.0.0` a las `03:21:40Z`.

Esto coincide con una caída de potencia óptica de ≈11 dB sin causa meteorológica, recuperada a las `07:39Z`.

El volcado de memoria `memdump.bin` corrobora el comando ejecutado y etiqueta esa misma IP como `relay_c2`.

---

## 4. Resultado de la actividad — Timeline consolidado

### DÍA 1 — 2026-02-13 (acceso inicial, persistencia, escalada)

**[Hora no registrada]** — **Hecho:** correo de phishing a Sistemas MLI con adjunto `.docm`; remitente falsificado. **Artefacto:** cabeceras `Received` / `Authentication-Results` (`spf/dkim/dmarc=fail`) del `.eml`; origen real IP `185.203.117.44` / dominio `transportes-delta.example`. **OBSERVADO** (cabeceras legibles directamente). Hora exacta no confirmada en el material.

**[Inmediatamente posterior, mismo artefacto]** — **Hecho:** macro `AutoOpen` / `Document_Open` ensambla, invierte y decodifica en base64 una configuración de 2º estadio, escrita en `%TEMP%\svc_upd.dat`. **Artefacto:** `vbaProject.bin` (descomprimido a mano, MS-OVBA); campos: `canal=cdn.transportes-delta.example`, `intervalo=900`, `mutex=Global\NavalSyncUpdater`, `identidad=MLI-ALG-0774`. **OBSERVADO** (desofuscación estática reproducible).

**[Orden inferido, sin timestamp]** — **Hecho:** instalación de persistencia del implante mediante unidad systemd. **Artefacto:** `persistencia.elf` (blob cifrado en offset `0x2020`, clave = FNV-64 del hostname `srv-planificacion-01`); descifrado estático apunta a `/etc/systemd/system/naval-syncd.timer`. **INFERIDO:** el propio informe advierte que `systemctl_list-timers.txt` pudo estar manipulado por el implante ("hipótesis, no conclusión"); la ruta se confirma por el algoritmo del binario, no de forma independiente.

**[Orden inferido, posterior a la persistencia]** — **Hecho:** escalada a root en `srv-planificacion-01` vía PATH hijacking del SUID `naval-report` → `naval-collect` (no SUID) → `/tmp/date` malicioso ejecutado como root. **Artefacto:** `/tmp/resultado.txt` con `uid=0(root)` y copia de `flag.out`. **OBSERVADO** directamente en la sesión de explotación. **No confirmado:** los timestamps de fichero visibles en el host (`Oct 6`, `Ago 13`) no son del incidente, sino del entorno del laboratorio — se excluyen de la cronología real.

### DÍA 2 — 2026-02-14 (sabotaje del enlace y baliza)

**03:17:44Z** — **Hecho:** apertura de la sesión SSH `a91f4c7e` en el sistema de gestión del cable ES-MA-01, usuario `mant-optico`, desde IP externa `185.203.117.44` — la misma IP de origen que el correo de phishing del Día 1. **Artefacto:** `acceso_ssh.log`; corroborado por `memdump.bin` del proceso `gestion-cable` (cadena `185.203.117.44` etiquetada `relay_c2`, timestamp `2026-02-14T03:17:44+00:00`). **OBSERVADO** en dos fuentes independientes. **No confirmado:** que sea la misma persona/operación; `mant-optico` es un usuario legítimo y lo más probable es credencial comprometida, no identidad falsa.

**03:19:12Z** — **Hecho:** cambio de `amplificador_edfa_ganancia_db` de 18,0 a 6,5 dentro de la sesión `a91f4c7e`. **Artefacto:** `cambios_config.log`. **OBSERVADO**.

**03:21:40Z** — **Hecho:** cambio de `traps_snmp_destino` de `10.12.4.9` a `0.0.0.0` (silencia alertas), misma sesión. **Artefacto:** `cambios_config.log`. **OBSERVADO**.

**03:19Z-07:38Z** — **Hecho:** degradación de potencia Rx de ≈−21,3 dBm a ≈−32,8/−33,0 dBm (≈11 dB), sin correlato meteorológico (parte: calma, viento flojo). **Artefacto:** `metricas_enlace.csv`. **OBSERVADO/medido;** la causalidad con el cambio de ganancia es **INFERIDA** por coincidencia temporal (inicio a los pocos segundos del cambio), no por un log causal explícito.

**07:38Z-07:39Z** — **Hecho:** restablecimiento del umbral en sesión autorizada `0e93aa47` y recuperación de potencia a ≈−21,5 dBm. **Artefacto:** `cambios_config.log`, `metricas_enlace.csv`. **OBSERVADO**.

**[Sin fecha operativa]** — **Hecho:** análisis estático de `implante.elf` (baliza ES-TAR-04, Punta Carnero) determina la condición algebraica de activación (30 ecuaciones sobre bytes de entrada) y reconstruye el código válido `12250106...fcb2`. **Artefacto:** tablas `T` (`0x4f158`) y `C` (`0x4f178`) en `.rodata`. **NO CONFIRMADO:** no se ejecutó el binario (sin `qemu-arm`), por lo que se conoce el mecanismo de disparo pero no si llegó a activarse realmente ni en qué momento.

---

## 5. Correlación Día 1 ↔ Día 2

La IP `185.203.117.44`, origen del correo de phishing y de la infraestructura C2 (`transportes-delta.example`) del Día 1 (Reto 4), reaparece como origen de la sesión SSH ilegítima `a91f4c7e` que sabotea el enlace óptico el Día 2 (Reto 13, `03:17:44Z`). Es la única sesión de ese log sin cobertura en el calendario de mantenimiento.

Esto vincula la misma infraestructura de red a dos acciones distintas de la campaña, aunque **no prueba identidad del operador** — la hipótesis más probable, señalada en el propio informe del Reto 13, es **credencial comprometida**.

---

## 6. Cierre de confianza

**Observado con artefacto directo:**

- Correo/macro (R4).
- Escalada SUID (R7).
- Sesión y cambios de configuración del cable (R13).
- Mecanismo de cifrado de la baliza (R12).

**Inferido:**

- La ruta de persistencia exacta (R5).
- La causalidad temporal ganancia→degradación (R13).

**No confirmado:**

- Activación real del implante de la baliza (R12).
- Atribución personal de la sesión `a91f4c7e` (R13).

---

## 7. Texto final y solución del reto

### Día 1

Hubo un correo de phishing dirigido a Sistemas MLI con un adjunto `.docm` y el remitente falsificado como `sistemas@mli-iberia.example`. Las cabeceras `Received` y el `Authentication-Results` del `.eml` muestran `spf`, `dkim` y `dmarc` en `fail`, y el origen real apunta a la IP `185.203.117.44` y al dominio `transportes-delta.example`. Esto es **observado** directamente, porque las cabeceras son legibles y no admiten otra lectura. **Artefacto:** cabeceras del `.eml`.

Inmediatamente después, con el mismo artefacto, la macro `AutoOpen` y `Document_Open` ensambla, invierte con `StrReverse` y decodifica en base64 una configuración de segundo estadio que se escribe en `%TEMP%\svc_upd.dat`. Es **observado**, porque la desofuscación es reproducible estáticamente. **Artefacto:** `vbaProject.bin`, con los campos `canal=cdn.transportes-delta.example`, `intervalo=900`, `mutex=Global\NavalSyncUpdater` e `identidad=MLI-ALG-0774`.

Más tarde, en orden **inferido** y sin timestamp, viene la instalación de persistencia del implante mediante una unidad systemd. **Artefacto:** `persistencia.elf`, con un blob cifrado en `.rodata` en el offset `0x2020` y clave FNV-64 del hostname `srv-planificacion-01`. El descifrado estático apunta a `/etc/systemd/system/naval-syncd.timer`. Hay que ser cuidadoso aquí, porque el propio informe advierte que la salida de `systemctl list-timers` pudo estar manipulada por el implante, así que la ruta se trata como hipótesis reforzada por el análisis del binario, no como hecho verificado de forma independiente.

Después está la escalada a root en `srv-planificacion-01` mediante PATH hijacking. El binario SUID `naval-report` llama por `system` a `naval-collect`, que no es SUID y que a su vez invoca `date` sin ruta absoluta. Al anteponer `/tmp` al PATH antes de ejecutar el binario SUID, un `/tmp/date` falso se ejecuta con `euid=0`. Esto queda confirmado directamente en la explotación. **Artefacto:** `/tmp/resultado.txt` con `uid=0(root)` y copia de `flag.out`. Eso sí, los timestamps de fichero visibles en el host, que marcan `Oct 6` y `Ago 13`, no son del incidente real, sino del entorno del laboratorio, y se excluyen de la cronología.

### Día 2

A las `03:17:44Z` se abre la sesión SSH `a91f4c7e` en el sistema de gestión del cable ES-MA-01, con el usuario `mant-optico` y desde la IP externa `185.203.117.44`, que es la misma IP del correo de phishing del Día 1. Es **observado** en dos fuentes independientes. **Artefacto:** `acceso_ssh.log`, corroborado por `memdump.bin`, donde aparece la cadena `185.203.117.44` etiquetada como `relay_c2`. Lo que **no está confirmado** es que sea la misma persona o la misma operación; `mant-optico` es un usuario legítimo y lo más probable es que sea una credencial comprometida, no una identidad falsa.

A las `03:19:12Z` se cambia el `amplificador_edfa_ganancia_db` de 18,0 a 6,5 dentro de la misma sesión `a91f4c7e`. **Artefacto:** `cambios_config.log`. **Observado.**

A las `03:21:40Z` se cambia `traps_snmp_destino` de `10.12.4.9` a `0.0.0.0` para silenciar alertas, también en la misma sesión. **Artefacto:** `cambios_config.log`. **Observado.**

Entre las `03:19Z` y las `07:38Z` se produce una degradación de potencia Rx de aproximadamente −21,3 dBm a −32,8 o −33,0 dBm, unos 11 dB, sin correlato meteorológico porque la parte indica calma y viento flojo. **Artefacto:** `metricas_enlace.csv`. Es **observado o medido**, pero la causalidad con el cambio de ganancia es **inferida** por coincidencia temporal, no por un log causal explícito.

Entre las `07:38Z` y las `07:39Z` se restablece el umbral en una sesión autorizada `0e93aa47` y la potencia se recupera a unos −21,5 dBm. **Artefacto:** `cambios_config.log` y `metricas_enlace.csv`. **Observado.**

Aparte está el análisis estático de `implante.elf` de la baliza ES-TAR-04 en Punta Carnero, que determina la condición algebraica de activación con 30 ecuaciones sobre bytes de entrada y reconstruye el código válido `12250106...fcb2`. **Artefacto:** tablas `T` en `0x4f158` y `C` en `0x4f178` dentro de `.rodata`. Esto **no está confirmado**, porque no se pudo ejecutar el binario al no haber `qemu-arm`, así que se conoce el mecanismo de disparo pero no si llegó a activarse ni en qué momento.

La correlación entre los dos días está clara. La IP `185.203.117.44`, que es el origen del correo de phishing y de la infraestructura C2 `transportes-delta.example` del Día 1 en el Reto 4, reaparece como origen de la sesión SSH ilegítima `a91f4c7e` que sabotea el enlace óptico el Día 2 en el Reto 13. Es la única sesión de ese log sin cobertura en el calendario de mantenimiento, lo que vincula la misma infraestructura de red a dos acciones distintas de la campaña, aunque no prueba la identidad del operador. La hipótesis más probable, como señala el propio informe del Reto 13, es **credencial comprometida**.

---

## 8. Lecciones aprendidas

- **La trazabilidad es esencial.** Un hallazgo sin artefacto que lo respalde no sirve para respaldar una decisión.
- **Distinguir observado, inferido y no confirmado** es clave para no sobrestimar la evidencia.
- **La correlación entre días** (misma IP en phishing y sabotaje) refuerza la hipótesis de una campaña coordinada.
- **La atribución requiere cautela.** Una IP compartida no prueba identidad; puede ser credencial comprometida.
- **Los timestamps del laboratorio** no son del incidente real y deben excluirse de la cronología.
- **La cadena de custodia** garantiza que la evidencia es admisible y que las conclusiones son defendibles.
