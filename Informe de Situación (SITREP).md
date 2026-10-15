# Informe de Situación (SITREP) — Cierre de la Operación Estrecho Fantasma

## 1. Datos del reto

| Campo | Valor |
|-------|-------|
| **Nombre** | Informe de Situación (SITREP) |
| **Categoría** | Documentación |
| **Dificultad** | Fácil |
| **Objetivo** | Entregar un SITREP consolidado que resuma la operación e integre las decisiones de los retos 8, 16 y 19 |
| **Extensión máxima** | 400 palabras |
| **Penalización** | -2 puntos si las decisiones entran en conflicto entre sí y no se explica |
| **Formato de entrega** | Sin flag. Se entrega por formulario y se puntúa íntegramente por rúbrica |

---

## 2. Contexto

La Operación Estrecho Fantasma llega a su último elemento. No hay flag final: lo que se entrega es un **SITREP consolidado** que resume la operación e integra las decisiones tomadas en los retos 8, 16 y 19.

El objetivo no es demostrar trazabilidad (eso ya se hizo en el Reto 17) sino **sintetizar el estado de la operación** para que un mando pueda decidir leyendo solo el primer párrafo.

El enunciado impone **cinco secciones** (situación, impacto, acciones, recomendación y riesgo residual), un máximo de **400 palabras**, y una penalización de **2 puntos** si las decisiones entran en conflicto entre sí y no se explica. **Reconocer la tensión puntúa.**

---

## 3. Las tres decisiones que hay que integrar

### Reto 8 — Contención del proceso comprometido

La decisión del Reto 8 fue **contención parcial**: no aislar por completo el servidor afectado, manteniendo el servicio legítimo activo para evitar pérdida de servicio. Es una decisión de **proporcionalidad**: se contiene el proceso comprometido, pero no se corta todo lo que depende de él.

### Reto 16 — Aislamiento del segmento OT

La decisión del Reto 16 fue **aislar el segmento OT**. Aquí el criterio es la contención del segmento comprometido, impidiendo que el implante alcance la red de gestión, el panel de disyuntores o la señalización portuaria. El aislamiento se aplica al segmento, no a toda la red, y respeta el tráfico civil y las balizas sanas.

### Reto 19 — Regla de enfrentamiento

La decisión del Reto 19 fue fijar una **regla de enfrentamiento** basada en necesidad militar, proporcionalidad y distinción: neutralizar la amenaza sin dañar a terceros ni escalar el conflicto. Es la decisión que enmarca las otras dos y les da coherencia.

---

## 4. La tensión entre los retos 8 y 16

Aquí está el punto crítico del ejercicio. El Reto 8 defendió explícitamente **no aislar del todo** para no perder el servicio legítimo. El Reto 16 aplicó **aislamiento del segmento OT**. Aparentemente, son decisiones contradictorias.

La resolución es que **no hay contradicción real**, porque:

- La contención parcial del Reto 8 se refería al **proceso comprometido**, no al segmento completo.
- El aislamiento del Reto 16 se aplica al **segmento**, pero no corta el tráfico civil ni las balizas sanas.

Es decir, el Reto 8 contiene el proceso sin cortar el servicio, y el Reto 16 aísla el segmento sin cortar el tráfico crítico. Ambas decisiones son compatibles porque actúan sobre **objetos distintos** (proceso vs. segmento) y respetan el mismo criterio (**proporcionalidad**).

El enunciado dice que "si esas decisiones entran en conflicto entre sí, explícalo" y que "reconocer la tensión puntúa". Por eso el SITREP menciona explícitamente la tensión y la resuelve. No se trata de esconderla, sino de explicar por qué no es un conflicto real.

---

## 5. Por qué el primer párrafo es autosuficiente

El enunciado pide que un mando pueda decidir leyendo solo el primer párrafo. Por eso la sección **SITUACIÓN** contiene:

- **Qué ha pasado:** alteración de la señalización marítima, contenida y aislada, pero no neutralizada del todo.
- **Estado actual:** tráfico civil operando con normalidad, ferris manteniendo frecuencia, implante latente sin capacidad de acción confirmada.
- **Investigación en curso:** atribución sin cerrar.
- **Decisión pendiente:** mantener el aislamiento, reforzar la vigilancia o escalar la respuesta.

Con eso, un mando sabe exactamente dónde está la operación y qué se le pide.

---

## 6. Por qué no se repite el timeline del Reto 17

El Reto 17 pedía **trazabilidad**: momento, hecho y artefacto. El Reto 20 (este) pide **síntesis**: no horas ni secuencias, sino estado y decisiones.

Por eso el SITREP no incluye fechas, horas, IPs, hashes ni rutas. Solo menciona los retos 8, 16 y 19 por número, para que el corrector los localice sin esfuerzo.

---

## 7. Estructura del SITREP

- **SITUACIÓN:** qué ha pasado y estado actual (un párrafo autosuficiente).
- **IMPACTO:** qué está afectado y qué no.
- **ACCIONES:** qué se hizo y con qué criterio, integrando los retos 8, 16 y 19, y resolviendo la tensión.
- **RECOMENDACIÓN:** dos o más medidas priorizadas.
- **RIESGO RESIDUAL:** qué sigue abierto.

---

## 8. Texto final

### Situación

La operación sigue abierta y aunque hemos contenido la alteración de la señalización marítima y el segmento comprometido quedó aislado, no lo hemos neutralizado del todo, así que el tráfico civil del Estrecho sigue operando con normalidad y los ferris mantienen su frecuencia, pero la amenaza principal, ese implante latente en una baliza de navegación, sigue presente aunque sin capacidad de acción confirmada, y mientras tanto la investigación sobre el origen del sabotaje del cable submarino continúa con la atribución sin cerrar, de modo que el mando tiene que decidir si mantenemos el aislamiento actual, si refuerzo la vigilancia o si escalamos la respuesta.

### Impacto

Está afectada la red de señalización portuaria, que depende del segmento aislado, y también el sistema de gestión del cable submarino, que sigue bajo investigación, pero no están afectados el tráfico civil, los ferris de pasaje ni las balizas sanas, que siguen operando, porque la contención evitó que el implante alcanzara la red de gestión y el panel de disyuntores, y el impacto económico y social se ha limitado al entorno OT, sin afectar a la operación portuaria ni a la navegación comercial.

### Acciones

Apliqué contención parcial del proceso comprometido en el Reto 8, manteniendo el servicio legítimo activo para evitar pérdida de servicio, y en el Reto 16 aislamos el segmento OT, pero ese aislamiento no entra en conflicto con la decisión anterior porque no corta el tráfico civil ni las balizas sanas, ya que la contención parcial del Reto 8 se refería al proceso y no al segmento completo, y en el Reto 19 fijamos una regla de enfrentamiento basada en necesidad militar, proporcionalidad y distinción, para neutralizar la amenaza sin dañar a terceros ni escalar el conflicto.

### Recomendación

Primero, mantener el aislamiento del segmento comprometido y revisar la regla de enfrentamiento cada 24h, segundo, reforzar la vigilancia sobre la baliza afectada y sobre el sistema de gestión del cable, con atención a cualquier reactivación del implante, y tercero, elevar al mando superior la decisión sobre atribución y respuesta al sabotaje, dado que afecta a infraestructura civil crítica y a un proveedor externo.

### Riesgo residual

Sigue abierto el riesgo de reactivación del implante, la atribución sin confirmar del sabotaje y la posibilidad de que el adversario use de nuevo la credencial comprometida, y también queda pendiente la validación independiente de la persistencia, así que el riesgo es bajo, pero requiere seguimiento.

---

## 9. Conclusión

El SITREP integra las tres decisiones, resuelve la tensión entre los retos 8 y 16, mantiene el primer párrafo autosuficiente y no repite el timeline del Reto 17. Está dentro del límite de 400 palabras y cumple la estructura sugerida. El texto final es el que se entrega por formulario.

---

## 10. Lecciones aprendidas

- **La síntesis es tan importante como la trazabilidad.** Un mando necesita decidir con el primer párrafo.
- **Reconocer la tensión puntúa.** No se trata de esconder las contradicciones aparentes, sino de explicar por qué no son conflictos reales.
- **Las decisiones actúan sobre objetos distintos.** Proceso vs. segmento: misma lógica, distinto alcance.
- **Los principios ROE son transversales.** Necesidad militar, proporcionalidad y distinción guían todas las decisiones.
- **El SITREP no repite el timeline.** Es un documento de síntesis, no de trazabilidad.
- **El riesgo residual debe reconocerse.** La operación sigue abierta y hay que ser honesto sobre lo que queda pendiente.
