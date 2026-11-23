# Bitácora de Requerimientos y Auditoría IA — Sesión 19
**Materia:** Lenguajes y Autómatas II (SCD-1016)  
**Tema:** Estructuras (struct) y Miembros  
**Docente Titular:** MCC. Iván Márquez Larios (`ijmarquezl`)  
**Estudiante:** Maria Jesus Fuentes Valdez  
**Matrícula:** 24531315  
**Track:** C (el harness evalúa C; la versión en Rust es opcional y no se califica con este harness)  

---

## 1. Fase 1: El Artesano (Dificultad Manual)
*Reto:* Offsets de miembros en estructuras  
*Describe qué problema o restricción de memoria/sintaxis identificaste al codificar a mano:*
> Aquí lo difícil fue el **padding**, que pasa en dos lugares distintos. Primero, antes de cada campo hay que mover la posición al siguiente múltiplo de su ancho: en `{char a; int b;}` el `char` queda en 0, pero el `int` no puede ir en 1 y se va a 4. Para eso hice una función `alinear(pos, alin)` que saca el residuo y suma lo que falta. Segundo, al final el tamaño total se redondea al múltiplo del campo **más grande**, y por eso `{double a; char b;}` mide 16 y no 9. Esto me costó entenderlo hasta que pensé en un arreglo de esos structs: el siguiente `double` tiene que volver a quedar alineado a 8. Un tipo desconocido, la lista `NULL` o un campo que no existe regresan -1, y un struct sin campos mide 0.

---

## 2. Fase 2: El Científico (Ingeniería de Requerimientos C-R-E-O)

### 2.1 Matriz de Especificación Formal
| Elemento C-R-E-O | Definición Técnica |
| :--- | :--- |
| **Contexto (C)** | C99 / GCC o Rust 2021 sin bibliotecas no estándar |
| **Rol (R)** | Ingeniero de software de sistemas y compiladores |
| **Especificación (E)** | Alineación de memoria por palabras |
| **Output (O)** | Solo código modular y análisis de complejidad |

### 2.2 Prompt Estructurado
```text
CONTEXTO: C11 con GCC (-Wall -Wextra -Werror), x86-64, solo libc. La cabecera
api_sesion19.h define:
  typedef struct Campo { const char *nombre; const char *tipo; } Campo;
  int offset_campo(const Campo *campos, int total, const char *nombre);
  int tamano_struct(const Campo *campos, int total);

ROL: Ingeniero de software de sistemas y compiladores.

ESPECIFICACIÓN: Offsets de miembros con alineación natural.
 - Anchos: char 1, short 2, int 4, float 4, long 8, double 8, puntero 8;
   cada campo se alinea a su propio ancho antes de recibir offset.
 - tamano_struct redondea el total al múltiplo del campo más grande;
   cero campos miden 0.
 - Campo inexistente, lista NULL o tipo desconocido devuelven -1.
 - Sin main() y sin cambiar las firmas.

OUTPUT: Solo código C modular y un análisis breve de complejidad.
```

---

## 3. Código Devuelto por el Modelo
```c
// Código devuelto por el LLM
#include "api_sesion19.h"
#include <string.h>

static int tamano_tipo(const char *tipo) {
    static const struct { const char *n; int t; } tabla[] = {
        {"char", 1}, {"short", 2}, {"int", 4}, {"float", 4},
        {"long", 8}, {"double", 8}, {"puntero", 8}
    };
    if (tipo == NULL) return -1;
    for (size_t i = 0; i < sizeof tabla / sizeof tabla[0]; i++) {
        if (strcmp(tabla[i].n, tipo) == 0) return tabla[i].t;
    }
    return -1;
}

static int redondear(int valor, int multiplo) {
    return (valor + multiplo - 1) / multiplo * multiplo;
}

int offset_campo(const Campo *campos, int total, const char *nombre) {
    if (campos == NULL || nombre == NULL) return -1;
    int offset = 0;
    for (int i = 0; i < total; i++) {
        int t = tamano_tipo(campos[i].tipo);
        if (t < 0) return -1;
        offset = redondear(offset, t);
        if (strcmp(campos[i].nombre, nombre) == 0) return offset;
        offset += t;
    }
    return -1;
}

int tamano_struct(const Campo *campos, int total) {
    if (campos == NULL) return -1;
    int offset = 0, alin_max = 1;
    for (int i = 0; i < total; i++) {
        int t = tamano_tipo(campos[i].tipo);
        if (t < 0) return -1;
        offset = redondear(offset, t) + t;
        if (t > alin_max) alin_max = t;
    }
    return redondear(offset, alin_max);
}
```

---

## 4. Fase 3: El Auditor (Resultados del Test Harness)
*Pruebas ejecutadas:* Acceso a miembros inexistentes

### Salida del harness
```text
=== AUDITORIA SESION 19 - Offsets de miembros y tamano de struct ===
Caso                               esperado:              obtenido:              resultado
[ 1] struct char e int: b               esperado: 4            obtenido: 4            OK
[ 2] struct de tres campos: c           esperado: 8            obtenido: 8            OK
[ 3] tamano total con padding           esperado: 12           obtenido: 12           OK
[ 4] double seguido de char             esperado: 16           obtenido: 16           OK
[ 5] miembro inexistente                esperado: -1           obtenido: -1           OK
[ 6] struct sin campos                  esperado: 0            obtenido: 0            OK

RESULTADO: 6/6 pruebas superadas
TODO EN VERDE. Documenta tu auditoria en auditorias/auditoria_sesion19.md
```

* [x] Caso base y límites superados sin *Crash*. (en los casos del harness; el reporte describe un caso fuera de ellos que sí truena)
* [x] No existen accesos fuera de límites ni *Memory Leaks*.
* [x] La complejidad teórica coincide con la implementada.

### Reporte Crítico
*¿Qué caso borde o problema omitió la IA y cómo lo solucionaste?*
> La IA pasó los 6 casos, pero en `offset_campo` hace `strcmp(campos[i].nombre, nombre)` sin revisar si el campo **tiene nombre**. Lo probé con `{NULL, "char"}, {"b", "int"}` (un campo de relleno anónimo, que en C sí existe, por ejemplo en bit-fields sin nombre) y al buscar `b` el programa se cayó con *segmentation fault* (código 139) en la primera comparación. En mi versión pongo `campos[i].nombre != NULL &&` antes del `strcmp`, así el campo anónimo cuenta para el offset (porque sí ocupa espacio) pero nunca coincide con una búsqueda. También noté que la IA no revisa `total` negativo; con -1 no entra al ciclo y en `tamano_struct` regresa 0, cuando debería ser -1. Complejidad: O(n) en tiempo, con n = número de campos (más el `strcmp` del tipo contra una tabla fija), y O(1) en espacio.

---
**Firma digital:** [mariajeje2006]