# Bitácora de Requerimientos y Auditoría IA — Sesión 22
**Materia:** Lenguajes y Autómatas II (SCD-1016)  
**Tema:** Generación de TAC para Expresiones  
**Docente Titular:** MCC. Iván Márquez Larios (`ijmarquezl`)  
**Estudiante:** Maria Jesus Fuentes Valdez  
**Matrícula:** 24531315  
**Track:** C (el harness evalúa C; la versión en Rust es opcional y no se califica con este harness)  

---

## 1. Fase 1: El Artesano (Dificultad Manual)
*Reto:* Traductor de nodos AST a TAC  
*Describe qué problema o restricción de memoria/sintaxis identificaste al codificar a mano:*
> La clave fue el **postorden**: primero se traducen los dos hijos, luego se pide el temporal y al final se escribe la línea del operador. Si pedía el temporal antes de bajar a los hijos, en `a + b * c` el `+` se quedaba con `_t0` y el `*` con `_t1`, al revés de lo que espera el harness. Hice una función recursiva `traducir()` que regresa el "nombre" del nodo: si es hoja, su valor (las hojas no generan línea), y si es operador, el temporal que le tocó. Así el padre arma `"_t1 = a + _t0"` con lo que le regresan los hijos. Como tenía que llevar varias cosas a la vez (las líneas, el máximo, cuántas llevo, el contador y el `Pool`), las metí en un `struct Estado` en lugar de pasar seis parámetros. Antes de escribir reviso que `emitidas < max`, y uso `snprintf` con 64 para no pasarme del ancho de la línea.

---

## 2. Fase 2: El Científico (Ingeniería de Requerimientos C-R-E-O)

### 2.1 Matriz de Especificación Formal
| Elemento C-R-E-O | Definición Técnica |
| :--- | :--- |
| **Contexto (C)** | C99 / GCC o Rust 2021 sin bibliotecas no estándar |
| **Rol (R)** | Ingeniero de software de sistemas y compiladores |
| **Especificación (E)** | Conservación de precedencia de operadores |
| **Output (O)** | Solo código modular y análisis de complejidad |

### 2.2 Prompt Estructurado
```text
CONTEXTO: C11 con GCC (-Wall -Wextra -Werror), solo libc. La cabecera
api_sesion22.h define:
  typedef struct Nodo { char tipo; char op; char valor[16];
                        struct Nodo *izq; struct Nodo *der; } Nodo;
  int generar_tac(const Nodo *raiz, char lineas[][64], int max);
tipo es 'H' para hoja o 'B' para operador binario. El archivo ya trae un
Pool y const char *nuevo_temporal(Pool *p, int *contador) que se deben usar
tal cual.

ROL: Ingeniero de software de sistemas y compiladores.

ESPECIFICACIÓN: Traductor de AST a TAC con precedencia.
 - Recorrido en postorden: hijos primero, luego el operador.
 - Cada operador usa un temporal nuevo de nuevo_temporal y escribe
   "_tK = izq op der" en lineas[n]; las hojas no generan línea.
 - Nunca más de max líneas ni más de 63 caracteres por línea.
 - Devuelve cuántas líneas emitió. Sin main() y sin cambiar la firma.

OUTPUT: Solo código C modular y un análisis breve de complejidad.
```

---

## 3. Código Devuelto por el Modelo
```c
// Código devuelto por el LLM
#include "api_sesion22.h"
#include <stdio.h>
#include <string.h>

/* Los temporales ya estan resueltos: usalos tal cual. */
typedef struct {
    char nombre[64][16];
    int usados;
} Pool;

const char *nuevo_temporal(Pool *p, int *contador) {
    if (p->usados >= 64) return "_t?";
    snprintf(p->nombre[p->usados], sizeof p->nombre[0], "_t%d", (*contador)++);
    return p->nombre[p->usados++];
}

static const char *emitir(const Nodo *n, char lineas[][64], int max, int *k, Pool *p, int *cont) {
    if (n->tipo == 'H') {
        return n->valor;
    }
    const char *a = emitir(n->izq, lineas, max, k, p, cont);
    const char *b = emitir(n->der, lineas, max, k, p, cont);
    const char *t = nuevo_temporal(p, cont);
    if (*k < max) {
        snprintf(lineas[*k], 64, "%s = %s %c %s", t, a, n->op, b);
        (*k)++;
    }
    return t;
}

int generar_tac(const Nodo *raiz, char lineas[][64], int max) {
    if (raiz == NULL) {
        return 0;
    }
    Pool p = {0};
    int contador = 0, k = 0;
    emitir(raiz, lineas, max, &k, &p, &contador);
    return k;
}
```

---

## 4. Fase 3: El Auditor (Resultados del Test Harness)
*Pruebas ejecutadas:* Expresiones anidadas con paréntesis

### Salida del harness
```text
=== AUDITORIA SESION 22 - Traductor de AST a TAC con precedencia ===
Caso                               esperado:              obtenido:              resultado
[ 1] a + b * c                          esperado: _t0 = b * c  obtenido: _t0 = b * c  OK
[ 2] a + b * c (2a linea)
     esperado: _t1 = a + _t0
     obtenido: _t1 = a + _t0
     -> OK
[ 3] (a + b) * c
     esperado: _t1 = _t0 * c
     obtenido: _t1 = _t0 * c
     -> OK
[ 4] x * (y - z) + w
     esperado: _t2 = _t1 + w
     obtenido: _t2 = _t1 + w
     -> OK
[ 5] hoja sin operaciones               esperado: 0            obtenido: 0            OK
[ 6] (a + b) * (c - d)
     esperado: _t2 = _t0 * _t1
     obtenido: _t2 = _t0 * _t1
     -> OK

RESULTADO: 6/6 pruebas superadas
TODO EN VERDE. Documenta tu auditoria en auditorias/auditoria_sesion22.md
```

* [x] Caso base y límites superados sin *Crash*. (en los casos del harness; el reporte describe un caso fuera de ellos que sí truena)
* [x] No existen accesos fuera de límites ni *Memory Leaks*.
* [x] La complejidad teórica coincide con la implementada.

### Reporte Crítico
*¿Qué caso borde o problema omitió la IA y cómo lo solucionaste?*
> La IA pasó los 6 casos, pero su función recursiva `emitir()` hace `n->tipo` **sin revisar si el nodo es `NULL`**. Solo revisa la raíz en `generar_tac`. Lo probé con un operador `-` que tenía `izq = a` y `der = NULL` (un árbol mal armado, como el que dejaría el parser después de un error de sintaxis) y el programa se cayó con *segmentation fault* (código 139). En mi versión, `traducir()` regresa `"?"` si el nodo es `NULL`, así la línea sale como `_t0 = a - ?` y el error se ve en el TAC en vez de tumbar el compilador. Tampoco valida `lineas == NULL` ni `max <= 0`; con `max` negativo no escribe nada, pero sería mejor rechazarlo desde el principio. Las dos versiones comparten un límite: el `Pool` solo tiene 64 nombres y después de eso todos los temporales salen como `_t?`. Complejidad: O(n) en tiempo, porque visita cada nodo una vez, y O(h) en espacio por la recursión, donde h es la altura del árbol.

---
**Firma digital:** [mariajeje2006]