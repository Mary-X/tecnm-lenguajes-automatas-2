# Bitácora de Requerimientos y Auditoría IA — Sesión 21
**Materia:** Lenguajes y Autómatas II (SCD-1016)  
**Tema:** Fundamentos de Código de 3 Direcciones  
**Docente Titular:** MCC. Iván Márquez Larios (`ijmarquezl`)  
**Estudiante:** Maria Jesus Fuentes Valdez  
**Matrícula:** 24531315  
**Track:** C (el harness evalúa C; la versión en Rust es opcional y no se califica con este harness)  

---

## 1. Fase 1: El Artesano (Dificultad Manual)
*Reto:* Representación en Cuádruplos (op,a,b,r)  
*Describe qué problema o restricción de memoria/sintaxis identificaste al codificar a mano:*
> Aquí el detalle fue el orden del contador: "se usa y después se incrementa". Si primero incrementaba y luego escribía, la primera llamada daba `_t1` en lugar de `_t0`. Así que escribo con `snprintf(dst, cap, "_t%d", *contador)` y después hago `(*contador)++`. Para el cuádruplo usé el formato `(op, a1, a2, res)` con un `snprintf` también, porque el contrato dice que hay que respetar la capacidad del buffer y no reservar memoria. Lo que más me costó fue el caso de los `NULL`: en una asignación `x = a` no hay segundo operando, y en un `goto` no hay ni segundo operando ni resultado. Para no repetir el `if` cuatro veces hice una función `o_guion()` que regresa `"_"` si el texto es `NULL`, y la uso en los cuatro campos.

---

## 2. Fase 2: El Científico (Ingeniería de Requerimientos C-R-E-O)

### 2.1 Matriz de Especificación Formal
| Elemento C-R-E-O | Definición Técnica |
| :--- | :--- |
| **Contexto (C)** | C99 / GCC o Rust 2021 sin bibliotecas no estándar |
| **Rol (R)** | Ingeniero de software de sistemas y compiladores |
| **Especificación (E)** | Generación de temporales únicos (_t0..) |
| **Output (O)** | Solo código modular y análisis de complejidad |

### 2.2 Prompt Estructurado
```text
CONTEXTO: C11 con GCC (-Wall -Wextra -Werror), solo libc. La cabecera
api_sesion21.h declara:
  void nuevo_temporal(char *dst, int cap, int *contador);
  void emitir_cuadruplo(char *dst, int cap, const char *op, const char *a1,
                        const char *a2, const char *res);

ROL: Ingeniero de software de sistemas y compiladores.

ESPECIFICACIÓN: Cuádruplos de código de tres direcciones.
 - nuevo_temporal escribe "_tK" con el valor actual del contador y después
   lo incrementa (la primera llamada da _t0).
 - emitir_cuadruplo escribe "(op, a1, a2, res)"; un argumento NULL se
   escribe como "_".
 - Escribe solo en dst sin pasarte de cap; sin malloc ni variables globales.
 - Sin main() y sin cambiar las firmas.

OUTPUT: Solo código C modular y un análisis breve de complejidad.
```

---

## 3. Código Devuelto por el Modelo
```c
// Código devuelto por el LLM
#include "api_sesion21.h"
#include <stdio.h>

void nuevo_temporal(char *dst, int cap, int *contador) {
    if (dst == NULL || contador == NULL || cap <= 0) {
        return;
    }
    snprintf(dst, (size_t)cap, "_t%d", (*contador)++);
}

void emitir_cuadruplo(char *dst, int cap, const char *op, const char *a1, const char *a2, const char *res) {
    if (dst == NULL || cap <= 0) {
        return;
    }
    snprintf(dst, (size_t)cap, "(%s, %s, %s, %s)",
             op, a1 ? a1 : "_", a2 ? a2 : "_", res ? res : "_");
}
```

---

## 4. Fase 3: El Auditor (Resultados del Test Harness)
*Pruebas ejecutadas:* Expresiones aritméticas compuestas

### Salida del harness
```text
=== AUDITORIA SESION 21 - Cuadruplos y temporales unicos ===
Caso                               esperado:              obtenido:              resultado
[ 1] nuevo_temporal(_t0)                esperado: _t0          obtenido: _t0          OK
[ 2] nuevo_temporal(_t7)                esperado: _t7          obtenido: _t7          OK
[ 3] cuadruplo de a + b
     esperado: (+, a, b, _t0)
     obtenido: (+, a, b, _t0)
     -> OK
[ 4] cuadruplo de _t0 * c
     esperado: (*, _t0, c, _t1)
     obtenido: (*, _t0, c, _t1)
     -> OK
[ 5] asignacion simple                  esperado: (=, a, _, x) obtenido: (=, a, _, x) OK
[ 6] goto sin operandos
     esperado: (goto, L1, _, _)
     obtenido: (goto, L1, _, _)
     -> OK

RESULTADO: 6/6 pruebas superadas
TODO EN VERDE. Documenta tu auditoria en auditorias/auditoria_sesion21.md
```

* [x] Caso base y límites superados sin *Crash*.
* [x] No existen accesos fuera de límites ni *Memory Leaks*.
* [x] La complejidad teórica coincide con la implementada.

### Reporte Crítico
*¿Qué caso borde o problema omitió la IA y cómo lo solucionaste?*
> La IA pasó los 6 casos, pero solo cambia por `"_"` los argumentos `a1`, `a2` y `res`; al **operador** `op` lo pasa directo a `%s`. El contrato dice que *un argumento NULL* se escribe como guion bajo, y `op` también es argumento. Lo probé con `emitir_cuadruplo(b, 64, NULL, "a", "b", "_t0")` y salió `((null), a, b, _t0)`. Eso pasa porque glibc imprime "(null)", pero en el estándar de C es comportamiento indefinido y en otra plataforma se puede caer. En mi versión uso la misma función `o_guion()` para los cuatro campos, así que el resultado es `(_, a, b, _t0)`. También noté que ninguna de las dos versiones avisa si el texto se truncó por falta de `cap`; se podría revisar lo que regresa `snprintf`. Complejidad: O(L) en tiempo, donde L es el largo de la cadena que se escribe, y O(1) en espacio extra, porque todo va directo al buffer.

---
**Firma digital:** [mariajeje2006]