# Bitácora de Requerimientos y Auditoría IA — Sesión 08
**Materia:** Lenguajes y Autómatas II (SCD-1016)  
**Tema:** Inferencia y Promoción de Tipos  
**Docente Titular:** MCC. Iván Márquez Larios (`ijmarquezl`)  
**Estudiante:** Maria Jesus Fuentes Valdez  
**Matrícula:** 24531315  
**Track:** C (el harness evalúa C; la versión en Rust es opcional y no se califica con este harness)  

---

## 1. Fase 1: El Artesano (Dificultad Manual)
*Reto:* Reglas de casteo implícito  
*Describe qué problema o restricción de memoria/sintaxis identificaste al codificar a mano:*
> Lo que más me costó fue entender que la promoción y la pérdida son dos preguntas distintas. `promover(a, b)` tiene que regresar el tipo "más grande" de los dos, y como el enum ya está ordenado (CHAR = 1, INT = 2, FLOAT = 3) pude usar `(a > b) ? a : b`. Pero `TIPO_ERROR` vale 0, así que si no lo revisaba primero, `promover(ERROR, INT)` habría dado INT, cuando lo correcto es que el error se contagie. En `hay_perdida(destino, origen)` me confundía el orden de los parámetros: hay pérdida cuando el destino es más angosto que el origen (`destino < origen`), por ejemplo meter un FLOAT en un INT. Con ERROR en cualquiera de los dos lados regreso 1, porque no se puede garantizar nada. En `nombre_tipo` usé `switch` con `default` para que no hubiera advertencias.

---

## 2. Fase 2: El Científico (Ingeniería de Requerimientos C-R-E-O)

### 2.1 Matriz de Especificación Formal
| Elemento C-R-E-O | Definición Técnica |
| :--- | :--- |
| **Contexto (C)** | C99 / GCC o Rust 2021 sin bibliotecas no estándar |
| **Rol (R)** | Ingeniero de software de sistemas y compiladores |
| **Especificación (E)** | Promoción entera y ensanchamiento |
| **Output (O)** | Solo código modular y análisis de complejidad |

### 2.2 Prompt Estructurado
```text
CONTEXTO: C11 con GCC (-Wall -Wextra -Werror), sin bibliotecas no estándar.
La cabecera api_sesion08.h define:
  typedef enum { TIPO_ERROR = 0, TIPO_CHAR = 1, TIPO_INT = 2, TIPO_FLOAT = 3 } Tipo;
  Tipo promover(Tipo a, Tipo b);
  int hay_perdida(Tipo destino, Tipo origen);
  const char *nombre_tipo(Tipo t);

ROL: Ingeniero de software de sistemas y compiladores.

ESPECIFICACIÓN:
 - promover devuelve el tipo más ancho de los dos (CHAR < INT < FLOAT);
   si alguno es TIPO_ERROR devuelve TIPO_ERROR.
 - hay_perdida devuelve 1 si asignar origen a destino pierde información
   (destino más angosto) o si hay TIPO_ERROR; 0 si es seguro.
 - nombre_tipo devuelve "char", "int", "float" o "error".
 - Sin main() y sin cambiar las firmas.

OUTPUT: Solo código C modular y un análisis breve de complejidad.
```

---

## 3. Código Devuelto por el Modelo
```c
// Código devuelto por el LLM
#include "api_sesion08.h"
#include <stddef.h>

Tipo promover(Tipo a, Tipo b) {
    if (a == TIPO_ERROR || b == TIPO_ERROR) {
        return TIPO_ERROR;
    }
    return (a > b) ? a : b;
}

int hay_perdida(Tipo destino, Tipo origen) {
    if (destino == TIPO_ERROR || origen == TIPO_ERROR) {
        return 1;
    }
    return (destino < origen) ? 1 : 0;
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
*Pruebas ejecutadas:* Mezcla de tipos enteros y flotantes

### Salida del harness
```text
=== AUDITORIA SESION 08 - Reglas de promocion y perdida ===
Caso                               esperado:              obtenido:              resultado
[ 1] promover(CHAR,INT)                 esperado: int          obtenido: int          OK
[ 2] promover(INT,FLOAT)                esperado: float        obtenido: float        OK
[ 3] promover(FLOAT,FLOAT)              esperado: float        obtenido: float        OK
[ 4] promover(ERROR,INT)                esperado: error        obtenido: error        OK
[ 5] hay_perdida(CHAR,INT)              esperado: 1            obtenido: 1            OK
[ 6] hay_perdida(INT,CHAR)              esperado: 0            obtenido: 0            OK

RESULTADO: 6/6 pruebas superadas
TODO EN VERDE. Documenta tu auditoria en auditorias/auditoria_sesion08.md
```

* [x] Caso base y límites superados sin *Crash*.
* [x] No existen accesos fuera de límites ni *Memory Leaks*.
* [x] La complejidad teórica coincide con la implementada.

### Reporte Crítico
*¿Qué caso borde o problema omitió la IA y cómo lo solucionaste?*
> La IA y yo hicimos casi lo mismo, y los dos pasamos los 6 casos, pero la regla "un destino más ancho nunca pierde" no es del todo cierta con **INT → FLOAT**. `hay_perdida(FLOAT, INT)` regresa 0, pero un `float` solo tiene 24 bits de mantisa, así que los enteros grandes pierden precisión. Lo comprobé convirtiendo `16777217` (2^24 + 1) a `float` y de regreso: salió `16777216`, o sea que sí se perdió una unidad. Para el alcance del curso la jerarquía CHAR < INT < FLOAT está bien, pero un compilador real (como GCC con `-Wconversion`) avisa en este caso. La solución sería tratar INT → FLOAT como "posible pérdida" (regresar un código distinto, tipo advertencia) o documentar en el contrato que solo se considera el rango y no la precisión. Complejidad: O(1) en tiempo y espacio, solo son comparaciones de enteros.

---
**Firma digital:** [mariajeje2006]