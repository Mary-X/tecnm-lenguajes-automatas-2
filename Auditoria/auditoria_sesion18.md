# Bitácora de Requerimientos y Auditoría IA — Sesión 18
**Materia:** Lenguajes y Autómatas II (SCD-1016)  
**Tema:** Arreglos y Verificación de Límites  
**Docente Titular:** MCC. Iván Márquez Larios (`ijmarquezl`)  
**Estudiante:** Maria Jesus Fuentes Valdez  
**Matrícula:** 24531315  
**Track:** C (el harness evalúa C; la versión en Rust es opcional y no se califica con este harness)  

---

## 1. Fase 1: El Artesano (Dificultad Manual)
*Reto:* Cálculo de tamaño de arreglo en memoria  
*Describe qué problema o restricción de memoria/sintaxis identificaste al codificar a mano:*
> Lo primero que tuve que pensar fue de dónde saco el ancho de cada tipo. En lugar de hacer una cadena de `if`, armé una tabla pequeña con pares `{nombre, ancho}` (char 1, short 2, int 4, float 4, double 8, puntero 8) y la recorro con `strcmp`. Si no encuentro el nombre, regreso -1. El otro detalle fue el tipo de dato: `bytes_arreglo` regresa `long`, así que hice la multiplicación en `long` (`ancho * (long)elementos`), porque si la hago en `int`, un arreglo grande se desborda antes de convertirse. Para `indice_en_rango` lo que más me confundía era el límite de arriba: el último índice válido es `elementos - 1`, así que `indice == elementos` ya está fuera, igual que cualquier índice negativo. Los elementos negativos regresan -1 y cero elementos pesan 0 bytes.

---

## 2. Fase 2: El Científico (Ingeniería de Requerimientos C-R-E-O)

### 2.1 Matriz de Especificación Formal
| Elemento C-R-E-O | Definición Técnica |
| :--- | :--- |
| **Contexto (C)** | C99 / GCC o Rust 2021 sin bibliotecas no estándar |
| **Rol (R)** | Ingeniero de software de sistemas y compiladores |
| **Especificación (E)** | Indexación entera no negativa |
| **Output (O)** | Solo código modular y análisis de complejidad |

### 2.2 Prompt Estructurado
```text
CONTEXTO: C11 con GCC (-Wall -Wextra -Werror), arquitectura x86-64, solo
libc. La cabecera api_sesion18.h declara:
  long bytes_arreglo(const char *tipo, int elementos);
  int indice_en_rango(int indice, int elementos);

ROL: Ingeniero de software de sistemas y compiladores.

ESPECIFICACIÓN: Tamaño de arreglos y verificación de límites estáticos.
 - Tipos válidos: char 1, short 2, int 4, float 4, double 8, puntero 8 bytes;
   otro nombre o NULL devuelve -1.
 - bytes_arreglo: elementos negativos devuelven -1; cero elementos pesan 0.
 - indice_en_rango: 1 si 0 <= indice <= elementos - 1, si no 0.
 - Sin main() y sin cambiar las firmas.

OUTPUT: Solo código C modular y un análisis breve de complejidad.
```

---

## 3. Código Devuelto por el Modelo
```c
// Código devuelto por el LLM
#include "api_sesion18.h"
#include <string.h>

static long ancho_de(const char *tipo) {
    if (strcmp(tipo, "char") == 0) return 1;
    if (strcmp(tipo, "short") == 0) return 2;
    if (strcmp(tipo, "int") == 0 || strcmp(tipo, "float") == 0) return 4;
    if (strcmp(tipo, "double") == 0 || strcmp(tipo, "puntero") == 0) return 8;
    return 0;
}

long bytes_arreglo(const char *tipo, int elementos) {
    if (tipo == NULL || elementos < 0) {
        return -1;
    }
    long ancho = ancho_de(tipo);
    if (ancho == 0) {
        return -1;
    }
    return ancho * elementos;
}

int indice_en_rango(int indice, int elementos) {
    return (unsigned)indice < (unsigned)elementos;
}
```

---

## 4. Fase 3: El Auditor (Resultados del Test Harness)
*Pruebas ejecutadas:* Límites estáticos y dimensiones inválidas

### Salida del harness
```text
=== AUDITORIA SESION 18 - Tamano de arreglo y limites ===
Caso                               esperado:              obtenido:              resultado
[ 1] int[10]                            esperado: 40           obtenido: 40           OK
[ 2] double[3]                          esperado: 24           obtenido: 24           OK
[ 3] int[-4]                            esperado: -1           obtenido: -1           OK
[ 4] indice 0 de 5                      esperado: 1            obtenido: 1            OK
[ 5] indice 5 de 5                      esperado: 0            obtenido: 0            OK
[ 6] indice negativo                    esperado: 0            obtenido: 0            OK

RESULTADO: 6/6 pruebas superadas
TODO EN VERDE. Documenta tu auditoria en auditorias/auditoria_sesion18.md
```

* [x] Caso base y límites superados sin *Crash*.
* [x] No existen accesos fuera de límites ni *Memory Leaks*.
* [x] La complejidad teórica coincide con la implementada.

### Reporte Crítico
*¿Qué caso borde o problema omitió la IA y cómo lo solucionaste?*
> La IA pasó los 6 casos, y usó un truco para validar el índice en una sola comparación: `(unsigned)indice < (unsigned)elementos`. Con un índice negativo funciona, porque el `-1` se vuelve un número enorme. El problema es cuando el que viene negativo es **`elementos`**: probé `indice_en_rango(0, -3)` y regresó **1**, porque `-3` también se vuelve enorme y entonces cualquier índice "cabe". Un arreglo con -3 elementos no existe, así que debería regresar 0, como en `bytes_arreglo`, que sí rechaza los negativos. Lo solucioné en mi versión revisando por separado `indice < 0` e `indice >= elementos`, así no dependo de la conversión a `unsigned`. Lo demás estuvo bien: hizo la multiplicación en `long` y `int[600000000]` dio 2400000000 sin desbordarse. Complejidad: O(1) en tiempo (la tabla de tipos es fija) y O(1) en espacio.

---
**Firma digital:** [mariajeje2006]