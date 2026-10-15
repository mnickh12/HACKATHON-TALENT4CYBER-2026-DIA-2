# Cripto del Convoy — PIR-14

## 1. Datos del reto

| Campo | Valor |
|-------|-------|
| **Nombre** | PIR-14. Recuperar el tráfico de órdenes del convoy |
| **Categoría** | Criptografía |
| **Puntos** | 20 pts |
| **Objetivo** | Recuperar la clave privada de firma y descifrar el mensaje 3 |
| **Formato de entrega** | `T4C{...}` |
| **Flag** | `T4C{3cds4_n0nc3_r3us3_c0nv0y}` |

---

## 2. Material

- **`LEEME.txt`:** las órdenes viajan en claro y firmadas con **ECDSA sobre secp256k1**. El tercer mensaje va además cifrado con **AES-256-GCM**, con formato `nonce (12) | etiqueta (16) | texto cifrado`. La clave simétrica es `SHA-256(clave privada de firma, 32 bytes big-endian)`.
- **`trafico_convoy.txt`:** la clave pública `Q = (x, y)`, los textos de **ORDOP-041** y **ORDOP-042** y sus firmas `(r, s)`.
- **El fallo:** las dos firmas tienen el **mismo `r`**: `0xa752f26e...9f30e2`. El enunciado decía que el fallo se veía a simple vista, y es este.

---

## 3. Por qué un `r` repetido rompe ECDSA

Para firmar un mensaje con hash `h` y clave privada `d`, el firmante elige un **nonce secreto `k`** y calcula:

    r = (k·G).x mod n
    s = k⁻¹ · (h + r·d) mod n

`r` solo depende de `k`. Si dos firmas distintas tienen el mismo `r`, usaron el **mismo `k`**. El reinicio de la unidad es la causa probable: el generador de números aleatorios volvió a arrancar con el mismo estado y repitió el nonce.

Con el mismo `k` y `r` hay dos ecuaciones y dos incógnitas (`k` y `d`):

    s1 = k⁻¹ (h1 + r·d)
    s2 = k⁻¹ (h2 + r·d)

**Despejar `k`.** Restando las ecuaciones desaparece `r · d`:

    s1 – s2 = k⁻¹ (h1 – h2)

es decir:

    k = (h1 – h2) / (s1 – s2) mod n

**Despejar `d`.** Sustituyendo `k` en la primera ecuación:

    d = (s1·k – h1) / r mod n

---

## 4. Cálculo

- **Hashes:** `h1` y `h2` son el `SHA-256` de cada texto en claro (UTF-8), convertidos a entero big-endian y reducidos módulo `n`, como indica el campo `digesto` del JSON. `n` es el orden de la curva.
- **Aritmética de la curva:** se implementaron suma y multiplicación escalar sobre `secp256k1` para verificar resultados.
- **Dos candidatos:** se probaron `s1 - s2` y `s1 + s2`, porque si un firmante usa `-s` la firma sigue siendo válida. Solo el primer caso es consistente.
- **Resultado:**

    d = 0x5c1a6f3b8e20d47a91ff03c6b5e82d14907ac3f65b1d8e240a7c93f1b6e5d082

---

## 5. Verificaciones

1. `d·G == Q`, la clave pública del fichero. Confirma que `d` es la clave privada real.
2. `k·G` tiene coordenada `x = r`, así que el nonce recuperado es coherente con las firmas.
3. El candidato de signo opuesto (`0xb5e2e658...7487`) da `d·G ≠ Q` y se descarta.

---

## 6. Descifrar el mensaje 3

- Se leyó `mensaje_03.enc` (484 bytes) y se partió en `nonce = bytes[0:12]`, `tag = bytes[12:28]` y `ct = bytes[28:]`.
- La clave AES se derivó como `SHA-256(d.to_bytes(32, 'big'))`.
- Se descifró con **AES-256-GCM**. La librería `cryptography` espera `ct + tag` concatenados, por eso se reordenaron.
- La etiqueta GCM validó: la clave es correcta y el texto no está alterado. Con la clave derivada del candidato erróneo falla con `InvalidTag`.

---

## 7. Resultado

El texto descifrado es la **ORDOP-043**, que informa de una alteración del plan de transporte y ordena no aceptar rutas sin firma. Contiene la línea:

    T4C{3cds4_n0nc3_r3us3_c0nv0y}

---

## 8. Lectura final

La clave privada no se obtiene rompiendo la curva (secp256k1 es sólida), sino por un **fallo de implementación: reutilizar el nonce**. Es el mismo error que expuso la clave de firma de la **PlayStation 3 en 2010**.

La mitigación es usar **nonces deterministas (RFC 6979)** o derivarlos de la clave y el mensaje, de modo que un reinicio no pueda repetirlos.

La anotación sobre «parámetros de dominio alterados» era un **señuelo**: la curva usada es la estándar y no hizo falta nada especial.

---

## 9. Lecciones aprendidas

- **Reutilizar el nonce en ECDSA es catastrófico.** Con dos firmas que comparten `r`, se recupera la clave privada con álgebra básica.
- **El fallo se ve a simple vista:** basta comparar los valores de `r` de dos firmas.
- **Los reinicios de la unidad** pueden repetir el estado del PRNG y, por tanto, el nonce.
- **La mitigación estándar es RFC 6979:** nonces deterministas derivados de la clave y el mensaje.
- **No todo lo que dice el enunciado es relevante.** La pista sobre «parámetros de dominio alterados» era un distractor.
- **La verificación es esencial:** comprobar que `d·G == Q` confirma que la clave recuperada es la correcta.
