# Convertir BYTE a INTEGER en Step 7

## Descripción

En Siemens Step 7 (S7-300/S7-400), los datos de tipo **BYTE** (8 bits, sin signo, rango: 0..255) a veces necesitan ser tratados como **INT** (16 bits, con signo, rango: -32768..32767) para poder usarlos en operaciones aritméticas o comparaciones numéricas.

A continuación se describen los métodos más comunes para realizar esta conversión.

---

## Método 1: Usando la instrucción BTI (BCD to Integer) — Solo para valores BCD

La instrucción `BTI` convierte un valor en formato **BCD empaquetado** (Binary-Coded Decimal) a **INT**. Solo aplica si el BYTE contiene un valor en formato BCD.

### STL (AWL)

```
L     MB10         // Cargar BYTE (en formato BCD)
BTI              // Convertir BCD a Integer
T     MW20         // Transferir resultado a WORD/INT
```

> ⚠️ **Nota**: Este método solo es válido si el BYTE contiene un valor BCD válido (cada nibble entre 0 y 9). Si el valor no es BCD, usar el Método 2.

---

## Método 2: Conversión directa BYTE → INT usando MOVE (Método recomendado)

Para la mayoría de los casos (valores binarios normales), el método más simple es transferir el BYTE a una variable INT usando un bloque de datos intermediario o mediante la instrucción de transferencia con extensión de cero.

### STL (AWL)

```
L     B#16#00      // Cargar 0 en acumulador 1
T     MW20         // Limpiar la palabra de destino (parte alta)
L     MB10         // Cargar el valor BYTE de origen (p.ej. MB10)
T     MB21         // Transferir al byte bajo de la palabra destino (MW20 = MB20+MB21)
L     MW20         // Cargar la palabra completa (ya como INT positivo)
T     MW30         // Guardar el resultado INTEGER
```

### Alternativa STL más directa

```
L     MB10         // Cargar BYTE (8 bits) — se carga con ceros en la parte alta
T     MW20         // Transferir a WORD/INT (la parte alta queda en 0)
```

> En Step 7, al cargar un BYTE con `L MB10`, el acumulador queda con los 24 bits superiores en cero. Al transferir con `T MW20`, los 16 bits bajos se guardan en la palabra destino, dando el valor entero positivo equivalente.

---

## Método 3: Usando el bloque FC de conversión en LAD/FBD

En el lenguaje **LAD** (KOP) o **FBD** (FUP), se puede usar la función estándar de conversión disponible en la librería de Step 7:

### LAD (KOP)

```
       +--------+
  -----|  MOVE  |-----
  EN   |        | ENO
  IN   | MB10   |
       |   →    |
       | MW20   |
       +--------+
```

O bien usando una asignación directa de tipos compatibles mediante un bloque FC personalizado.

### FBD (FUP)

```
+-------+
| MOVE  |
|  IN:  MB10  |
|  OUT: MW20  |
+-------+
```

> En LAD/FBD, un `MOVE` desde un `BYTE` hacia un `INT` o `WORD` realiza la conversión automáticamente, colocando el valor del BYTE en los 8 bits bajos y rellenando con ceros los 8 bits altos.

---

## Método 4: Usando bloques FC estándar de la librería IEC

Step 7 incluye funciones de conversión en su librería estándar IEC (disponibles en el Administrador de Simatic → Librerías → Standard Library → TI-S7 Converting Blocks):

| Función | Nombre | Descripción                          |
|---------|--------|--------------------------------------|
| `FC40`  | `B_I`  | Convierte BYTE a INT                 |
| `FC39`  | `B_DINT` | Convierte BYTE a DINT (entero doble) |
| `FC41`  | `B_R`  | Convierte BYTE a REAL                |

### Uso en STL

```
CALL  FC40
  IN  := MB10      // Byte de entrada
  RET_VAL := MW20  // Resultado INTEGER
```

---

## Ejemplo práctico completo

**Escenario**: Se recibe un valor de temperatura en un BYTE (0–200°C) y se necesita usarlo como INTEGER para comparaciones aritméticas.

```
// Variables:
// MB10  → Temperatura_BYTE  (tipo BYTE, valor recibido de instrumento)
// MW20  → Temperatura_INT   (tipo INT, para uso en cálculos)

// Paso 1: Limpiar la palabra destino
L     0
T     MW20

// Paso 2: Copiar BYTE al byte bajo de la palabra
L     MB10
T     MB21          // MB21 es el byte bajo de MW20

// Paso 3: Usar el valor INTEGER
L     MW20
L     100
>=I                 // Comparar: ¿Temperatura >= 100°C?
=     M50.0         // Activar alarma si es verdadero
```

---

## Notas importantes

- En Step 7, los tipos **BYTE** y **WORD** son sin signo. El tipo **INT** es con signo. Al convertir un BYTE (0–255) a INT, el resultado siempre será positivo (0–255), ya que el bit de signo del INT queda en 0.
- Si el BYTE contiene un valor > 127 y se necesita interpretar como signed, se debe evaluar el bit 7 y extender el signo manualmente.
- Para conversiones en TIA Portal (S7-1200/S7-1500), utilizar la instrucción `CONVERT` (CONV) que soporta múltiples tipos de datos directamente.

---

*Documento elaborado por Franco Javier Loudet — Automatización Industrial*  
*Referencia: Siemens Step 7 v5.x — S7-300/S7-400*
