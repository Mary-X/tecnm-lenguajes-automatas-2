# Bitácora de Requerimientos y Auditoría IA — Sesión 20
**Materia:** Lenguajes y Autómatas II (SCD-1016)  
**Tema:** Hito 2: Analizador Semántico y Scopes  
**Docente Titular:** MCC. Iván Márquez Larios (`ijmarquezl`)  
**Estudiante:** Maria Jesus Fuentes Valdez  
**Matrícula:** 24531315  
**Track:** C (el harness evalúa C; la versión en Rust es opcional y no se califica con este harness)  

---

## 1. Fase 1: El Artesano (Dificultad Manual)
*Reto:* Integración de Tabla de Símbolos y AST  
*Describe qué problema o restricción de memoria/sintaxis identificaste al codificar a mano:*
> Lo que más me costó fue `contar_visibles`, porque un nombre repetido en varios ámbitos solo debe contar una vez. `profundidad_de` fue más directa: subo por los `padre` con un contador de nivel, y el primer ámbito que tenga el nombre gana, así el shadowing sale solo (si `x` está en el bloque y en el global, regresa 0). Para contar sin repetir no quise usar un arreglo auxiliar de tamaño fijo, porque no sé cuántos ámbitos va a haber. Lo que hice fue: por cada símbolo, revisar si ya apareció en un ámbito **más interno** o antes en el mismo ámbito; si no apareció, lo cuento. Es más lento, pero no tiene límite. Los argumentos `NULL` regresan -2 en `profundidad_de`, como pide el contrato, y un nombre que no existe regresa -1.

---

## 2. Fase 2: El Científico (Ingeniería de Requerimientos C-R-E-O)

### 2.1 Matriz de Especificación Formal
| Elemento C-R-E-O | Definición Técnica |
| :--- | :--- |
| **Contexto (C)** | C99 / GCC o Rust 2021 sin bibliotecas no estándar |
| **Rol (R)** | Ingeniero de software de sistemas y compiladores |
| **Especificación (E)** | Contrato de Validación Semántica Integral |
| **Output (O)** | Solo código modular y análisis de complejidad |

### 2.2 Prompt Estructurado
```text
CONTEXTO: C11 con GCC (-Wall -Wextra -Werror), solo libc. La cabecera
api_sesion20.h define:
  #define AMBITO_MAX 8
  typedef struct Ambito { const char *nombre; const char *simbolos[AMBITO_MAX];
                          int total; struct Ambito *padre; } Ambito;
  int profundidad_de(const Ambito *actual, const char *nombre);
  int contar_visibles(const Ambito *actual);

ROL: Ingeniero de software de sistemas y compiladores.

ESPECIFICACIÓN: Resolución de nombres con shadowing (Hito 2).
 - profundidad_de: niveles hacia arriba donde está el nombre (0 = actual);
   -1 si no existe; -2 si algún argumento es NULL. El más interno gana.
 - contar_visibles: nombres distintos visibles desde el actual; cada nombre
   cuenta una sola vez aunque esté en varios ámbitos.
 - Sin main() y sin cambiar las firmas.

OUTPUT: Solo código C modular y un análisis breve de complejidad.
```

---

## 3. Código Devuelto por el Modelo
```c
// Código devuelto por el LLM
#include "api_sesion20.h"
#include <string.h>

#define MAX_VISIBLES 64

int profundidad_de(const Ambito *actual, const char *nombre) {
    if (actual == NULL || nombre == NULL) {
        return -2;
    }
    int profundidad = 0;
    while (actual != NULL) {
        for (int i = 0; i < actual->total; i++) {
            if (strcmp(actual->simbolos[i], nombre) == 0) {
                return profundidad;
            }
        }
        actual = actual->padre;
        profundidad++;
    }
    return -1;
}

int contar_visibles(const Ambito *actual) {
    const char *vistos[MAX_VISIBLES];
    int n = 0;
    for (const Ambito *a = actual; a != NULL; a = a->padre) {
        for (int i = 0; i < a->total; i++) {
            int nuevo = 1;
            for (int k = 0; k < n; k++) {
                if (strcmp(vistos[k], a->simbolos[i]) == 0) {
                    nuevo = 0;
                    break;
                }
            }
            if (nuevo && n < MAX_VISIBLES) {
                vistos[n++] = a->simbolos[i];
            }
        }
    }
    return n;
}
```

---

## 4. Fase 3: El Auditor (Resultados del Test Harness)
*Pruebas ejecutadas:* Casos complejos de shadowing y ámbito

### Salida del harness
```text
=== AUDITORIA SESION 20 - Resolucion de nombres y visibilidad ===
Caso                               esperado:              obtenido:              resultado
[ 1] nombre en el ambito actual         esperado: 0            obtenido: 0            OK
[ 2] nombre dos niveles arriba          esperado: 2            obtenido: 2            OK
[ 3] shadowing gana el interno          esperado: 0            obtenido: 0            OK
[ 4] nombre no visible                  esperado: -1           obtenido: -1           OK
[ 5] visibles sin repetir               esperado: 3            obtenido: 3            OK
[ 6] ambito NULL                        esperado: -2           obtenido: -2           OK

RESULTADO: 6/6 pruebas superadas
TODO EN VERDE. Documenta tu auditoria en auditorias/auditoria_sesion20.md
```

* [x] Caso base y límites superados sin *Crash*. (en los casos del harness; el reporte describe un caso fuera de ellos que sí truena)
* [x] No existen accesos fuera de límites ni *Memory Leaks*.
* [x] La complejidad teórica coincide con la implementada.

### Reporte Crítico
*¿Qué caso borde o problema omitió la IA y cómo lo solucionaste?*
> La IA pasó los 6 casos, pero encontré dos problemas. El primero: `contar_visibles` guarda los nombres en un arreglo fijo `vistos[64]` y, cuando se llena, **deja de contar sin avisar**. Lo probé con 10 ámbitos anidados de 8 nombres distintos cada uno (80 en total) y regresó **64**. El contrato no pone límite de profundidad, así que el resultado es incorrecto. El segundo: ninguna de las dos funciones revisa si un símbolo es `NULL`. Con un ámbito que tenía `total = 1` y `simbolos[0] = NULL`, `profundidad_de` se cayó con *segmentation fault* (código 139). Mi versión no usa arreglo auxiliar: para cada símbolo reviso si ya apareció en un ámbito más interno, así no hay límite, y me salto las entradas `NULL`. Complejidad: `profundidad_de` es O(p·k) y mi `contar_visibles` es O((p·k)²), donde p es la profundidad y k ≤ 8 los símbolos por ámbito; la de la IA es igual de cuadrática, pero con el tope de 64. Espacio O(1) en la mía.

---
**Firma digital:** [mariajeje2006]