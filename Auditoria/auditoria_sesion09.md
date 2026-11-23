# Bitácora de Requerimientos y Auditoría IA — Sesión 09
**Materia:** Lenguajes y Autómatas II (SCD-1016)  
**Tema:** Reporte Formal de Errores Semánticos  
**Docente Titular:** MCC. Iván Márquez Larios (`ijmarquezl`)  
**Estudiante:** Maria Jesus Fuentes Valdez  
**Matrícula:** 24531315  
**Track:** C (el harness evalúa C; la versión en Rust es opcional y no se califica con este harness)  

---

## 1. Fase 1: El Artesano (Dificultad Manual)
*Reto:* Diagnóstico con línea y columna  
*Describe qué problema o restricción de memoria/sintaxis identificaste al codificar a mano:*
> Aquí lo difícil no fue la lógica, sino armar la cadena sin pasarme del buffer. Primero pensé en usar `sprintf` o ir pegando con `strcat`, pero cualquiera de los dos puede escribir más allá de `cap` si el nombre del archivo o el mensaje son largos. Por eso usé `snprintf(salida, cap, "%s:%d:%d: error: %s", ...)`, que corta solo y siempre deja el `'\0'` al final. Eso hace que pase el caso de `cap = 10`, donde la salida tiene que quedar exactamente `main.c:14`. También revisé que `salida` no sea `NULL` y que `cap` sea mayor que 0 antes de escribir, porque con `cap = 0` no hay ni lugar para el terminador. El formato lo copié tal cual del estilo de GCC: `archivo:línea:columna: error: mensaje`.

---

## 2. Fase 2: El Científico (Ingeniería de Requerimientos C-R-E-O)

### 2.1 Matriz de Especificación Formal
| Elemento C-R-E-O | Definición Técnica |
| :--- | :--- |
| **Contexto (C)** | C99 / GCC o Rust 2021 sin bibliotecas no estándar |
| **Rol (R)** | Ingeniero de software de sistemas y compiladores |
| **Especificación (E)** | Estructura estándar de errores GCC |
| **Output (O)** | Solo código modular y análisis de complejidad |

### 2.2 Prompt Estructurado
```text
CONTEXTO: C11 con GCC (-Wall -Wextra -Werror), solo biblioteca estándar.
La cabecera api_sesion09.h declara:
  void reportar(const char *archivo, int linea, int columna,
                const char *mensaje, char *salida, int cap);

ROL: Ingeniero de software de sistemas y compiladores.

ESPECIFICACIÓN: Escribe en salida un diagnóstico con el formato estándar de GCC:
  archivo:linea:columna: error: mensaje
 - Nunca escribas más de cap bytes (incluido el '\0'); si no cabe, se trunca.
 - Si salida es NULL o cap <= 0, no hagas nada.
 - No imprimas en pantalla, no uses memoria dinámica, sin main().

OUTPUT: Solo el código C y un análisis breve de complejidad.
```

---

## 3. Código Devuelto por el Modelo
```c
// Código devuelto por el LLM
#include "api_sesion09.h"
#include <stdio.h>
#include <string.h>

void reportar(const char *archivo, int linea, int columna, const char *mensaje, char *salida, int cap) {
    if (salida != NULL && cap > 0) {
        snprintf(salida, cap, "%s:%d:%d: error: %s", archivo, linea, columna, mensaje);
    }
}
```

---

## 4. Fase 3: El Auditor (Resultados del Test Harness)
*Pruebas ejecutadas:* Múltiples errores sin abortar parser

### Salida del harness
```text
=== AUDITORIA SESION 09 - Diagnostico con linea y columna ===
Caso                               esperado:              obtenido:              resultado
[ 1] main.c:14:5
     esperado: main.c:14:5: error: tipo incompatible
     obtenido: main.c:14:5: error: tipo incompatible
     -> OK
[ 2] a.c:1:1
     esperado: a.c:1:1: error: x
     obtenido: a.c:1:1: error: x
     -> OK
[ 3] linea 0
     esperado: x.c:0:0: error: y
     obtenido: x.c:0:0: error: y
     -> OK
[ 4] cap=10 trunca                      esperado: main.c:14    obtenido: main.c:14    OK

RESULTADO: 4/4 pruebas superadas
TODO EN VERDE. Documenta tu auditoria en auditorias/auditoria_sesion09.md
```

* [x] Caso base y límites superados sin *Crash*.
* [x] No existen accesos fuera de límites ni *Memory Leaks*.
* [x] La complejidad teórica coincide con la implementada.

### Reporte Crítico
*¿Qué caso borde o problema omitió la IA y cómo lo solucionaste?*
> La IA pasó los 4 casos del harness, pero no valida que `archivo` y `mensaje` sean distintos de `NULL`. Le pasé `reportar(NULL, 3, 7, "x", b, 64)` y la salida fue `(null):3:7: error: x`. Eso funciona solo porque la `libc` de Linux (glibc) imprime "(null)" por cortesía, pero según el estándar de C pasar `NULL` a `%s` es comportamiento indefinido, y en otras plataformas el programa se puede caer. También noté que el truncado es silencioso: con `cap = 16` el mensaje quedó como `programa_largo.` y nadie se entera de que se cortó. Para corregirlo pondría `archivo ? archivo : "<desconocido>"` y lo mismo con el mensaje, y usaría el valor que regresa `snprintf` para saber si la cadena se truncó. Complejidad: O(n) en tiempo, donde n es el largo del diagnóstico, y O(1) en espacio extra, porque escribe directo en el buffer recibido.

---
**Firma digital:** [mariajeje2006]