# Bitácora de Requerimientos y Auditoría IA — Sesión 11
**Materia:** Lenguajes y Autómatas II (SCD-1016)  
**Tema:** Arquitectura de la Tabla de Símbolos  
**Docente Titular:** MCC. Iván Márquez Larios (`ijmarquezl`)  
**Estudiante:** Maria Jesus Fuentes Valdez  
**Matrícula:** 24531315  
**Track:** C (el harness evalúa C; la versión en Rust es opcional y no se califica con este harness)  

---

## 1. Fase 1: El Artesano (Dificultad Manual)
*Reto:* Tabla Hash con encadenamiento  
*Describe qué problema o restricción de memoria/sintaxis identificaste al codificar a mano:*
> Lo que más me costó fue la función hash. djb2 empieza en 5381 y por cada carácter hace `hash * 33 + c`, y lo escribí como `(hash << 5) + hash` para que fuera con corrimientos. Tuve que convertir cada carácter a `unsigned char`, porque si no, una letra con acento daría un número negativo. Después, para insertar, saco el índice con `hash % cubetas` y recorro la lista de esa cubeta: si encuentro el mismo nombre regreso 0 y no lo duplico, y si no, creo el nodo con `malloc` y lo pego **al final** de la cadena para conservar el orden de inserción. Como el campo `nombre` es un arreglo de 16, copié con `strncpy` y puse el `'\0'` a mano en la última posición, porque `strncpy` no lo garantiza.

---

## 2. Fase 2: El Científico (Ingeniería de Requerimientos C-R-E-O)

### 2.1 Matriz de Especificación Formal
| Elemento C-R-E-O | Definición Técnica |
| :--- | :--- |
| **Contexto (C)** | C99 / GCC o Rust 2021 sin bibliotecas no estándar |
| **Rol (R)** | Ingeniero de software de sistemas y compiladores |
| **Especificación (E)** | Función de dispersión y colisiones O(1) |
| **Output (O)** | Solo código modular y análisis de complejidad |

### 2.2 Prompt Estructurado
```text
CONTEXTO: C11 con GCC (-Wall -Wextra -Werror), solo libc. La cabecera
api_sesion11.h define:
  typedef struct Simbolo { char nombre[16]; int tipo; int offset;
                           struct Simbolo *sig; } Simbolo;
  unsigned int hash_djb2(const char *clave);
  int insertar_simbolo(Simbolo *tabla[], unsigned int cubetas,
                       const char *nombre, int tipo, int offset);

ROL: Ingeniero de software de sistemas y compiladores.

ESPECIFICACIÓN: Tabla hash con encadenamiento separado.
 - hash_djb2: hash = 5381; por cada carácter hash = hash * 33 + c.
 - insertar_simbolo: índice = hash % cubetas; si el nombre ya está en la
   cadena regresa 0; si no, crea el nodo y regresa 1; -1 con argumentos
   inválidos (tabla NULL, nombre NULL, cubetas 0) o si falla malloc.
 - Copia el nombre de forma segura (máximo 15 caracteres más '\0').
 - Sin main() y sin cambiar las firmas.

OUTPUT: Solo código C modular y un análisis breve de complejidad.
```

---

## 3. Código Devuelto por el Modelo
```c
// Código devuelto por el LLM
#include "api_sesion11.h"
#include <string.h>
#include <stdlib.h>

unsigned int hash_djb2(const char *clave) {
    if (!clave) return 0;
    unsigned int hash = 5381;
    int c;
    while ((c = (unsigned char)*clave++)) {
        hash = ((hash << 5) + hash) + c; /* hash * 33 + c */
    }
    return hash;
}

int insertar_simbolo(Simbolo *tabla[], unsigned int cubetas, const char *nombre, int tipo, int offset) {
    if (tabla == NULL || nombre == NULL || cubetas == 0) {
        return -1;
    }

    unsigned int hash_val = hash_djb2(nombre);
    unsigned int indice = hash_val % cubetas;

    /* Buscar si el simbolo ya existe en la cubeta */
    Simbolo *actual = tabla[indice];
    Simbolo *anterior = NULL;
    while (actual != NULL) {
        if (strcmp(actual->nombre, nombre) == 0) {
            return 0; /* Ya existe, no se duplica */
        }
        anterior = actual;
        actual = actual->sig;
    }

    /* Crear nuevo nodo */
    Simbolo *nuevo = (Simbolo *)malloc(sizeof(Simbolo));
    if (nuevo == NULL) {
        return -1; /* Error de asignacion */
    }

    strncpy(nuevo->nombre, nombre, sizeof(nuevo->nombre) - 1);
    nuevo->nombre[sizeof(nuevo->nombre) - 1] = '\0';
    nuevo->tipo = tipo;
    nuevo->offset = offset;
    nuevo->sig = NULL;

    /* Insertar al final de la cadena de colision para conservar el orden */
    if (anterior == NULL) {
        tabla[indice] = nuevo;
    } else {
        anterior->sig = nuevo;
    }

    return 1; /* Exito */
}
```

---

## 4. Fase 3: El Auditor (Resultados del Test Harness)
*Pruebas ejecutadas:* Manejo de claves duplicadas y colisiones

### Salida del harness
```text
=== AUDITORIA SESION 11 - Tabla hash con encadenamiento ===
Caso                               esperado:              obtenido:              resultado
[ 1] hash_djb2("")                      esperado: 5381         obtenido: 5381         OK
[ 2] hash_djb2("abc")                   esperado: 193485963    obtenido: 193485963    OK
[ 3] insertar x dos veces               esperado: 0            obtenido: 0            OK
[ 4] dos claves, una cubeta             esperado: 2            obtenido: 2            OK
[ 5] offset de beta en cadena           esperado: 4            obtenido: 4            OK
[ 6] insertar en tabla NULL             esperado: -1           obtenido: -1           OK

RESULTADO: 6/6 pruebas superadas
TODO EN VERDE. Documenta tu auditoria en auditorias/auditoria_sesion11.md
```

* [x] Caso base y límites superados sin *Crash*.
* [x] No existen accesos fuera de límites ni *Memory Leaks*.
* [x] La complejidad teórica coincide con la implementada.

### Reporte Crítico
*¿Qué caso borde o problema omitió la IA y cómo lo solucionaste?*
> La IA pasó los 6 casos, pero cuando el nombre ya existe regresa 0 **y descarta en silencio el tipo y el offset nuevos**. Lo probé insertando `x` con tipo 1 y offset 0, y después otra vez `x` con tipo 3 y offset 8: la segunda llamada regresó 0 y el símbolo se quedó con `tipo=1 offset=0`. Para la tabla está bien no duplicar, pero en un compilador esa segunda inserción es una **redeclaración**, posiblemente con otro tipo, y quien llama no tiene forma de saber si el tipo coincidía o no. Lo resolvería regresando un código distinto cuando el tipo no coincide (por ejemplo -2), o devolviendo un puntero al símbolo existente para que el analizador compare. Además, no hay ninguna función para liberar los nodos que se crean con `malloc`, así que al terminar quedan como fugas de memoria; hace falta un `liberar_tabla(tabla, cubetas)`. Complejidad: O(1) promedio por inserción (O(n) en el peor caso, si todo cae en la misma cubeta) y O(n) de memoria.

---
**Firma digital:** [mariajeje2006]