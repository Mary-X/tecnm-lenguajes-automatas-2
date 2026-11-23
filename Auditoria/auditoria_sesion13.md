# Bitácora de Requerimientos y Auditoría IA — Sesión 13
**Materia:** Lenguajes y Autómatas II (SCD-1016)  
**Tema:** Manejo de Ámbitos (Scope Global/Local)  
**Docente Titular:** MCC. Iván Márquez Larios (`ijmarquezl`)  
**Estudiante:** Maria Jesus Fuentes Valdez  
**Matrícula:** 24531315  
**Track:** C (el harness evalúa C; la versión en Rust es opcional y no se califica con este harness)  

---

## 1. Fase 1: El Artesano (Dificultad Manual)
*Reto:* Pila de tablas de símbolos (Stack Scopes)  
*Describe qué problema o restricción de memoria/sintaxis identificaste al codificar a mano:*
> Lo que me costó fue separar bien las dos búsquedas. Para `declarar` solo se busca en el ámbito actual, porque es válido declarar una `x` adentro de un bloque aunque ya exista otra `x` afuera (eso es el shadowing). En cambio `buscar` tiene que subir por la cadena de `padre` hasta llegar al global, y el primero que encuentre gana. Si lo hacía al revés, o sea buscando primero en el global, el shadowing no funcionaba. También me fijé en el orden de las validaciones en `declarar`: primero reviso si ya existe (regreso 0) y después si hay espacio (regreso -1 si está lleno), para que un nombre repetido no se confunda con la tabla llena. El nombre vacío lo rechazo con `strlen(nombre) == 0`.

---

## 2. Fase 2: El Científico (Ingeniería de Requerimientos C-R-E-O)

### 2.1 Matriz de Especificación Formal
| Elemento C-R-E-O | Definición Técnica |
| :--- | :--- |
| **Contexto (C)** | C99 / GCC o Rust 2021 sin bibliotecas no estándar |
| **Rol (R)** | Ingeniero de software de sistemas y compiladores |
| **Especificación (E)** | Ocultamiento de variables (Shadowing) |
| **Output (O)** | Solo código modular y análisis de complejidad |

### 2.2 Prompt Estructurado
```text
CONTEXTO: C11 con GCC (-Wall -Wextra -Werror), solo libc. La cabecera
api_sesion13.h define:
  typedef struct Ambito { const char *nombre; const char *nombres[AMBITO_MAX];
                          int total; struct Ambito *padre; } Ambito;
  int declarar(Ambito *ambito, const char *nombre);
  const char *buscar(const Ambito *ambito, const char *nombre);

ROL: Ingeniero de software de sistemas y compiladores.

ESPECIFICACIÓN: Pila de ámbitos con ocultamiento (shadowing).
 - declarar: busca solo en el ámbito actual; 1 si insertó, 0 si ya existía,
   -1 si hay NULL, nombre vacío o el ámbito está lleno.
 - buscar: busca del ámbito actual hacia los padres y devuelve el nombre del
   ámbito donde lo encontró primero, o NULL si no existe.
 - Sin main() y sin cambiar las firmas.

OUTPUT: Solo código C modular y un análisis breve de complejidad.
```

---

## 3. Código Devuelto por el Modelo
```c
// Código devuelto por el LLM
#include "api_sesion13.h"
#include <string.h>
#include <stddef.h>

int declarar(Ambito *ambito, const char *nombre) {
    if (ambito == NULL || nombre == NULL || strlen(nombre) == 0) {
        return -1;
    }

    /* Verificar duplicados en el ambito actual únicamente */
    for (int i = 0; i < ambito->total; i++) {
        if (strcmp(ambito->nombres[i], nombre) == 0) {
            return 0;
        }
    }

    /* Verificar si hay espacio suficiente */
    if (ambito->total >= AMBITO_MAX) {
        return -1;
    }

    /* Registrar el simbolo en el ambito actual */
    ambito->nombres[ambito->total] = nombre;
    ambito->total++;
    return 1;
}

const char *buscar(const Ambito *ambito, const char *nombre) {
    if (ambito == NULL || nombre == NULL || strlen(nombre) == 0) {
        return NULL;
    }

    const Ambito *actual = ambito;
    while (actual != NULL) {
        for (int i = 0; i < actual->total; i++) {
            if (strcmp(actual->nombres[i], nombre) == 0) {
                return actual->nombre;
            }
        }
        /* Subir en la jerarquia de ambitos */
        actual = actual->padre;
    }

    return NULL;
}
```

---

## 4. Fase 3: El Auditor (Resultados del Test Harness)
*Pruebas ejecutadas:* Búsqueda jerárquica de identificadores

### Salida del harness
```text
=== AUDITORIA SESION 13 - Pila de ambitos y busqueda jerarquica ===
Caso                               esperado:              obtenido:              resultado
[ 1] declarar(&g, "x")                  esperado: 1            obtenido: 1            OK
[ 2] buscar en el mismo ambito          esperado: global       obtenido: global       OK
[ 3] buscar subiendo al padre           esperado: global       obtenido: global       OK
[ 4] shadowing en el hijo               esperado: bloque       obtenido: bloque       OK
[ 5] tres niveles hasta la raiz         esperado: raiz         obtenido: raiz         OK
[ 6] nombre no declarado                esperado: (nulo)       obtenido: (nulo)       OK

RESULTADO: 6/6 pruebas superadas
TODO EN VERDE. Documenta tu auditoria en auditorias/auditoria_sesion13.md
```

* [x] Caso base y límites superados sin *Crash*.
* [x] No existen accesos fuera de límites ni *Memory Leaks*.
* [x] La complejidad teórica coincide con la implementada.

### Reporte Crítico
*¿Qué caso borde o problema omitió la IA y cómo lo solucionaste?*
> La IA pasó los 6 casos, pero `declarar` **guarda el puntero del nombre, no una copia**. Si quien la llama reutiliza el mismo buffer, todo se descompone. Lo probé con un `char buf[8]`: copié `"x"` y declaré (regresó 1), luego copié `"y"` en el **mismo** buffer y declaré otra vez. Regresó 0, como si `y` ya existiera, y después `buscar(&g, "x")` dio `NULL`, porque la entrada guardada ahora apunta a un texto que dice `"y"`. En un compilador real esto pasa fácil, porque los lexemas suelen salir de un buffer temporal. Lo solucionaría copiando el nombre con `malloc` + `strcpy` (o un `strdup` propio) al declarar, y liberándolo al cerrar el ámbito; la otra opción es dejar escrito en el contrato que el nombre debe vivir mientras viva el ámbito. Complejidad: `declarar` es O(k) con k ≤ AMBITO_MAX, y `buscar` es O(k · p), donde p es la profundidad de la pila de ámbitos.

---
**Firma digital:** [mariajeje2006]