# Bitácora de Requerimientos y Auditoría IA — Sesión 06
**Materia:** Lenguajes y Autómatas II (SCD-1016)  
**Tema:** Sistema de Tipos Primitivos  
**Docente Titular:** MCC. Iván Márquez Larios (`ijmarquezl`)  
**Estudiante:** Maria Jesus Fuentes Valdez  
**Matrícula:** 24531315  
**Track:** C (el harness evalúa C; la versión en Rust es opcional y no se califica con este harness)  

---

## 1. Fase 1: El Artesano (Dificultad Manual)
*Reto:* Chequeo de compatibilidad de tipos  
*Describe qué problema o restricción de memoria/sintaxis identificaste al codificar a mano:*
> La dificultad principal fue que `TIPO_ERROR` vale 0 en el enum, o sea que tiene el rango más bajo. Si solo comparaba rangos con `destino >= origen`, `TIPO_ERROR` habría resultado compatible con otros tipos (por ejemplo, `compatible(ERROR, ERROR)` daría 1). Por eso tuve que poner primero la regla de que `TIPO_ERROR` nunca es compatible con nada, y después aplicar la comparación de rangos de la jerarquía CHAR (1) < INT (2) < FLOAT (3). También tuve que cuidar la dirección de la comparación: se acepta asignar a un destino de igual o mayor rango (ensanchamiento) y se rechaza al más angosto, que exige cast explícito. En `nombre_tipo` usé un `switch` con `default` devolviendo "error" para que el compilador no marcara advertencias de retorno con `-Wall -Wextra -Werror`.

---

## 2. Fase 2: El Científico (Ingeniería de Requerimientos C-R-E-O)

### 2.1 Matriz de Especificación Formal
| Elemento C-R-E-O | Definición Técnica |
| :--- | :--- |
| **Contexto (C)** | C99 / GCC o Rust 2021 sin bibliotecas no estándar |
| **Rol (R)** | Ingeniero de software de sistemas y compiladores |
| **Especificación (E)** | Matriz de compatibilidad estática |
| **Output (O)** | Solo código modular y análisis de complejidad |

### 2.2 Prompt Estructurado
```text
CONTEXTO: Estoy trabajando en C (estándar C11, compilado con GCC y las banderas
-Wall -Wextra -Werror), sin bibliotecas no estándar. Tengo la cabecera
api_sesion06.h con:
  typedef enum { TIPO_ERROR = 0, TIPO_CHAR = 1, TIPO_INT = 2, TIPO_FLOAT = 3 } Tipo;
  int compatible(Tipo destino, Tipo origen);
  const char *nombre_tipo(Tipo t);

ROL: Actúa como ingeniero de software de sistemas y compiladores.

ESPECIFICACIÓN: Implementa exactamente esas dos firmas como una tabla de
compatibilidad estática de tipos primitivos. Reglas:
 - La jerarquía es CHAR < INT < FLOAT.
 - compatible(destino, origen) devuelve 1 solo si el destino tiene igual o mayor
   rango que el origen (asignación segura); devuelve 0 si el destino es más
   angosto (requiere cast explícito).
 - TIPO_ERROR nunca es compatible con nada, ni como destino ni como origen.
 - nombre_tipo devuelve "char", "int", "float" o "error".
 - No incluyas main(); no cambies las firmas; el código no debe generar
   advertencias con -Wall -Wextra -Werror.

OUTPUT: Entrega solo el código en C, modular, más un breve análisis de
complejidad temporal y espacial.
```

---

## 3. Código Devuelto por el Modelo
```c
// Código devuelto por el LLM
#include "api_sesion06.h"
#include <stdio.h>

int compatible(Tipo destino, Tipo origen) {
    /* Si cualquiera de los tipos es de tipo error, no es compatible */
    if (destino == TIPO_ERROR || origen == TIPO_ERROR) {
        return 0;
    }
    /* Es compatible si el destino tiene un rango mayor o igual al origen */
    return (destino >= origen);
}

const char *nombre_tipo(Tipo t) {
    switch (t) {
        case TIPO_CHAR:
            return "char";
        case TIPO_INT:
            return "int";
        case TIPO_FLOAT:
            return "float";
        case TIPO_ERROR:
        default:
            return "error";
    }
}
```

---

## 4. Fase 3: El Auditor (Resultados del Test Harness)
*Pruebas ejecutadas:* Rechazo de asignaciones incompatibles

### Salida del harness
```text
=== AUDITORIA SESION 06 - Tabla de compatibilidad de tipos ===
Caso                               esperado:              obtenido:              resultado
[ 1] compatible(INT,INT)                esperado: 1            obtenido: 1            OK
[ 2] compatible(FLOAT,INT)              esperado: 1            obtenido: 1            OK
[ 3] compatible(INT,FLOAT)              esperado: 0            obtenido: 0            OK
[ 4] compatible(INT,ERROR)              esperado: 0            obtenido: 0            OK
[ 5] nombre_tipo(INT)                   esperado: int          obtenido: int          OK
[ 6] nombre_tipo(ERROR)                 esperado: error        obtenido: error        OK

RESULTADO: 6/6 pruebas superadas
TODO EN VERDE. Documenta tu auditoria en auditorias/auditoria_sesion06.md
```

* [x] Caso base y límites superados sin *Crash*.
* [x] No existen accesos fuera de límites ni *Memory Leaks*.
* [x] La complejidad teórica coincide con la implementada.

### Reporte Crítico
*¿Qué caso borde o problema omitió la IA y cómo lo solucionaste?*
> El código de la IA pasa los 6 casos del harness y quedó prácticamente igual a mi versión manual, pero omitió el caso borde de **valores fuera del enum**. `compatible` solo revisa `TIPO_ERROR` y luego compara con `destino >= origen`, así que un destino inválido, por ejemplo `(Tipo)7`, resultaría compatible con cualquier tipo válido (7 >= 2), cuando debería rechazarse como si fuera error. `nombre_tipo` sí lo cubre gracias al `default`, pero `compatible` no. Lo detecté al revisar el código contra el criterio "solo acepta destinos de igual o mayor rango" y notar que la comparación numérica asume que ambos valores están dentro de 1..3. La solución es validar que `destino` y `origen` estén entre `TIPO_CHAR` y `TIPO_FLOAT` antes de comparar y devolver 0 en caso contrario. Además, el harness no prueba casos con `TIPO_CHAR` (por ejemplo `compatible(INT, CHAR)` = 1 y `compatible(CHAR, INT)` = 0) ni `compatible(ERROR, ERROR)`, así que convendría agregarlos. Complejidad: tiempo O(1) y espacio O(1), porque solo hay comparaciones de enteros y un `switch` que devuelve cadenas literales, sin memoria dinámica.

---
**Firma digital:** [mariajeje2006]
