# Cable Roto — PIR-13

## 1. Datos del reto

| Campo | Valor |
|-------|-------|
| **Nombre** | PIR-13. Acotar la ventana de manipulación del enlace |
| **Categoría** | Forense |
| **Caso** | IR-2026-0214-CABLE |
| **Enlace** | Algeciras–Ceuta (fibra submarina) |
| **Periodo** | 2026-02-13 00:00Z – 2026-02-14 12:00Z |
| **Puntos** | 15 pts |
| **Formato de entrega** | `T4C{...}` |
| **Flag** | `T4C{20260214T031744Z_sess_a91f4c7e}` |

---

## 2. Conclusión

La sesión intrusa es `a91f4c7e`, abierta a las **2026-02-14T03:17:44Z** con el usuario `mant-optico` desde la IP externa **185.203.117.44**. Es la única sesión sin cobertura en el calendario de mantenimiento y en ella se hacen los dos cambios que degradan el enlace y silencian las alertas. La sesión duró hasta las 03:24:09Z, pero la degradación se mantuvo hasta las 07:39Z.

---

## 3. Método

El enunciado advierte de tres distractores: accesos nocturnos legítimos, cambios de configuración inocuos y una degradación ambiental. El método fue cruzar las fuentes y descartar lo que tiene explicación.

### 3.1. Línea temporal de accesos (`acceso_ssh.log`)

| Sesión | Usuario | Origen | Inicio (Z) | Fin (Z) |
|--------|---------|--------|------------|---------|
| 3b7e2201 | mant-optico | 10.12.4.31 | 13 feb 09:02:11 | 09:41:03 |
| c410dd93 | supervisor | 10.12.4.12 | 13 feb 14:20:55 | 14:26:40 |
| 7f0a5b64 | mant-optico | 10.12.4.31 | 13 feb 22:48:09 | 23:31:52 |
| d2c88e15 | supervisor | 10.12.4.12 | 14 feb 01:55:12 | 02:04:38 |
| **a91f4c7e** | **mant-optico** | **185.203.117.44 (externa)** | **14 feb 03:17:44** | **03:24:09** |
| 0e93aa47 | mant-optico | 10.12.4.31 | 14 feb 07:10:02 | 07:52:17 |

### 3.2. Contraste con el calendario (`calendario_mantenimiento.csv`)

| Ventana aprobada | Responsable | Sesión que cubre |
|------------------|-------------|------------------|
| 13 feb 09:00–10:00 | mant-optico | 3b7e2201 |
| 13 feb 14:00–15:00 | supervisor | c410dd93 |
| 13 feb 22:30 – 14 feb 00:00 | mant-optico | 7f0a5b64 |
| 14 feb 01:30–02:30 | supervisor | d2c88e15 |
| 14 feb 07:00–08:00 | mant-optico | 0e93aa47 |

Cinco de las seis sesiones están dentro de una ventana aprobada, con el usuario correcto y desde la red interna. La de las 03:17Z queda entre dos ventanas y **no tiene cobertura**. Las dos sesiones nocturnas (`7f0a5b64`, `d2c88e15`) son legítimas: la hora por sí sola no incrimina.

### 3.3. Cambios de configuración (`cambios_config.log`)

| Hora (Z) | Sesión | Parámetro | Cambio | Efecto en el enlace |
|----------|--------|-----------|--------|---------------------|
| 13 feb 09:14 | 3b7e2201 | umbral_alarma_potencia_dbm | −24,0 → −24,5 | Ninguno (umbral de alarma) |
| 13 feb 09:33 | 3b7e2201 | intervalo_muestreo_s | 60 → 60 | Ninguno (sin cambio) |
| 13 feb 23:02 | 7f0a5b64 | perfil_compensacion_dispersion | auto → auto | Ninguno (sin cambio) |
| 13 feb 23:19 | 7f0a5b64 | retencion_registros_dias | 30 → 90 | Ninguno (retención de registros) |
| **14 feb 03:19:12** | **a91f4c7e** | **amplificador_edfa_ganancia_db** | **18,0 → 6,5** | **Reduce la potencia óptica** |
| **14 feb 03:21:40** | **a91f4c7e** | **traps_snmp_destino** | **10.12.4.9 → 0.0.0.0** | **Descarta las alertas SNMP** |
| 14 feb 07:38 | 0e93aa47 | umbral_alarma_potencia_dbm | −24,5 → −24,0 | Restablecimiento aprobado |

### 3.4. Métricas del enlace (`metricas_enlace.csv`, 2.161 muestras por minuto)

| Tramo | Potencia Rx (dBm) | Latencia | Interpretación |
|-------|-------------------|----------|----------------|
| Línea base (13 feb 00:00–20:14) | −21,3 a −21,5 | ≈ 4,82 ms | Normal |
| 13 feb 20:15–21:30 | −23,1 a −23,3 | ≈ 4,82 ms | Atenuación ambiental (≈ 2 dB), explicada por el parte meteorológico |
| 14 feb 03:19–07:38 | −32,8 a −33,0 | ≈ 7,9 ms | Degradación de unos 11 dB, sin causa ambiental |
| Desde 14 feb 07:39 | ≈ −21,5 | ≈ 4,82 ms | Recuperación |

La caída empieza a las 03:19Z, justo tras el cambio de ganancia del EDFA (03:19:12Z), y se recupera a las 07:39Z, tras el restablecimiento de la ventana de las 07:00Z. El parte meteorológico explica solo la atenuación del día 13 entre las 20:15Z y las 21:30Z; el 14 de febrero a las 03:00Z consta calma y viento flojo.

### 3.5. Corroboración en el volcado de memoria (`memdump.bin`)

El volcado del proceso de gestión (`gestion-cable`) contiene cadenas del comando ejecutado por `mant-optico`: `amplificador_edfa_ganancia_db=6.5; traps_snmp_destino=0.0.0.0`, la IP `185.203.117.44` marcada como `relay_c2` y la marca temporal `2026-02-14T03:17:44+00:00`. Coincide con el inicio de la sesión `a91f4c7e` en el log SSH.

---

## 4. Descartes

- **Sesiones nocturnas `7f0a5b64` y `d2c88e15`:** dentro de ventanas aprobadas, origen interno y cambios sin efecto en la potencia.
- **Atenuación del 13 feb 20:15–21:30Z:** origen ambiental documentado (≈ 2 dB, transitoria), muy inferior a la manipulación.
- **Cambios de umbral, muestreo, compensación y retención:** no alteran la potencia óptica.
- **Sesión `0e93aa47`:** restablece los umbrales en ventana aprobada y no causa la degradación.

---

## 5. Matiz sobre atribución

El usuario `mant-optico` es legítimo, pero la sesión usa su clave pública desde una IP externa. Lo más probable es que sus credenciales o su clave estén comprometidas. Esto **no demuestra que la persona sea responsable**, por lo que conviene revocar la clave y revisar de dónde se filtró.

---

## 6. Recomendaciones

1. Revocar y rotar la clave de `mant-optico`.
2. Bloquear el acceso SSH desde fuera de `10.12.4.0/24` y registrar el origen `185.203.117.44`.
3. Restaurar `traps_snmp_destino=10.12.4.9` y comprobar que la ganancia del EDFA está en 18,0 dB.
4. Añadir una alerta de acceso fuera de calendario y de cambios en ganancia o SNMP.
5. Enviar los traps SNMP a un destino adicional que no pueda cambiarse desde la misma sesión.

---

## 7. Lecciones aprendidas

- **Cruzar fuentes es clave.** El log SSH, el calendario, los cambios de configuración y las métricas del enlace permiten reconstruir la ventana de manipulación.
- **La hora por sí sola no incrimina.** Las sesiones nocturnas legítimas son un distractor habitual.
- **Los cambios inocuos también son distractores.** Hay que centrarse en los que afectan a la potencia o a las alertas.
- **La degradación ambiental puede explicar una atenuación**, pero no una caída de 11 dB.
- **El volcado de memoria corrobora la línea temporal.** Las cadenas del proceso de gestión coinciden con el inicio de la sesión intrusa.
- **La atribución requiere cautela.** El usuario legítimo puede haber sido suplantado; hay que revocar credenciales, no culpar a la persona.
