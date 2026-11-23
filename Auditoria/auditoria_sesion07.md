# Bitácora de Requerimientos y Auditoría IA — Sesión 07
**Materia:** Lenguajes y Autómatas II (SCD-1016)  
**Tema:** Evaluación de Expresiones Aritméticas  
**Docente Titular:** MCC. Iván Márquez Larios (`ijmarquezl`)  
**Estudiante:** Maria Jesus Fuentes Valdez  
**Matrícula:** 24531315  
**Track:** C (el harness evalúa C; la versión en Rust es opcional y no se califica con este harness)  

---

## 1. Fase 1: El Artesano (Dificultad Manual)
*Reto:* Evaluador recursivo de operaciones  
*Describe qué problema o restricción de memoria/sintaxis identificaste al codificar a mano:*
> Lo más difícil fue la propagación del error. Como `evaluar` es recursiva, si el hijo izquierdo ya marcó `*error` no tenía caso seguir calculando, así que después de cada llamada recursiva puse un `if (*error != 0) return 0;` para cortar la recursión y que el error suba sin recalcular a los padres. También tuve que revisar el divisor **antes** de hacer la división: si hacía `izq / der` primero y luego preguntaba, el programa ya se habría caído con la excepción de punto flotante. Le di el mismo trato al operador `%`, porque `7 % 0` también truena. Para el árbol `NULL` y para un operador que no conozco regreso `0` con `error = 2`, y al principio valido que el puntero `error` no sea `NULL` para no escribir en una dirección inválida.

---

## 2. Fase 2: El Científico (Ingeniería de Requerimientos C-R-E-O)

### 2.1 Matriz de Especificación Formal
| Elemento C-R-E-O | Definición Técnica |
| :--- | :--- |
| **Contexto (C)** | C99 / GCC o Rust 2021 sin bibliotecas no estándar |
| **Rol (R)** | Ingeniero de software de sistemas y compiladores |
| **Especificación (E)** | Manejo estricto de división entre cero |
| **Output (O)** | Solo código modular y análisis de complejidad |

### 2.2 Prompt Estructurado
```text
CONTEXTO: Trabajo en C11 con GCC y las banderas -Wall -Wextra -Werror, sin
bibliotecas externas. La cabecera api_sesion07.h define:
  typedef enum { NODO_NUM, NODO_BINOP } TipoNodo;
  typedef struct ASTNode { TipoNodo tipo; int valor; char op;
                           struct ASTNode *izq; struct ASTNode *der; } ASTNode;
  int evaluar(ASTNode *n, int *error);

ROL: Actúa como ingeniero de software de sistemas y compiladores.

ESPECIFICACIÓN: Implementa evaluar() como un evaluador recursivo del árbol.
 - Operadores: + - * / y %.
 - Antes de dividir o sacar módulo revisa el divisor; si es 0 pon *error = 1
   y regresa 0 sin hacer la operación.
 - Si el nodo es NULL, el tipo no es válido o el operador no se conoce,
   pon *error = 2.
 - Si un hijo ya marcó error, no sigas evaluando: el error sube tal cual.
 - No escribas main() ni cambies la firma.

OUTPUT: Solo el código C modular y un análisis breve de complejidad.
```

---

## 3. Código Devuelto por el Modelo
```c
// Código devuelto por el LLM
#include "api_sesion07.h"
#include <stddef.h>

int evaluar(ASTNode *n, int *error) {
    if (error == NULL) {
        return 0;
    }

    // Si el nodo de entrada es nulo, reportamos error tipo 2
    if (n == NULL) {
        *error = 2;
        return 0;
    }

    // Caso base: Nodo de tipo numérico
    if (n->tipo == NODO_NUM) {
        return n->valor;
    }

    // Caso inductivo: Operaciones binarias
    if (n->tipo == NODO_BINOP) {
        // Evaluar el subárbol izquierdo
        int izq_val = evaluar(n->izq, error);
        if (*error != 0) {
            return 0;
        }

        // Evaluar el subárbol derecho
        int der_val = evaluar(n->der, error);
        if (*error != 0) {
            return 0;
        }

        // Evaluar la operación actual con protección de errores
        switch (n->op) {
            case '+':
                return izq_val + der_val;
            case '-':
                return izq_val - der_val;
            case '*':
                return izq_val * der_val;
            case '/':
                if (der_val == 0) {
                    *error = 1; // Error por división entre cero
                    return 0;
                }
                return izq_val / der_val;
            case '%':
                if (der_val == 0) {
                    *error = 1; // Error por módulo entre cero
                    return 0;
                }
                return izq_val % der_val;
            default:
                *error = 2; // Operación inválida / desconocida
                return 0;
        }
    }

    // Tipo de nodo no soportado
    *error = 2;
    return 0;
}
```

---

## 4. Fase 3: El Auditor (Resultados del Test Harness)
*Pruebas ejecutadas:* Expresiones complejas y división por 0

### Salida del harness
```text
=== AUDITORIA SESION 07 - Evaluador recursivo con deteccion de errores ===
Caso                               esperado:              obtenido:              resultado
[ 1] (2+3)*4                            esperado: 20           obtenido: 20           OK
[ 2] 7/0 -> error 1                     esperado: 1            obtenido: 1            OK
[ 3] 7%0 -> error 1                     esperado: 1            obtenido: 1            OK
[ 4] arbol NULL -> error 2              esperado: 2            obtenido: 2            OK
[ 5] 10-2-3                             esperado: 5            obtenido: 5            OK
[ 6] (8/2)+1                            esperado: 5            obtenido: 5            OK

RESULTADO: 6/6 pruebas superadas
TODO EN VERDE. Documenta tu auditoria en auditorias/auditoria_sesion07.md
```

* [x] Caso base y límites superados sin *Crash*. (en los casos del harness; el reporte describe un caso fuera de ellos que sí truena)
* [x] No existen accesos fuera de límites ni *Memory Leaks*.
* [x] La complejidad teórica coincide con la implementada.

### Reporte Crítico
*¿Qué caso borde o problema omitió la IA y cómo lo solucionaste?*
> La IA pasó los 6 casos, pero se le pasó un caso de **desbordamiento en la división**: además de dividir entre 0, en C también es indefinido dividir `INT_MIN / -1`, porque el resultado (2147483648) no cabe en un `int`. Lo probé armando un nodo `/` con `INT_MIN` a la izquierda y `-1` a la derecha, y el programa se cayó con *Floating point exception* (código de salida 136). O sea, el evaluador "no se cae ante una división entre cero", pero sí con este otro caso, y lo mismo pasa con `INT_MIN % -1`. La solución es agregar en `/` y `%` la condición `if (izq_val == INT_MIN && der_val == -1)` y marcar error antes de operar (o calcular en `long` y revisar el rango). Otro detalle: la función no pone `*error = 0` al empezar, así que depende de que quien la llama lo inicialice. Complejidad: tiempo O(n) porque visita cada nodo una vez, y espacio O(h) por la pila de recursión, donde h es la altura del árbol.

---
**Firma digital:** [mariajeje2006]