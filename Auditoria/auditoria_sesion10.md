# Bitácora de Requerimientos y Auditoría IA — Sesión 10
**Materia:** Lenguajes y Autómatas II (SCD-1016)  
**Tema:** Hito 1: Mini-Evaluador de Expresiones  
**Docente Titular:** MCC. Iván Márquez Larios (`ijmarquezl`)  
**Estudiante:** Maria Jesus Fuentes Valdez  
**Matrícula:** 24531315  
**Track:** C (el harness evalúa C; la versión en Rust es opcional y no se califica con este harness)  

---

## 1. Fase 1: El Artesano (Dificultad Manual)
*Reto:* Integración AST y Tipado Estático  
*Describe qué problema o restricción de memoria/sintaxis identificaste al codificar a mano:*
> Este fue el más largo, porque había que juntar todo: precedencia, paréntesis, espacios y los dos tipos de error. Lo armé como descenso recursivo con tres funciones: `expr` maneja `+` y `-`, `termino` maneja `*` y `/`, y `primaria` lee un número o un paréntesis. Como `expr` llama a `termino`, la multiplicación queda "más adentro" y por eso `2+3*4` da 14 y no 20. Lo que más me costó fue la asociatividad: si lo hacía recursivo por la derecha, `5-10-2` salía mal, así que usé un `while` que va acumulando en `v`. Con `"2+"`, la `primaria` no encuentra ni número ni `(` y marca `error = 2`. Al final, en `evaluar_expresion`, reviso que no sobre texto después de la expresión, porque si no, `"7 8"` se aceptaba como 7. La división entre cero se revisa antes de dividir, igual que en la sesión 07.

---

## 2. Fase 2: El Científico (Ingeniería de Requerimientos C-R-E-O)

### 2.1 Matriz de Especificación Formal
| Elemento C-R-E-O | Definición Técnica |
| :--- | :--- |
| **Contexto (C)** | C99 / GCC o Rust 2021 sin bibliotecas no estándar |
| **Rol (R)** | Ingeniero de software de sistemas y compiladores |
| **Especificación (E)** | Contrato formal de Frontend Semántico |
| **Output (O)** | Solo código modular y análisis de complejidad |

### 2.2 Prompt Estructurado
```text
CONTEXTO: C11 con GCC (-Wall -Wextra -Werror), solo libc. La cabecera
api_sesion10.h declara: long evaluar_expresion(const char *fuente, int *error);
El analizador usa un struct Parser { const char *s; int error; } y tres
funciones: primaria, termino y expr.

ROL: Ingeniero de software de sistemas y compiladores.

ESPECIFICACIÓN: Mini-evaluador por descenso recursivo.
 - Acepta enteros sin signo, + - * /, paréntesis y espacios.
 - Respeta precedencia (* / antes que + -) y asociatividad a la izquierda.
 - error = 1 si hay división entre cero (revisar antes de dividir);
   error = 2 si la sintaxis es inválida o sobra texto al final.
 - Sin variables ni menos unario. Sin main() y sin cambiar la firma.

OUTPUT: Solo código C modular y un análisis breve de complejidad.
```

---

## 3. Código Devuelto por el Modelo
```c
// Código devuelto por el LLM
#include "api_sesion10.h"
#include <stdio.h>
#include <stdlib.h>
#include <ctype.h>

typedef struct {
    const char *s;
    int error;
} Parser;

static void saltar_espacios(Parser *p) {
    while (*p->s == ' ') {
        p->s++;
    }
}

long expr(Parser *p);

long primaria(Parser *p) {
    saltar_espacios(p);
    if (p->error) return 0;

    if (*p->s == '(') {
        p->s++;
        long v = expr(p);
        if (p->error) return 0;
        saltar_espacios(p);
        if (*p->s == ')') {
            p->s++;
            return v;
        } else {
            p->error = 2;
            return 0;
        }
    }

    if (isdigit((unsigned char)*p->s)) {
        long v = 0;
        while (isdigit((unsigned char)*p->s)) {
            v = v * 10 + (*p->s - '0');
            p->s++;
        }
        return v;
    }

    p->error = 2;
    return 0;
}

long termino(Parser *p) {
    long v = primaria(p);
    if (p->error) return 0;
    while (1) {
        saltar_espacios(p);
        if (*p->s == '*') {
            p->s++;
            long der = primaria(p);
            if (p->error) return 0;
            v *= der;
        } else if (*p->s == '/') {
            p->s++;
            long der = primaria(p);
            if (p->error) return 0;
            if (der == 0) {
                p->error = 1;
                return 0;
            }
            v /= der;
        } else {
            break;
        }
    }
    return v;
}

long expr(Parser *p) {
    long v = termino(p);
    if (p->error) return 0;
    while (1) {
        saltar_espacios(p);
        if (*p->s == '+') {
            p->s++;
            long der = termino(p);
            if (p->error) return 0;
            v += der;
        } else if (*p->s == '-') {
            p->s++;
            long der = termino(p);
            if (p->error) return 0;
            v -= der;
        } else {
            break;
        }
    }
    return v;
}

long evaluar_expresion(const char *fuente, int *error) {
    static int temporal_error;
    if (error == NULL) error = &temporal_error;
    *error = 0;
    if (fuente == NULL) {
        *error = 2;
        return 0;
    }

    Parser p = {fuente, 0};
    long v = expr(&p);
    saltar_espacios(&p);
    if (p.error == 0 && *p.s != '\0') {
        p.error = 2;
    }
    *error = p.error;
    return p.error != 0 ? 0 : v;
}
```

---

## 4. Fase 3: El Auditor (Resultados del Test Harness)
*Pruebas ejecutadas:* Batería de 10 expresiones mixtas complejas

### Salida del harness
```text
=== AUDITORIA SESION 10 - Mini-evaluador de expresiones (Hito 1) ===
Caso                               esperado:              obtenido:              resultado
[ 1] "2+3*4"                            esperado: 14           obtenido: 14           OK
[ 2] "(2+3)*4"                          esperado: 20           obtenido: 20           OK
[ 3] "10/0" -> error 1                  esperado: 1            obtenido: 1            OK
[ 4] "2+" -> error 2                    esperado: 2            obtenido: 2            OK
[ 5] "  7 "                             esperado: 7            obtenido: 7            OK
[ 6] "5-10"                             esperado: -5           obtenido: -5           OK

RESULTADO: 6/6 pruebas superadas
TODO EN VERDE. Documenta tu auditoria en auditorias/auditoria_sesion10.md
```

* [x] Caso base y límites superados sin *Crash*.
* [x] No existen accesos fuera de límites ni *Memory Leaks*.
* [x] La complejidad teórica coincide con la implementada.

### Reporte Crítico
*¿Qué caso borde o problema omitió la IA y cómo lo solucionaste?*
> La IA pasó los 6 casos, pero encontré dos problemas que el harness no prueba. El primero: `saltar_espacios` solo brinca el carácter `' '`, así que una expresión con tabulador como `"1\t+ 2"` da **error 2 de sintaxis**, aunque sea válida. Lo arreglaría usando `isspace((unsigned char)*p->s)`, que también cubre `\t`, `\r` y `\n`. El segundo: los números no tienen límite. Con `"99999999999999999999"` regresó `7766279631452241919` con `error = 0`, o sea un valor basura sin avisar, porque `v * 10 + d` se desborda en `long`. La solución es revisar antes de multiplicar si `v > (LONG_MAX - d) / 10` y en ese caso marcar error. Mi versión manual tiene los mismos dos detalles, así que el arreglo aplica a las dos. Complejidad: O(n) en tiempo, porque cada carácter se lee una sola vez, y O(p) en espacio por la recursión, donde p es la profundidad de paréntesis.

---
**Firma digital:** [mariajeje2006]