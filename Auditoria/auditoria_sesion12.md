# Bitácora de Requerimientos y Auditoría IA — Sesión 12
**Materia:** Lenguajes y Autómatas II (SCD-1016)  
**Tema:** Atributos de Símbolos y Memoria  
**Docente Titular:** MCC. Iván Márquez Larios (`ijmarquezl`)  
**Estudiante:** Maria Jesus Fuentes Valdez  
**Matrícula:** 24531315  
**Track:** C (el harness evalúa C; la versión en Rust es opcional y no se califica con este harness)  

---

## 1. Fase 1: El Artesano (Dificultad Manual)
*Reto:* Estructura de registro de símbolos  
*Describe qué problema o restricción de memoria/sintaxis identificaste al codificar a mano:*
> El problema principal fue la alineación. No basta con ir sumando anchos: un `int` tiene que empezar en un múltiplo de 4 y un `double` en un múltiplo de 8. Así que si el cursor va en 1 (después de un `char`), el `int` no puede quedar en 1, sino en 4. Para redondear hacia arriba al múltiplo usé `((cursor + (ancho - 1)) / ancho) * ancho`, que aprovecha que la división entera corta los decimales. Después guardo ese offset en el símbolo y regreso `offset + ancho` como el siguiente cursor. En `ancho_de_tipo` hice una cadena de `strcmp` y agregué que cualquier tipo que termine en `*` cuente como puntero de 8 bytes. Con un tipo desconocido o un cursor negativo regreso -1.

---

## 2. Fase 2: El Científico (Ingeniería de Requerimientos C-R-E-O)

### 2.1 Matriz de Especificación Formal
| Elemento C-R-E-O | Definición Técnica |
| :--- | :--- |
| **Contexto (C)** | C99 / GCC o Rust 2021 sin bibliotecas no estándar |
| **Rol (R)** | Ingeniero de software de sistemas y compiladores |
| **Especificación (E)** | Cálculo de desplazamientos y alineación |
| **Output (O)** | Solo código modular y análisis de complejidad |

### 2.2 Prompt Estructurado
```text
CONTEXTO: C11 con GCC (-Wall -Wextra -Werror), arquitectura x86-64, solo
libc. La cabecera api_sesion12.h define:
  typedef struct Simbolo { const char *nombre; const char *tipo; int offset; } Simbolo;
  int ancho_de_tipo(const char *tipo);
  int asignar_offset(Simbolo *simbolo, int cursor);

ROL: Ingeniero de software de sistemas y compiladores.

ESPECIFICACIÓN:
 - ancho_de_tipo: char 1, short 2, int 4, float 4, double 8, puntero 8;
   desconocido o NULL devuelve -1.
 - asignar_offset: alinea el cursor al ancho del tipo, guarda ese offset en
   el símbolo y devuelve el siguiente cursor (offset + ancho). Símbolo NULL,
   cursor negativo o tipo desconocido devuelven -1.
 - Sin main() y sin cambiar las firmas.

OUTPUT: Solo código C modular y un análisis breve de complejidad.
```

---

## 3. Código Devuelto por el Modelo
```c
// Código devuelto por el LLM
#include "api_sesion12.h"
#include <string.h>
#include <stddef.h>

int ancho_de_tipo(const char *tipo) {
    if (tipo == NULL) {
        return -1;
    }

    if (strcmp(tipo, "char") == 0) {
        return 1;
    }
    if (strcmp(tipo, "short") == 0) {
        return 2;
    }
    if (strcmp(tipo, "int") == 0) {
        return 4;
    }
    if (strcmp(tipo, "float") == 0) {
        return 4;
    }
    if (strcmp(tipo, "double") == 0) {
        return 8;
    }

    // Punteros: "puntero" o cualquier tipo que termine con el caracter de desreferencia '*'
    size_t len = strlen(tipo);
    if (strcmp(tipo, "puntero") == 0 || (len > 0 && tipo[len - 1] == '*')) {
        return 8;
    }

    return -1;
}

int asignar_offset(Simbolo *simbolo, int cursor) {
    if (simbolo == NULL || cursor < 0) {
        return -1;
    }

    int ancho = ancho_de_tipo(simbolo->tipo);
    if (ancho == -1) {
        return -1;
    }

    // Alineación natural: Redondear el cursor al múltiplo más cercano del tamaño de alineación (ancho)
    int offset = ((cursor + (ancho - 1)) / ancho) * ancho;

    simbolo->offset = offset;

    // Retornar el siguiente cursor disponible después del espacio del tipo de dato
    return offset + ancho;
}
```

---

## 4. Fase 3: El Auditor (Resultados del Test Harness)
*Pruebas ejecutadas:* Alineación de tipos y tamaños en bytes

### Salida del harness
```text
=== AUDITORIA SESION 12 - Registro de simbolo y offsets alineados ===
Caso                               esperado:              obtenido:              resultado
[ 1] ancho_de_tipo("int")               esperado: 4            obtenido: 4            OK
[ 2] ancho_de_tipo("double")            esperado: 8            obtenido: 8            OK
[ 3] int en cursor 0                    esperado: 4            obtenido: 4            OK
[ 4] char e int: offset de b            esperado: 4            obtenido: 4            OK
[ 5] double desde cursor 4              esperado: 8            obtenido: 8            OK
[ 6] registro NULL                      esperado: -1           obtenido: -1           OK

RESULTADO: 6/6 pruebas superadas
TODO EN VERDE. Documenta tu auditoria en auditorias/auditoria_sesion12.md
```

* [x] Caso base y límites superados sin *Crash*.
* [x] No existen accesos fuera de límites ni *Memory Leaks*.
* [x] La complejidad teórica coincide con la implementada.

### Reporte Crítico
*¿Qué caso borde o problema omitió la IA y cómo lo solucionaste?*
> La IA pasó los 6 casos, pero la fórmula de alineación se puede **desbordar**. `cursor + (ancho - 1)` se calcula en `int`, así que con un cursor cerca del máximo truena. Lo probé con un `int` y `cursor = INT_MAX - 2`: el offset salió `-2147483648` y el siguiente cursor `-2147483644`, o sea negativos, que es justo lo que la función debería rechazar. Es un caso raro (un marco de 2 GB), pero un compilador no debería regresar offsets negativos sin avisar. Lo solucionaría revisando antes si `cursor > INT_MAX - ancho` y regresando -1, o haciendo la cuenta en `long`. También noté que acepta cualquier cosa que termine en `*` como puntero, incluso `"*"` solo, lo cual es más permisivo de lo que pide el contrato. Complejidad: O(1) en tiempo y espacio (el `strlen` depende del largo del nombre del tipo, que es muy corto).

---
**Firma digital:** [mariajeje2006]