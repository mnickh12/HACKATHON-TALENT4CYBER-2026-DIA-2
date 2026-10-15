# Eco en el AIS — PIR-11

## 1. Datos del reto

| Campo | Valor |
|-------|-------|
| **Nombre** | PIR-11. Identificar el contacto fantasma |
| **Categoría** | Redes |
| **Caso** | Corredor del Estrecho |
| **Captura** | `corredor_estrecho.pcap` (2026-02-14 02:00Z – 03:20Z, AIS por UDP 10110) |
| **Registro** | `registro_buques.csv` |
| **Puntos** | 20 pts |
| **Formato de entrega** | `T4C{...}` |
| **Flag** | `T4C{MMSI_224815920_FANTASMA}` |

---

## 2. Conclusión

El buque fantasma es el **MMSI 224815920**. No figura en el registro de buques con derecho a transitar el corredor y solo emite **dos mensajes de posición**, ambos con checksum NMEA correcto (la trama está bien formada).

Lo que no cuadra es la **cinemática**: declara 12 nudos, pero entre sus dos reportes la posición avanza lo que correspondería a unos **361 nudos**, la misma velocidad que vio el VTS.

---

## 3. Método

### 3.1. Extracción de tramas

- La captura es un pcap clásico (little-endian, enlace Ethernet) con **322 paquetes**.
- De cada paquete se extrajo la sentencia `!AIVDM` en claro, que es NMEA 0183 sobre UDP.

### 3.2. Descarte del ruido (checksum NMEA)

El operador avisa de que siempre hay tramas con la suma de verificación mal calculada. Se calculó el XOR de todos los caracteres entre `!` y `*` y se comparó con el valor de la trama:

| Resultado | Tramas |
|-----------|--------|
| Checksum correcto | 319 |
| Checksum incorrecto | 3 |

Se decodificaron también las 3 tramas erróneas para comprobar que no ocultaban nada. Son posiciones normales de buques del registro (`263740021` dos veces y `224553110` una), coherentes con su derrota, así que son **ruido de radio** y no hallazgo.

### 3.3. Decodificación AIS

Para cada trama se pasó el payload ASCII de 6 bits a bits y se extrajo, en mensajes de tipo 1/2/3:

- **MMSI:** bits 8–37
- **SOG** (velocidad sobre el fondo): bits 50–59, décimas de nudo
- **Longitud y latitud:** bits 61–88 y 89–115, en 1/10000 de minuto, con signo
- **COG** (rumbo): bits 116–127, décimas de grado

### 3.4. Contraste con el registro

| MMSI | Nombre | Reportes en la captura | En el registro |
|------|--------|------------------------|----------------|
| 224107930 | CABO DE GATA | 80 | Sí |
| 224553110 | PUNTA EUROPA | 80 | Sí |
| 225880410 | ALBORAN DOS | 80 | Sí |
| 263740021 | SERRA DA ESTRELA | 80 | Sí |
| **224815920** | — | **2** | **No** |

### 3.5. Coherencia cinemática

Para cada MMSI se ordenaron los reportes por tiempo y se calculó la **velocidad implícita** entre reportes consecutivos (distancia entre posiciones dividida entre el tiempo transcurrido). Después se comparó con el **SOG declarado**:

| MMSI | SOG declarado | Velocidad máxima implícita |
|------|---------------|----------------------------|
| 224107930 | 11,5 kn | 11,5 kn |
| 224553110 | 14,0 kn | 14,0 kn |
| 225880410 | 6,5 kn | 6,5 kn |
| 263740021 | 16,0 kn | 16,0 kn |
| **224815920** | **12,0 kn** | **≈ 361 kn** |

**Reportes del MMSI 224815920:**

| Hora (Z) | Latitud | Longitud | SOG | COG |
|----------|---------|----------|-----|-----|
| 03:12:00 | 36,000° N | 5,400° O | 12,0 kn | 090° |
| 03:14:00 | 36,000° N | 5,152° O | 12,0 kn | 090° |

Entre ambos reportes hay **0,248° de longitud**, que a 36° N son unas **12 millas náuticas**, recorridas en **120 s**. Eso da **≈ 361 kn**, frente a los 12 kn que declara el propio mensaje.

Los cuatro buques del registro cumplen su velocidad declarada.

---

## 4. Descartes

- **Tramas con checksum erróneo:** 3 tramas, todas de buques autorizados y con posiciones coherentes. Ruido de radio, como avisaba el operador.
- **Buques del registro:** su velocidad implícita coincide con la declarada durante toda la captura.
- **Formato de la inyección:** las dos tramas del fantasma **pasan el checksum**, así que el formato no la delata.

---

## 5. Limitaciones

- El contacto que el VTS vio a las **03:31Z** queda **fuera de la captura**, que termina a las 03:20Z. Los dos reportes de `224815920` (03:12Z y 03:14Z) son los únicos del buque en la captura.
- La conclusión se apoya en dos cosas a la vez: el MMSI **no está en el registro** y su movimiento es **físicamente imposible** a la velocidad que declara.

---

## 6. Recomendaciones

1. Filtrar en el VTS los MMSI que no estén en el **registro autorizado** del corredor.
2. Añadir una **alerta de coherencia SOG/posición**: marcar cualquier contacto cuya velocidad implícita difiera mucho de la declarada.
3. No tratar el checksum como prueba de autenticidad: **AIS no autentica el origen** de los mensajes.
4. Contrastar contactos AIS con **eco radar** antes de ordenar maniobras evasivas.

---

## 7. Lecciones aprendidas

- **El checksum NMEA solo verifica la integridad de la trama, no la autenticidad del emisor.** Una trama bien formada puede ser maliciosa.
- **La coherencia cinemática es una prueba clave.** Una velocidad implícita imposible delata una inyección.
- **El contraste con el registro de buques** permite descartar contactos no autorizados.
- **El ruido de radio** (checksum erróneo) es un distractor habitual que hay que descartar sin confundirlo con un hallazgo.
- **La ventana temporal de la captura** puede dejar fuera el contacto principal; hay que ser consciente de las limitaciones.
- **AIS no está autenticado.** Cualquiera puede inyectar mensajes con un MMSI falso.
