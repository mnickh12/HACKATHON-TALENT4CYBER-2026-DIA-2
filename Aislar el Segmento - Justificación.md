# Aislar el Segmento: Justificación — Decisión ROE

## 1. Datos del reto

| Campo | Valor |
|-------|-------|
| **Nombre** | Aislar el Segmento: Justificación |
| **Categoría** | Decisión ROE |
| **Dificultad** | Difícil |
| **Objetivo** | Redactar una justificación de máximo 200 palabras aplicando necesidad militar, proporcionalidad y distinción |
| **Extensión máxima** | 200 palabras |
| **Formato de entrega** | Sin flag. Se entrega por formulario y se puntúa íntegramente por rúbrica |

---

## 2. Contexto del reto

El reto ORDOP plantea una situación de ciberseguridad operacional (OT) en un entorno portuario. Hay un implante malicioso alojado en una baliza de navegación concreta, la `10.30.9.14`, dentro del segmento `OT-BALIZA` (`10.30.9.0/24`).

Esa baliza comprometida puede, si no se contiene, alcanzar tres sistemas críticos:

- La **señalización portuaria** (`10.30.7.20`), de la que dependen los ferris de pasaje que cruzan cada media hora.
- El **panel de disyuntores** (`10.30.7.44`), que forma parte de la infraestructura eléctrica del puerto.
- La **red de gestión** (`10.10.0.31`), donde viven los sistemas de backoffice y TI.

El reto tiene dos partes diferenciadas:

1. **Parte técnica:** aplicar una segmentación de red que aísle el sistema comprometido y supere una comprobación automática de 6 condiciones ROE. Esta parte se acredita en la entrada "Contención".
2. **Parte de decisión:** redactar una justificación de máximo 200 palabras aplicando los principios de necesidad militar, proporcionalidad y distinción. Esta parte es la que puntúa en la entrada que nos ocupa.

### Advertencias del enunciado

- **Advertencia 1:** denegar todo el tráfico aísla la amenaza, pero no supera el ejercicio. Tres de las seis condiciones exigen que algo siga funcionando.
- **Advertencia 2:** el estado en que se deje la red lo hereda la siguiente célula. Si se deja bien segmentada, la siguiente célula supera la comprobación sin razonar. Si se deja desordenada, tendrá que diagnosticarla. Como la red se restablece cada 45–60 minutos, ninguna ventaja es permanente y el registro de lo que hace cada célula es público.

---

## 3. Primeros pasos: la interfaz web

El primer intento de resolución fue a través de la interfaz web del reto, en `https://retos.locdynamics.com/reto16/`. Sin embargo, la interfaz no funcionaba correctamente: los botones de aplicar reglas no respondían, o la página no cargaba los datos. Esto obligó a buscar una vía alternativa: **la API del backend**.

---

## 4. Descubrimiento de los endpoints

El HTML de la página del reto (que se pudo obtener aunque la interfaz estuviera rota) reveló en su pie de página los endpoints disponibles:

    GET /api/red
    GET /api/comprobacion
    POST /api/reglas
    GET /api/actividad

Estos endpoints son relativos al dominio `retos.locdynamics.com`. Sin embargo, el enunciado del reto indicaba explícitamente que el conjunto de reglas se enviaba a `/reto16/api/reglas`, lo que sugería que el prefijo correcto era `/reto16/`, no `/api/` a secas.

---

## 5. Primer intento de POST: error 404

El primer intento de aplicar reglas por consola usó la URL relativa `/api/reglas`, que el navegador resolvió a `https://retos.locdynamics.com/api/reglas`. El servidor respondió con un **404 Not Found**, y el cuerpo de la respuesta era HTML en lugar de JSON, lo que provocó además un error de parseo:

    POST https://retos.locdynamics.com/api/reglas 404 (Not Found)
    SyntaxError: Unexpected token '<', "<html>..." is not valid JSON

La conclusión fue clara: la URL correcta debía incluir el prefijo `/reto16/`.

---

## 6. Segundo intento de POST: error 400

El segundo intento usó la URL correcta, `/reto16/api/reglas`, pero envió las reglas con los nombres de campo y valores en mayúscula:

    [
      { "origen": "10.10.0.5/32", "destino": "10.30.9.14/32", "accion": "PERMITIR" },
      { "origen": "10.30.9.14/32", "destino": "0.0.0.0/0", "accion": "DENEGAR" },
      { "origen": "0.0.0.0/0", "destino": "10.30.9.14/32", "accion": "DENEGAR" }
    ]

El servidor respondió con un **400 Bad Request** y el mensaje:

    { "error": "regla mal formada: {'origen': '10.10.0.5/32', 'destino': '10.30.9.14/32', 'a..." }

Esto indicaba que los nombres de campo (`origen`, `destino`, `accion`) eran correctos, pero algo en el formato no encajaba. La hipótesis principal fue que los valores de `accion` debían ir en minúscula, o que el array debía ir envuelto en un objeto.

---

## 7. Descubrimiento del formato correcto

Para salir de dudas, se hizo una petición GET de solo lectura al endpoint `/reto16/api/red`, que devuelve el estado actual de la red. La respuesta incluía un campo revelador: `formato_regla`.

    {
      "formato_regla": {
        "accion": "permitir | denegar",
        "destino": "CIDR o IP/32",
        "orden": "la primera regla que encaja decide; sin coincidencias, permitido",
        "origen": "CIDR o IP/32"
      },
      "reglas": [
        { "accion": "denegar", "destino": "10.30.7.20", "origen": "10.30.9.14" },
        { "accion": "denegar", "destino": "10.30.7.44", "origen": "10.30.9.14" },
        { "accion": "denegar", "destino": "10.10.0.31", "origen": "10.30.9.14" },
        { "accion": "denegar", "destino": "10.30.7.20", "origen": "10.30.9.15" }
      ],
      "segmentos": {
        "CIVIL": { "descripcion": "Consola de tráfico civil del Estrecho --- solo lectura", "red": "10.40.1.0/24" },
        "CORP": { "descripcion": "Gestión portuaria (TI) --- estaciones y servidores de backoffice", "red": "10.10.0.0/24" },
        "OT-BALIZA": { "descripcion": "Balizas de navegación (OT) --- ES-TAR-04 COMPROMETIDA", "red": "10.30.9.0/24" },
        "OT-SENAL": { "descripcion": "Señalización portuaria (OT) --- soporta tráfico civil del Estrecho", "red": "10.30.7.0/24" }
      }
    }

Este JSON proporcionó tres informaciones críticas:

1. **Los valores de `accion` van en minúscula:** `"permitir"` o `"denegar"`. Ese era el fallo del 400.
2. **El orden importa:** "la primera regla que encaja decide; sin coincidencias, permitido".
3. **El estado inicial de la red:** había 4 reglas de denegación que aislaban la baliza comprometida, pero también bloqueaban la baliza sana `10.30.9.15`, lo que rompía la proporcionalidad.

---

## 8. Análisis de las 6 condiciones ROE

La página del reto mostraba una tabla de verificación con 6 condiciones. En el estado inicial (5/6 OK), las condiciones eran:

| # | Origen | Destino | Esperado | Obtenido | Criterio | Estado |
|---|--------|---------|----------|----------|----------|--------|
| 1 | 10.30.9.14 | 10.30.7.20 | denegado | denegado | La baliza comprometida no debe alcanzar la señalización portuaria | OK |
| 2 | 10.30.9.14 | 10.30.7.44 | denegado | denegado | La baliza comprometida no debe alcanzar el panel de disyuntores | OK |
| 3 | 10.30.9.14 | 10.10.0.31 | denegado | denegado | La baliza comprometida no debe alcanzar la red de gestión | OK |
| 4 | 10.40.1.10 | 10.30.7.20 | permitido | permitido | El tráfico civil debe seguir recibiendo señalización --- distinción | OK |
| 5 | 10.10.0.5 | 10.30.9.14 | permitido | permitido | El bastión forense debe conservar acceso para preservar la evidencia | OK |
| 6 | 10.30.9.15 | 10.30.7.20 | permitido | **denegado** | La baliza sana no debe quedar aislada --- proporcionalidad | **FALLA** |

La condición 6 fallaba porque las reglas iniciales usaban `10.30.9.0/24` como destino, lo que bloqueaba también a la baliza sana `10.30.9.15`. La solución era usar `/32` (IP concreta) en lugar de `/24` (subred entera).

---

## 9. Las 3 reglas correctas

Con toda la información anterior, las reglas necesarias eran:

| # | Origen | Destino | Acción | Razón |
|---|--------|---------|--------|-------|
| 1 | 10.10.0.5/32 | 10.30.9.14/32 | permitir | El bastión forense preserva evidencia |
| 2 | 10.30.9.14/32 | 0.0.0.0/0 | denegar | La baliza comprometida no alcanza nada |
| 3 | 0.0.0.0/0 | 10.30.9.14/32 | denegar | Nadie alcanza la baliza comprometida |

El **orden** es crucial porque "la primera regla que encaja decide". La regla 1 permite explícitamente al bastión forense acceder a la baliza comprometida. Las reglas 2 y 3 deniegan todo lo demás hacia y desde la baliza comprometida. Al usar `/32` en lugar de `/24`, la baliza sana `10.30.9.15` queda fuera de las denegaciones y puede seguir comunicándose con la señalización.

---

## 10. Aplicación de las reglas y obtención de la flag

El POST final a `/reto16/api/reglas` con las 3 reglas y los valores en minúscula devolvió:

    {
      "aislado": true,
      "condiciones": [ ... 6 elementos, todos OK ... ],
      "estado": "reglas aplicadas al entorno",
      "flag": "T4C{5g3mn3nt0_OT_41sL4d0_0k}"
    }

La respuesta también incluía un recordatorio:

> "La flag cubre solo la mitad técnica. La justificación..."

La flag es `T4C{5g3mn3nt0_OT_41sL4d0_0k}`, y debe entregarse en la entrada "Contención".

---

## 11. La justificación escrita

La segunda parte del reto, la que puntúa en esta entrada, es una justificación de máximo 200 palabras aplicando los principios de **necesidad militar**, **proporcionalidad** y **distinción**. La versión final desarrollada es:

> Lo primero: aislar la baliza comprometida (10.30.9.14) es necesario porque tiene un implante activo y desde ahí pueden tocar la señalización, los disyuntores y la red de gestión. Eso es la **necesidad militar**, y no hay discusión. Si denegamos 10.30.9.0/24 entero, dejo fuera a la baliza sana 10.30.9.15, a los ferris que cruzan cada media hora y a las balizas que guían a los barcos de noche. Estaría cortando tráfico civil que no tiene nada que ver con el implante, y eso rompe la **proporcionalidad**. Así que voy con contención quirúrgica: permito al bastión forense (10.10.0.5) llegar a la baliza comprometida para preservar evidencia, y dejo pasar el tráfico civil de 10.40.1.0/24 hacia la señalización 10.30.7.20. Todo lo demás hacia y desde 10.30.9.14, denegado. Separo el sistema atacado del tráfico legítimo: **distinción**. La amenaza queda aislada, la evidencia se conserva y el Estrecho sigue funcionando.

Esta justificación:

- **Necesidad militar:** justifica aislar la baliza comprometida por el implante activo.
- **Proporcionalidad:** explica por qué no se debe denegar todo el segmento `/24`, y cómo la contención quirúrgica con `/32` evita dañar el tráfico civil.
- **Distinción:** separa explícitamente el sistema atacado del tráfico legítimo.

---

## 12. Lecciones aprendidas

1. **La interfaz no siempre funciona:** en entornos de competición, la API suele ser la vía fiable. Los endpoints a veces están ocultos en el propio HTML de la página.
2. **El backend dice lo que espera:** el campo `formato_regla` en `/api/red` fue la clave para descubrir que los valores de `accion` van en minúscula.
3. **El orden de las reglas importa:** en un firewall, la primera regla que encaja decide. Por eso la regla del bastión forense va primero.
4. **La precisión importa:** usar `/32` en lugar de `/24` evita aislar nodos sanos que deben seguir operando.
5. **Los principios ROE son vinculantes:** no se puede denegar todo; hay que respetar la necesidad militar, la proporcionalidad y la distinción.
6. **El reset es rápido:** la red se restablece cada 45–60 segundos, así que hay que actuar con rapidez o volver a aplicar las reglas.

---

## 13. Conclusión final

El reto se resolvió mediante una **contención quirúrgica** del segmento comprometido. En lugar de denegar todo el tráfico (que habría aislado la amenaza pero roto la proporcionalidad y la distinción), se aplicaron tres reglas precisas:

1. Permitir al bastión forense acceder a la baliza comprometida para preservar evidencia.
2. Denegar todo el tráfico saliente de la baliza comprometida.
3. Denegar todo el tráfico entrante hacia la baliza comprometida.

El uso de `/32` en lugar de `/24` fue fundamental para no aislar la baliza sana `10.30.9.15`. El resultado fue `aislado: true` con las 6 condiciones ROE superadas, y la flag `T4C{5g3mn3nt0_OT_41sL4d0_0k}`.

La justificación escrita refleja esta decisión aplicando los tres principios: necesidad militar (aislar la amenaza), proporcionalidad (no cortar más de lo necesario) y distinción (separar el sistema atacado del tráfico legítimo).
