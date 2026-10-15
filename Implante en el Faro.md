# Implante en el Faro — PIR-12

## 1. Datos del reto

| Campo | Valor |
|-------|-------|
| **Nombre** | PIR-12. ¿Bajo qué condición se activa el implante? |
| **Categoría** | Reversing |
| **Baliza** | ES-TAR-04 "Punta Carnero" |
| **Material** | `implante.elf` y `NOTA_TECNICA.txt` |
| **Puntos** | 25 pts |
| **Formato de entrega** | `T4C{...}` |
| **Flag** | `T4C{4rm_1mpl4nt3_f4r0_tr1gg3r}` |

---

## 2. Resumen ejecutivo

| Concepto | Resultado |
|----------|-----------|
| **Flag** | `T4C{4rm_1mpl4nt3_f4r0_tr1gg3r}` |
| **Código de activación (60 hex / 30 bytes)** | `12250106c6df03d685fe4443fe1bccb2f357e055727ce4e155b84df6fcb2` |
| **Arquitectura** | ARM 32 bits, EABI5, enlazado estáticamente, sin símbolos, código en modo **Thumb-2** |
| **Método** | Análisis estático manual del código de `main` + resolución algebraica de las 30 condiciones |
| **Ejecución del binario** | **No realizada** (no hay `qemu-arm` ni emulador ARM en el entorno) |
| **Verificación** | Las 30 condiciones se recalcularon hacia delante con el código obtenido y se cumplen todas. La salida es texto legible en leetspeak, lo que respalda la lectura del código |

La condición de activación es un conjunto de **30 ecuaciones independientes** sobre los bytes del código. No hizo falta ejecución simbólica porque cada ecuación se puede invertir directamente.

---

## 3. Entorno y limitaciones

Lo primero fue comprobar qué herramientas había disponibles.

- **Disponibles:** `python3` 3.12, `gcc` (x86), `objdump` (solo targets x86), `readelf`, `strings`, `numpy`, `sympy`.
- **No disponibles:** `qemu-arm-static`, desensamblador ARM (`objdump` no soporta ARM), `capstone`, `z3`, `angr`, `unicorn`, `gdb`, `radare2`, `ghidra`.
- **Red desactivada:** `pip download z3-solver` falló, así que no se pudieron instalar herramientas.

**Consecuencia:** no se podía ni ejecutar el binario ni desensamblarlo con herramientas. El trabajo se hizo **leyendo los bytes del código Thumb-2 y decodificando las instrucciones a mano**, apoyado en scripts de Python para localizar direcciones, resolver literales y calcular los resultados.

---

## 4. Reconocimiento inicial

### 4.1. Formato del binario

    ELF 32-bit LSB executable, ARM, EABI5 version 1 (GNU/Linux), statically linked, stripped
    Entry point: 0x10341

El punto de entrada tiene el bit 0 a 1 (`0x10341`), lo que indica **modo Thumb**. Esto condiciona cómo se decodifica todo el código.

Secciones relevantes:

| Sección | Dirección | Contenido |
|---------|-----------|-----------|
| `.text` | `0x101c0` | Código (0x3e670 bytes) |
| `.rodata` | `0x4f02c` | Datos de solo lectura, cadenas y tablas |
| `.got` | `0x68000` | Tabla de offsets globales |
| `.init_array` | `0x660b8` | Constructores |

### 4.2. Cadenas visibles

Buscando mensajes del propio programa en `.rodata` aparecieron:

| Dirección | Texto |
|-----------|-------|
| `0x4f0c0` | `uso: %s <codigo-activacion-hex>` |
| `0x4f0e4` | `baliza: longitud de codigo incorrecta` |
| `0x4f10c` | `baliza: codigo no hexadecimal` |
| `0x4f12c` | `baliza: ciclo nominal, 3 destellos / 15 s` |

Esto confirma lo que decía la nota técnica: el binario valida longitud y formato hexadecimal. Como se esperaba, **la flag no aparece como cadena**.

---

## 5. Localización de `main` (primer obstáculo)

El binario no tiene símbolos y está enlazado estáticamente, así que hay cientos de funciones de glibc mezcladas con el código del reto. Había que encontrar `main`.

### 5.1. Intentos fallidos por referencias a cadenas

Se buscaron referencias a los mensajes por varias vías:

1. Palabras de 32 bits con la dirección de la cadena (literal pool).
2. Pares `movw/movt` en Thumb-2 y en ARM.
3. `adr` de 16 y 32 bits (Thumb).

Ninguna dio un resultado útil. Un literal cercano (`0x4f0bc`, en `0x172f4`) resultó ser un falso positivo: pertenecía a código de `vfprintf` de glibc, no al reto.

### 5.2. Resolución por `_start`

Se decodificó `_start` (`0x10340`) a mano:

    0x10340  mov.w   fp, #0
    0x10344  mov.w   lr, #0
    0x10348  pop     {r1}
    0x1034a  mov     r2, sp
    0x1034c  push    {r2}
    0x1034e  push    {r0}
    0x10350  ldr.w   sl, [pc, #0x18]    ; literal en 0x1036c = 0x57c94
    0x10354  adr     r3, ...
    0x10356  add     sl, r3             ; sl = base de la GOT = 0x68000
    0x10358  mov.w   r3, #0
    0x1035c  push    {r3}
    0x1035e  ldr     r0, [pc, #16]      ; literal en 0x10370 = 0x188
    0x10360  ldr.w   r0, [sl, r0]       ; r0 = GOT[0x188] = main
    0x10364  bl      __libc_start_main

Con el literal `0x188` y la base `0x68000`, la entrada de la GOT en `0x68188` contiene `0x1046f`. Quitando el bit Thumb: **`main` está en `0x1046e`**.

---

## 6. Análisis de `main` (0x1046e)

Se volcaron los bytes de la función y se decodificaron las instrucciones Thumb-2 manualmente.

### 6.1. Validación de argumentos

    0x1046e  push.w  {r4-r9, lr}
    0x10472  sub     sp, #32            ; buffer de 32 bytes: aquí irán los 30 bytes del código
    0x10474  ldr.w   sl, [pc, #0x10c]   ; base de la GOT (PIC)
    0x10478  add     sl, pc
    0x1047a  cmp     r0, #2             ; argc == 2 ?
    0x1047c  bne     → "uso: ..."
    0x1047e  ldr     r7, [r1, #4]       ; r7 = argv[1]
    0x10480  mov     r0, r7
    0x10482  bl      strlen
    0x10486  cmp     r0, #0x3c          ; longitud == 60 ?
    0x10488  bne     → "longitud de codigo incorrecta"

### 6.2. Conversión hexadecimal

    0x1048a  add.w   r8, sp, #0xffffffff   ; r8 = sp - 1
    0x1048e  mov     r6, r8
    0x10490  movs    r5, #0
    0x10492  add.w   r9, r7, #1
    loop_hex:
    0x10496  ldrb.w  r0, [r7, r5, lsl #1]  ; carácter 2i   (nibble alto)
    0x1049a  bl      hexval
    0x1049e  mov     r4, r0
    0x104a0  ldrb.w  r0, [r9, r5, lsl #1]  ; carácter 2i+1 (nibble bajo)
    0x104a4  bl      hexval
    0x104a8  orrs    r3, r4, r0            ; si algún nibble es -1, N=1
    0x104ac  bmi     → "codigo no hexadecimal"
    0x104ae  orr.w   r0, r0, r4, lsl #4    ; byte = (alto << 4) | bajo
    0x104b2  strb    r0, [r6, #1]!         ; b[i] en sp+i
    0x104b6  adds    r5, #1
    0x104b8  cmp     r5, #0x1e             ; 30 iteraciones
    0x104ba  bne     loop_hex

Aquí hay un detalle que costó fijar: al principio `ldrb.w r0,[r7,r5]` parecía indexar con paso 1, pero el `shift` del encoding (`0x0015` en el segundo halfword) indica `lsl #1`, es decir, el índice avanza de dos en dos caracteres. Los bytes quedan en `b[0..29]` en la pila.

### 6.3. Las 30 comprobaciones

    0x104bc  ldr     r0, [pc, #200]     ; literal en 0x10588 = 0x3ec96
    0x104be  add     r0, pc             ; r0 = 0x3ec96 + 0x104c2 = 0x4f158  (tabla T)
    0x104c0  subs    r4, r0, #1         ; r4 = T - 1
    0x104c2  adds    r0, #0x1f          ; r0 = T + 0x1f
    0x104c4  mov     r1, r8             ; r1 = sp - 1
    0x104c6  movs    r2, #0             ; k = 0
    loop_check:
    0x104c8  tst     r2, #1             ; ¿k impar?
    0x104cc  ldrb    r3, [r1, #1]       ; r3 = b[k]
    0x104ce  itt     ne
    0x104d0  ldrbne  r5, [r1]           ;   (k impar) r5 = b[k-1]
    0x104d2  addne   r3, r3, r5         ;   (k impar) r3 = b[k] + b[k-1]
    0x104d4  uxtb    r3, r3
    0x104d6  ldrb    r5, [r4, #1]!      ; r5 = T[k]
    0x104da  eors    r3, r5             ; r3 ^= T[k]
    0x104dc  rsb     r5, r2, r2, lsl #3 ; r5 = 7*k
    0x104e0  add     r3, r5
    0x104e2  ldrb    r5, [r0, #1]!      ; r5 = C[k]   (C en 0x4f178)
    0x104e6  uxtb    r3, r3
    0x104e8  cmp     r3, r5
    0x104ea  bne     → "ciclo nominal"
    0x104ec  adds    r2, #1
    0x104ee  adds    r1, #1
    0x104f0  cmp     r2, #0x1e
    0x104f2  bne     loop_check

Las tablas quedan en:

- **T** (30 bytes) en `0x4f158`
- **C** (30 bytes) en `0x4f178`

**Error propio y corrección:** la primera vez que se calculó la dirección de la tabla hubo un error de instrucción: se tomó el `ldr` en `0x104ba` en lugar de `0x104bc`, y eso desplazó el `pc` de referencia. Al volcar esa dirección errónea (`0x68046`) aparecieron bytes que eran claramente punteros de la GOT, no datos de tabla. Esa incongruencia delató el error. Se reubicó la instrucción en `0x104bc`, se recalculó el literal (`0x10588` → `0x3ec96`) y la tabla salió en `.rodata` (`0x4f158`), con valores plausibles (T es una progresión aritmética de paso `0x25`, típica de un reto generado).

Valores extraídos:

    T = 5b 80 a5 ca ef 14 39 5e 83 a8 cd f2 17 3c 61 86 ab d0 f5 1a 3f 64 89 ae d3 f8 1d 42 67 8c
    C = 49 be b2 e2 45 d4 64 b8 3e 6a cf c2 3d 80 0f 61 c8 11 93 b4 d9 1d 07 0c 2e a4 06 be 5f ed

### 6.4. Construcción y emisión de la flag

Si las 30 comprobaciones pasan, el programa construye el resultado byte a byte y lo imprime:

    0x104f6  movs    r4, #0x2a              ; clave inicial = 42
    loop_out:
    0x104fa  ldr     r3, [pc, ...]
    0x104fc  ldr.w   r5, [sl, r3]           ; r5 = &stdout (vía GOT)
    0x10500  ldrb    r0, [r8, #1]!          ; r0 = b[i]
    0x10504  rsb     r0, r0, r0, lsl #3     ; r0 = 7 * b[i]
    0x10508  eors    r0, r4                 ; r0 ^= clave
    0x1050a  ldr     r1, [r5]
    0x1050c  uxtb    r0, r0
    0x1050e  bl      putc
    0x10512  adds    r4, #13                ; clave += 13
    0x10514  uxtb    r4, r4
    0x10516  cmp     r8, r6                 ; r6 = sp + 29
    0x10518  bne     loop_out
    ...      putc('\n')

Es decir, el carácter *i* de la flag es `((7·b[i]) ^ ((42 + 13·i) & 0xFF)) & 0xFF`.

---

## 7. Resolución

### 7.1. Planteamiento

Cada una de las 30 condiciones es una ecuación con una sola incógnita nueva:

- **k par:** `((b[k] ^ T[k]) + 7k) mod 256 = C[k]`
  → `b[k] = ((C[k] − 7k) mod 256) ^ T[k]`
- **k impar:** `(((b[k] + b[k-1]) mod 256 ^ T[k]) + 7k) mod 256 = C[k]`
  → `b[k] = (((C[k] − 7k) mod 256) ^ T[k]) − b[k-1] (mod 256)`

Como `b[k-1]` ya está resuelto cuando se llega a `k` impar, se despeja en orden de 0 a 29. **No hay ambigüedad:** cada byte tiene una única solución. Por eso no hace falta ejecución simbólica, pese a lo que sugería la nota técnica.

### 7.2. Código

    d = open('implante.elf', 'rb').read()
    base = 0x10000
    T = d[0x4f158 - base:][:30]
    C = d[0x4f178 - base:][:30]

    b = []
    for k in range(30):
        t = ((C[k] - 7 * k) & 0xff) ^ T[k]
        if k % 2 == 1:
            t = (t - b[k - 1]) & 0xff
        b.append(t)

    codigo = bytes(b).hex()
    flag = bytes(((7 * b[i]) ^ ((42 + 13 * i) & 0xff)) & 0xff for i in range(30))

    print(codigo)
    print(flag)

### 7.3. Resultado

    Código:  12250106c6df03d685fe4443fe1bccb2f357e055727ce4e155b84df6fcb2
    Flag:    T4C{4rm_1mpl4nt3_f4r0_tr1gg3r}

---

## 8. Verificación

1. **Verificación hacia delante:** se recalculó para cada `k` el valor `((v ^ T[k]) + 7k) & 0xFF` con `v = b[k]` (o `b[k] + b[k-1]` si `k` es impar) y se comparó con `C[k]`. Las 30 coincidieron.
2. **Coherencia de la salida:** la flag decodificada tiene exactamente 30 caracteres, empieza por `T4C{`, termina en `}` y contiene texto legible en leetspeak ("arm implante faro trigger"). Un error en cualquier punto de la lectura del código habría producido basura.
3. **Coherencia con la nota técnica:** 30 condiciones encadenadas, dependencia de la posición anterior en las impares, y valor construido en memoria sin existir como cadena. Todo encaja.

**Lo que no se hizo:** no se ejecutó el binario, por falta de emulador. Quien disponga de `qemu-arm-static` puede confirmarlo con:

    qemu-arm-static ./implante.elf 12250106c6df03d685fe4443fe1bccb2f357e055727ce4e155b84df6fcb2

Si el binario no activa el implante, el mensaje será `baliza: ciclo nominal, 3 destellos / 15 s`.

---

## 9. Condición de activación (respuesta a la pregunta del reto)

> El implante se activa cuando recibe por el canal de mantenimiento un único argumento de 60 caracteres hexadecimales (30 bytes `b[0..29]`) que cumpla, para todo `k` entre 0 y 29,
> `((b[k] + [k impar]·b[k-1]) mod 256 ^ T[k]) + 7k ≡ C[k] (mod 256)`.
> Hay una única solución, y al recibirla el implante emite la flag `T4C{4rm_1mpl4nt3_f4r0_tr1gg3r}`. Con cualquier otro valor la baliza funciona con normalidad.

---

## 10. Riesgos y notas

- Las tablas se leyeron del fichero estático. No se ha descartado que algún constructor de `.init_array` las modifique en tiempo de ejecución; solo se leyeron los punteros de `.init_array` (`0x10419` y `0x103e9`), sin analizar su código. La coherencia de la flag obtenida hace improbable que haya modificación, pero no se ha comprobado empíricamente.
- La decodificación fue manual. Los puntos más delicados fueron el `lsl #1` del bucle hexadecimal, el bloque `IT` condicional de las posiciones impares y la dirección de las tablas (ver sección 6.3).
- No se probó ningún valor adicional, así que no se puede asegurar que no haya otras rutas de activación aparte del bucle analizado.

---

## 11. Lecciones aprendidas

- **El análisis estático manual es viable** incluso sin desensamblador ARM, leyendo los bytes del código Thumb-2.
- **La resolución algebraica** de las 30 condiciones es más rápida y fiable que la ejecución simbólica en este caso.
- **Los detalles del encoding Thumb-2** (como el `lsl #1` del índice o el bloque `IT` condicional) son críticos y pueden inducir a error.
- **La verificación hacia delante** es esencial para confirmar que la solución es correcta.
- **La coherencia de la salida** (texto legible en leetspeak) respalda la lectura del código.
- **No siempre hace falta ejecutar el binario** para resolver un reto de reversing; el análisis estático puede ser suficiente.
