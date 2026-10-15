# HACKATHON TALENT4CYBER 2026: DÍA 2

![Talent4Cyber](https://img.shields.io/badge/Talent4Cyber-2026-blue)
![Día](https://img.shields.io/badge/D%C3%ADa-2%20de%202-orange)
![MCCE](https://img.shields.io/badge/MCCE-Ministerio%20de%20Defensa-green)

## Sobre el evento

**Talent4Cyber 2026** es la tercera edición del primer **Attack & Defense Hackathon universitario nacional de ciberseguridad**, una iniciativa del **Ministerio de Defensa de España**, a través del **Mando Conjunto del Ciberespacio (MCCE)**, en colaboración con la **Universidad de Granada**, la **Universidad Politécnica de Cartagena** y **Trustlab**.

El objetivo es acercar a la comunidad universitaria a la misión de los **Guardianes del Ciberespacio**. Durante dos días, los mejores talentos universitarios se enfrentan a desafíos técnicos en un escenario donde la **guerra electrónica** y las **infraestructuras marítimas** marcan la diferencia.

**Misión:** Defender el tráfico marítimo del Estrecho y sus infraestructuras esenciales para la defensa mediante estrategias de ataque y defensa en tiempo real.

- **Fechas:** 14 y 15 de octubre de 2026
- **Horario:** Día 2: de 9:00 a 17:00
- **Formato:** Attack & Defense (KotH)
- **Participación:** Equipos de 4 personas de la misma universidad
- **Premio:** Participar en **Locked Shields 2027**, organizado por el **CCDCOE** (OTAN)

Más información: [https://talent4cyber.es/](https://talent4cyber.es/)

---

## Datos del equipo

| Campo | Valor |
|-------|-------|
| **Nombre del equipo** | Brigada Bardino |
| **Universidad** | Universidad Europea de Madrid |
| **Miembro** | marcos |
| **Puntos completados (Día 2)** | 146 |
| **Puntos totales del evento (individual)** | 279 |
| **Tiempo en completarlo** | 7h 24m 54s 475ms |

---

## Operación Estrecho Fantasma

> *"03:31Z. Un buque a 361 nudos. Ningún barco del mundo hace eso. Pero ahí está, en las pantallas del control de tráfico marítimo de Algeciras. Rumbo 044. Doce millas náuticas en dos minutos. Sin eco de radar que lo acompañe y sin figurar en ningún registro de buques autorizados.*
>
> *Esa misma madrugada, alguien ha bajado la potencia del cable submarino que conecta dos continentes. No lo ha cortado: lo ha degradado. Un corte se ve. Una degradación se discute.*
>
> *Por el Estrecho de Gibraltar pasan hasta trescientos buques al día y un ferry cargado de pasaje cada media hora. Todos se fían de unos instrumentos que, desde esa noche, mienten.*
>
> *Tienes dos días para averiguar quién y cómo."*

### Día 2: Decidir

El AIS del corredor, el firmware de una baliza, la estación del cable, el panel del puerto. Y luego la parte que no se resuelve con una shell: aislar sin dejar a oscuras al tráfico civil, redactar el comunicado sin filtrar lo que no debe salir, y firmar una decisión bajo reglas de enfrentamiento.

**Redes, Reversing, Criptografía, Forense, Web, Contención, Comunicación y Decisión ROE.**

---

## Retos resueltos (Día 2)

| # | Reto | Categoría | Puntos | Flag / Solución |
|---|------|-----------|--------|------------------|
| 01 | [Cripto del Convoy](Cripto%20del%20Convoy.md) | Criptografía | 20 | `T4C{3cds4_n0nc3_r3us3_c0nv0y}` |
| 02 | [Aislar el Segmento: Contención](Aislar%20el%20Segmento%20-%20Contención.md) | Contención | 20 | `T4C{s3gm3nt0_0T_41sl4d0_0k}` |
| 03 | [Cable Roto](Cable%20Roto.md) | Forense | 15 | `T4C{20260214T031744Z_sess_a91f4c7e}` |
| 04 | [Eco en el AIS](Eco%20en%20el%20AIS.md) | Redes | 20 | `T4C{MMSI_224815920_FANTASMA}` |
| 05 | [Implante en el Faro](Implante%20en%20el%20Faro.md) | Reversing | 25 | `T4C{4rm_1mpl4nt3_f4r0_tr1gg3r}` |
| 06 | [Panel del Puerto](Panel%20del%20Puerto.md) | Web | 20 | `T4C{ssrf_l00pb4ck_0t_d1syunt0r}` |
| 07 | [Comunicado de Crisis](Comunicado%20de%20Crisis.md) | Comunicación | — | *(Comunicado institucional)* |
| 08 | [Aislar el Segmento: Justificación](Aislar%20el%20Segmento%20-%20Justificación.md) | Decisión ROE | — | *(Justificación escrita)* |
| 09 | [Regla de Enfrentamiento](Regla%20de%20Enfrentamiento.md) | Decisión ROE | — | *(Justificación escrita)* |
| 10 | [Informe de Situación (SITREP)](Informe%20de%20Situación%20(SITREP).md) | Documentación | — | *(SITREP consolidado)* |
| 11 | [Cadena de Custodia](Cadena%20de%20Custodia.md) | Forense integrador | — | *(Timeline consolidado)* |

---

## Metodología

Todo el análisis se realizó en **entorno controlado** y de forma **estática** siempre que fue posible:

- **Criptografía:** Reutilización de nonce en ECDSA (mismo `r`), recuperación de clave privada.
- **Redes:** Análisis de capturas pcap, decodificación AIS, coherencia cinemática, checksum NMEA.
- **Forense:** Correlación de logs (SSH, configuración, métricas), análisis de volcados de memoria.
- **Reversing:** Análisis estático de binarios ARM Thumb-2, resolución algebraica de condiciones de activación.
- **Web:** SSRF, bypass de filtros, SSTI en Jinja2.
- **Contención:** Microsegmentación, reglas de firewall, proporcionalidad y distinción.
- **Comunicación:** Redacción de comunicado institucional sin indicadores técnicos.
- **Decisión ROE:** Aplicación de necesidad militar, proporcionalidad y distinción.

---

## Notas

- Este repositorio recoge los **apuntes y soluciones** del Día 2 de la hackathon.
- El **Día 1** se centró en la parte de **reconstrucción** (criptografía, OSINT, forense, reversing, pwn, web).
- No todo el material recibido era útil: parte estaba diseñado para **perder el tiempo**. Saber cuándo cerrar una línea de investigación también puntúa.
- Los retos se resolvieron en equipo, pero este repositorio es una recopilación personal para estudio y referencia.

---

## Autor

**marcos** (mnickh12)

- GitHub: [@mnickh12](https://github.com/mnickh12)

---

## Licencia

Este proyecto es solo para **fines educativos**. Las flags y soluciones son parte de una competición pasada y no deben usarse para actividades ilegales.

---

## Derechos de autor y propiedad intelectual

Aunque los enunciados de la competición y el marco del evento sean de carácter público, **todos los apuntes, soluciones, análisis, explicaciones, comandos, scripts y desarrollos recogidos en este repositorio son de mi exclusiva autoría y propiedad intelectual**.

Queda **prohibida la copia, reproducción, distribución, comunicación pública, transformación o reutilización total o parcial** de este contenido sin mi consentimiento expreso y por escrito.

Este material se comparte **únicamente con fines educativos y de referencia personal**, no como recurso libre de copia. Cualquier uso distinto requerirá autorización expresa del autor.

Para solicitar permiso, contactar a través de mi perfil de GitHub: [@mnickh12](https://github.com/mnickh12).

© 2026 marcos (mnickh12). Todos los derechos reservados.

---

> *"La ciberdefensa no se gana en solitario. En el mundo real, las operaciones de defensa y ataque requieren trabajo en equipo, coordinación y diversidad de habilidades."*
