# Bitácora de Requerimientos y Auditoría IA — Sesión 14
**Materia:** Lenguajes y Autómatas II (SCD-1016)  
**Tema:** Declaración vs Uso de Variables  
**Docente Titular:** MCC. Iván Márquez Larios (`ijmarquezl`)  
**Estudiante:** Maria Jesus Fuentes Valdez  
**Matrícula:** 24531315  
**Track:** C (el harness evalúa C; la versión en Rust es opcional y no se califica con este harness)  

---

## 1. Fase 1: El Artesano (Dificultad Manual)
*Reto:* Detección de variables no declaradas  
*Describe qué problema o restricción de memoria/sintaxis identificaste al codificar a mano:*
> La parte difícil fue que la comparación tenía que ser **exacta**. Al principio pensé en `strncmp` con el largo del nombre buscado, pero con eso `"x"` y `"xy"` se veían iguales, y el harness justo prueba eso (caso 5). Con `strcmp` se arregla, porque compara hasta el `'\0'`. También tuve que respetar los tres valores de `declarar`: 1 si lo insertó, 0 si ya estaba y -1 si me pasan `NULL` o un nombre vacío. Para el nombre vacío revisé `strlen(nombre) == 0` antes de buscar. En `usar` cualquier problema regresa 0, porque solo me importa si la variable fue declarada o no. El `total` solo sube cuando de verdad se inserta, por eso el caso "total tras redeclarar" da 1.

---

## 2. Fase 2: El Científico (Ingeniería de Requerimientos C-R-E-O)

### 2.1 Matriz de Especificación Formal
| Elemento C-R-E-O | Definición Técnica |
| :--- | :--- |
| **Contexto (C)** | C99 / GCC o Rust 2021 sin bibliotecas no estándar |
| **Rol (R)** | Ingeniero de software de sistemas y compiladores |
| **Especificación (E)** | Invariante de definición previa |
| **Output (O)** | Solo código modular y análisis de complejidad |

### 2.2 Prompt Estructurado
```text
CONTEXTO: C11 con GCC (-Wall -Wextra -Werror), solo libc. La cabecera
api_sesion14.h define:
  typedef struct Tabla { const char *nombres[TABLA_MAX]; int total; } Tabla;
  int declarar(Tabla *tabla, const char *nombre);
  int usar(const Tabla *tabla, const char *nombre);

ROL: Ingeniero de software de sistemas y compiladores.

ESPECIFICACIÓN: Invariante de definición previa.
 - declarar devuelve 1 si insertó, 0 si el nombre ya estaba y -1 con NULL o
   nombre vacío.
 - usar devuelve 1 solo si el nombre ya fue declarado; si no, 0.
 - La comparación es exacta: "x" y "xy" son distintos.
 - Sin main() y sin cambiar las firmas.

OUTPUT: Solo código C modular y un análisis breve de complejidad.
```

---

## 3. Código Devuelto por el Modelo
```c
// Código devuelto por el LLM
#include "api_sesion14.h"
#include <string.h>
#include <stddef.h>

int declarar(Tabla *tabla, const char *nombre) {
    if (tabla == NULL || nombre == NULL || strlen(nombre) == 0) {
        return -1;
    }

    if (tabla->total >= TABLA_MAX) {
        return 0;
    }

    // Comprobar si el identificador ya existe en la tabla
    for (int i = 0; i < tabla->total; i++) {
        if (strcmp(tabla->nombres[i], nombre) == 0) {
            return 0;
        }
    }

    // Registrar nuevo símbolo
    tabla->nombres[tabla->total] = nombre;
    tabla->total++;
    return 1;
}

int usar(const Tabla *tabla, const char *nombre) {
    if (tabla == NULL || nombre == NULL || strlen(nombre) == 0) {
        return 0;
    }

    // Buscar secuencialmente en la tabla
    for (int i = 0; i < tabla->total; i++) {
        if (strcmp(tabla->nombres[i], nombre) == 0) {
            return 1;
        }
    }

    return 0;
}
```

---

## 4. Fase 3: El Auditor (Resultados del Test Harness)
*Pruebas ejecutadas:* Uso antes de declarar y redeclaración

### Salida del harness
```text
=== AUDITORIA SESION 14 - Declaracion previa y redeclaracion ===
Caso                               esperado:              obtenido:              resultado
[ 1] usar sin declarar                  esperado: 0            obtenido: 0            OK
[ 2] usar despues de declarar           esperado: 1            obtenido: 1            OK
[ 3] redeclarar el mismo nombre         esperado: 0            obtenido: 0            OK
[ 4] total tras redeclarar              esperado: 1            obtenido: 1            OK
[ 5] prefijo no es el mismo nombre      esperado: 0            obtenido: 0            OK
[ 6] declarar nombre vacio              esperado: -1           obtenido: -1           OK

RESULTADO: 6/6 pruebas superadas
TODO EN VERDE. Documenta tu auditoria en auditorias/auditoria_sesion14.md
```

* [x] Caso base y límites superados sin *Crash*.
* [x] No existen accesos fuera de límites ni *Memory Leaks*.
* [x] La complejidad teórica coincide con la implementada.

### Reporte Crítico
*¿Qué caso borde o problema omitió la IA y cómo lo solucionaste?*
> La IA pasó los 6 casos, pero cuando la tabla se llena `declarar` regresa **0, el mismo código que "ya estaba declarado"**. Lo probé declarando `v0` hasta `v16` con `TABLA_MAX = 16`: `v15` entró bien (1), pero `v16` regresó 0, y luego `usar(&t, "v16")` también dio 0. O sea, el analizador pensaría que `v16` es una redeclaración, cuando en realidad es una variable nueva que no cupo, y después reportaría "variable no declarada" al usarla. Son dos errores falsos por un problema de capacidad. Lo solucionaría regresando -1 (o un código propio, como -2) cuando `total >= TABLA_MAX`, y revisando la capacidad **después** de buscar duplicados, como hice en la sesión 13. Mi versión manual tiene el mismo detalle. Complejidad: O(n) en tiempo para las dos funciones, con n ≤ TABLA_MAX, y O(1) en espacio extra.

---
**Firma digital:** [mariajeje2006]