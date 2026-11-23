# Bitácora de Requerimientos y Auditoría IA — Sesión 04
**Materia:** Lenguajes y Autómatas II (SCD-1016)  
**Tema:** Anatomía de un Árbol Sintáctico (AST)  
**Docente Titular:** MCC. Iván Márquez Larios (`ijmarquezl`)  
**Estudiante:** Maria Jesus Fuentes Valdez  
**Matrícula:** 24531315  
**Track:** C (el harness evalúa C; la versión en Rust es opcional y no se califica con este harness)  

---

## 1. Fase 1: El Artesano (Dificultad Manual)
*Reto:* Estructura de nodo binario (ASTNode)  
*Describe qué problema o restricción de memoria/sintaxis identificaste al codificar a mano:*
> Un solo struct debe representar dos tipos de nodo (número y operación binaria). Lo resolví con el campo `tipo` (`NODO_NUM` / `NODO_BINOP`) e inicialicé todos los campos: en un número, `op = '\0'` e hijos en `NULL`; en una operación, `valor = 0`. Sin esto, `malloc` deja basura en los campos que no se usan.
>
> Cada nodo se reserva en el heap con `malloc`, por lo que hay que revisar si devuelve `NULL` antes de escribir en él, y alguien debe liberar el árbol después (el contrato no incluye una función para ello).
>
> `evaluar` es recursiva: evalúa ambos hijos (postorden) y después aplica el operador. Los casos de paro son el nodo número y `NULL` (`evaluar(NULL)` devuelve -1 sin caerse).
>
> Me atoré al compilar antes de crear los archivos: gcc marcó `No such file or directory` para `harness_sesion04_c.c` y `api_sesion04.h`. Se resolvió ejecutando primero las celdas `%%writefile`.

---

## 2. Fase 2: El Científico (Ingeniería de Requerimientos C-R-E-O)

### 2.1 Matriz de Especificación Formal
| Elemento C-R-E-O | Definición Técnica |
| :--- | :--- |
| **Contexto (C)** | C99 / GCC o Rust 2021 sin bibliotecas no estándar |
| **Rol (R)** | Ingeniero de software de sistemas y compiladores |
| **Especificación (E)** | Gramática E -> E + T \| T con tipos de nodo |
| **Output (O)** | Solo código modular y análisis de complejidad |

### 2.2 Prompt Estructurado
```text
Contexto: Trabajo en C con GCC (-std=c11 -Wall -Wextra -Werror), sin bibliotecas no estándar. Mi archivo no debe tener main(), porque el main() es de un harness externo. Ya existe api_sesion04.h con este contrato:

typedef enum { NODO_NUM, NODO_BINOP } TipoNodo;
typedef struct ASTNode { TipoNodo tipo; int valor; char op; struct ASTNode *izq; struct ASTNode *der; } ASTNode;
ASTNode *crear_numero(int v);
ASTNode *crear_binario(char op, ASTNode *izq, ASTNode *der);
int evaluar(ASTNode *n);

Rol: Actúa como ingeniero de software de sistemas y compiladores.

Especificación: Implementa las tres funciones con las firmas exactas. Cada nodo se reserva con malloc. crear_binario conserva el orden izquierdo/derecho. evaluar evalúa ambos hijos y después aplica +, - o *. evaluar(NULL) devuelve -1 sin provocar fallo de segmentación. Usa el campo tipo para distinguir número de operación, sin campos mágicos.

Output: Solo el código del archivo .c (sin main), modular y con comentarios breves, y al final un análisis de complejidad en tiempo y espacio.
```

---

## 3. Código Devuelto por el Modelo
```c
// Código devuelto por el LLM
#include "api_sesion04.h"
#include <stdlib.h>

ASTNode *crear_numero(int v) {
    ASTNode *nuevo = (ASTNode *)malloc(sizeof(ASTNode));
    if (nuevo == NULL) {
        return NULL;
    }
    nuevo->tipo = NODO_NUM;
    nuevo->valor = v;
    nuevo->op = '\0';
    nuevo->izq = NULL;
    nuevo->der = NULL;
    return nuevo;
}

ASTNode *crear_binario(char op, ASTNode *izq, ASTNode *der) {
    ASTNode *nuevo = (ASTNode *)malloc(sizeof(ASTNode));
    if (nuevo == NULL) {
        return NULL;
    }
    nuevo->tipo = NODO_BINOP;
    nuevo->valor = 0;
    nuevo->op = op;
    nuevo->izq = izq;
    nuevo->der = der;
    return nuevo;
}

int evaluar(ASTNode *n) {
    if (n == NULL) {
        return -1;
    }
    if (n->tipo == NODO_NUM) {
        return n->valor;
    }
    if (n->tipo == NODO_BINOP) {
        int l_val = evaluar(n->izq);
        int r_val = evaluar(n->der);
        if (n->op == '+') return l_val + r_val;
        if (n->op == '-') return l_val - r_val;
        if (n->op == '*') return l_val * r_val;
    }
    return -1;
}
```

---

## 4. Fase 3: El Auditor (Resultados del Test Harness)
*Pruebas ejecutadas:* Árbol vacío y evaluación de nodo raíz

### Salida del harness
```text
=== AUDITORIA SESION 04 - Nodo binario de AST ===
Caso                               esperado:              obtenido:              resultado
[ 1] evaluar(nodo 42)                   esperado: 42           obtenido: 42           OK
[ 2] evaluar(3+5*2)                     esperado: 13           obtenido: 13           OK
[ 3] hijo izquierdo de (* 5 2)          esperado: 5            obtenido: 5            OK
[ 4] operador guardado                  esperado: *            obtenido: *            OK
[ 5] evaluar(NULL)                      esperado: -1           obtenido: -1           OK
[ 6] evaluar(10-4)                      esperado: 6            obtenido: 6            OK

RESULTADO: 6/6 pruebas superadas
TODO EN VERDE. Documenta tu auditoria en auditorias/auditoria_sesion04.md
```

* [x] Caso base y límites superados sin *Crash*.
* [ ] No existen accesos fuera de límites ni *Memory Leaks*. *(El harness no libera los nodos; verifica con `valgrind --leak-check=full ./test_ia` y marca solo si aplica.)*
* [x] La complejidad teórica coincide con la implementada: `crear_numero` y `crear_binario` son O(1); `evaluar` es O(n) en tiempo y O(h) en pila (h = altura del árbol).

### Reporte Crítico
*¿Qué caso borde o problema omitió la IA y cómo lo solucionaste?*
> El código de la IA pasa los 6 casos y es prácticamente igual al del Artesano, pero omitió lo que el harness no mide:
>
> 1. **No hay función para liberar el árbol.** Cada nodo se reserva con `malloc` y nada lo libera, así que hay memory leaks. Lo detecté al revisar que el contrato exige nodos en el heap y ninguna función los libera. Solución: agregar `void liberar_arbol(ASTNode *n)` (postorden: hijos, luego el nodo).
> 2. **El -1 es ambiguo.** `evaluar(NULL)` devuelve -1, pero -1 también es un resultado válido (por ejemplo 3-4). Así, `crear_binario('+', crear_numero(3), NULL)` da 2 sin avisar del error. Solución propuesta: reportar el error por separado (por ejemplo `int evaluar(ASTNode *n, int *resultado)`), aunque cambiaría el contrato.
> 3. **`crear_binario` no valida entradas:** no comprueba hijos `NULL` ni que `op` sea `+`, `-` o `*`; y si `malloc` falla, los hijos recibidos quedan sin liberar.
> 4. **Desbordamiento de `int`** en sumas y multiplicaciones (comportamiento indefinido en C).
> 5. **Recursión profunda:** un árbol muy profundo puede agotar la pila.

---
**Firma digital:** mariajeje2006
