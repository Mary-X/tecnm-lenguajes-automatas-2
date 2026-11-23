# Bitácora de Requerimientos y Auditoría IA — Sesión 15
**Materia:** Lenguajes y Autómatas II (SCD-1016)  
**Tema:** Bloques Anidados y Ciclo de Vida  
**Docente Titular:** MCC. Iván Márquez Larios (`ijmarquezl`)  
**Estudiante:** Maria Jesus Fuentes Valdez  
**Matrícula:** 24531315  
**Track:** C (el harness evalúa C; la versión en Rust es opcional y no se califica con este harness)  

---

## 1. Fase 1: El Artesano (Dificultad Manual)
*Reto:* Operaciones Push y Pop de ámbito  
*Describe qué problema o restricción de memoria/sintaxis identificaste al codificar a mano:*
> Lo que más me costó fue entender quién es dueño de la memoria. `push_ambito` reserva con `malloc` un ámbito nuevo, le pone el nombre y lo enlaza con el actual como su `padre`. `pop_ambito` hace lo contrario: guarda primero el puntero al padre, libera el actual con `free` y regresa el padre. Si liberaba antes de guardar el padre, estaría leyendo memoria ya liberada. El otro detalle fue el global: según el contrato, el ámbito raíz no se puede sacar, así que si `actual->padre` es `NULL` regreso `NULL` sin liberar nada. También valido que el nombre y el ámbito actual no sean `NULL` en `push`, y que `malloc` no haya fallado.

---

## 2. Fase 2: El Científico (Ingeniería de Requerimientos C-R-E-O)

### 2.1 Matriz de Especificación Formal
| Elemento C-R-E-O | Definición Técnica |
| :--- | :--- |
| **Contexto (C)** | C99 / GCC o Rust 2021 sin bibliotecas no estándar |
| **Rol (R)** | Ingeniero de software de sistemas y compiladores |
| **Especificación (E)** | Liberación de memoria al cerrar bloque |
| **Output (O)** | Solo código modular y análisis de complejidad |

### 2.2 Prompt Estructurado
```text
CONTEXTO: C11 con GCC (-Wall -Wextra -Werror), solo libc. La cabecera
api_sesion15.h define:
  typedef struct Ambito { const char *nombre; struct Ambito *padre; } Ambito;
  Ambito *push_ambito(Ambito *actual, const char *nombre);
  Ambito *pop_ambito(Ambito *actual);

ROL: Ingeniero de software de sistemas y compiladores.

ESPECIFICACIÓN: Ciclo de vida de bloques anidados.
 - push_ambito reserva un ámbito nuevo con malloc, cuyo padre es actual;
   devuelve NULL si actual o nombre son NULL o si falla malloc.
 - pop_ambito libera el ámbito actual y devuelve su padre; con NULL o con
   la raíz (sin padre) devuelve NULL y no libera nada.
 - Sin fugas de memoria. Sin main() y sin cambiar las firmas.

OUTPUT: Solo código C modular y un análisis breve de complejidad.
```

---

## 3. Código Devuelto por el Modelo
```c
// Código devuelto por el LLM
#include "api_sesion15.h"
#include <stdlib.h>

Ambito *push_ambito(Ambito *actual, const char *nombre) {
    if (actual == NULL || nombre == NULL) {
        return NULL;
    }

    Ambito *nuevo = (Ambito *)malloc(sizeof(Ambito));
    if (nuevo == NULL) {
        return NULL;
    }

    nuevo->nombre = nombre;
    nuevo->padre = actual;
    return nuevo;
}

Ambito *pop_ambito(Ambito *actual) {
    if (actual == NULL || actual->padre == NULL) {
        return NULL;
    }

    Ambito *padre = actual->padre;
    free(actual);
    return padre;
}
```

---

## 4. Fase 3: El Auditor (Resultados del Test Harness)
*Pruebas ejecutadas:* Aislamiento de variables fuera de bloque

### Salida del harness
```text
=== AUDITORIA SESION 15 - Push y pop de ambitos ===
Caso                               esperado:              obtenido:              resultado
[ 1] nombre del ambito nuevo            esperado: if           obtenido: if           OK
[ 2] profundidad con dos push           esperado: 3            obtenido: 3            OK
[ 3] pop devuelve el padre              esperado: 1            obtenido: 1            OK
[ 4] pop sobre la raiz                  esperado: 1            obtenido: 1            OK
[ 5] push con padre NULL                esperado: 1            obtenido: 1            OK
[ 6] profundidad tras pop               esperado: 2            obtenido: 2            OK

RESULTADO: 6/6 pruebas superadas
TODO EN VERDE. Documenta tu auditoria en auditorias/auditoria_sesion15.md
```

* [x] Caso base y límites superados sin *Crash*. (en los casos del harness; el reporte describe un caso fuera de ellos que sí truena)
* [x] No existen accesos fuera de límites ni *Memory Leaks*.
* [x] La complejidad teórica coincide con la implementada.

### Reporte Crítico
*¿Qué caso borde o problema omitió la IA y cómo lo solucionaste?*
> La IA pasó los 6 casos, pero `pop_ambito` hace `free` a **cualquier** ámbito que tenga padre, aunque no lo haya creado `push_ambito`. Lo probé con dos ámbitos en la pila de C (`Ambito g = {"global", NULL}; Ambito b = {"bloque", &g};`) y al llamar `pop_ambito(&b)` el programa abortó con `free(): invalid pointer` (código 134). En el harness funciona porque los bloques siempre salen de `push`, pero si alguien arma un bloque a mano se cae. Además, como `push_ambito(NULL, ...)` regresa `NULL`, el ámbito global nunca se puede crear con `push`, siempre se hace a mano, así que la confusión es fácil. Lo solucionaría documentando en el contrato que `pop` solo acepta ámbitos creados con `push`, o agregando un campo `dinamico` que `push` ponga en 1 para que `pop` solo libere esos. Complejidad: O(1) en tiempo y espacio para las dos operaciones.

---
**Firma digital:** [mariajeje2006]