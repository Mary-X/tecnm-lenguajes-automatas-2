# Bitácora de Requerimientos y Auditoría IA — Sesión 05
**Materia:** Lenguajes y Autómatas II (SCD-1016)  
**Tema:** Recorridos del AST y Notación Polaca  
**Docente Titular:** MCC. Iván Márquez Larios (`ijmarquezl`)  
**Estudiante:** Maria Jesus Fuentes Valdez  
**Matrícula:** 24531315  
**Track:** C (el harness evalúa C; la versión en Rust es opcional y no se califica con este harness)  

---

## 1. Fase 1: El Artesano (Dificultad Manual)
*Reto:* Impresión en Postorden / RPN  
*Describe qué problema o restricción de memoria/sintaxis identificaste al codificar a mano:*
> La principal dificultad fue escribir en el buffer sin imprimir con `printf`, respetando la capacidad `cap`. Al ir concatenando los tokens (números y operadores) tenía que llevar la cuenta de cuántos caracteres llevaba escritos para no pasarme de `cap-1` y dejar siempre el `'\0'` al final. Otra restricción fue el separador: los tokens deben ir separados por un solo espacio y sin espacio al final, así que había que decidir cuándo agregar el espacio (solo si ya hay algo escrito). También tuve que manejar el árbol nulo, que debe producir cadena vacía, y respetar el orden izquierdo → derecho → raíz en la recursión. Por último, como el compilador usa `-Wall -Wextra -Werror`, cualquier variable sin usar o conversión descuidada cuenta como error.

---

## 2. Fase 2: El Científico (Ingeniería de Requerimientos C-R-E-O)

### 2.1 Matriz de Especificación Formal
| Elemento C-R-E-O | Definición Técnica |
| :--- | :--- |
| **Contexto (C)** | C99 / GCC o Rust 2021 sin bibliotecas no estándar |
| **Rol (R)** | Ingeniero de software de sistemas y compiladores |
| **Especificación (E)** | Invariantes de recorrido recursivo |
| **Output (O)** | Solo código modular y análisis de complejidad |

### 2.2 Prompt Estructurado
```text
CONTEXTO: Estoy trabajando en C (estándar C11, compilado con GCC y las banderas
-Wall -Wextra -Werror), sin bibliotecas no estándar. Tengo un AST de expresiones
aritméticas definido en api_sesion05.h:
  typedef enum { NODO_NUM, NODO_BINOP } TipoNodo;
  typedef struct ASTNode { TipoNodo tipo; int valor; char op;
                           struct ASTNode *izq; struct ASTNode *der; } ASTNode;

ROL: Actúa como ingeniero de software de sistemas y compiladores.

ESPECIFICACIÓN: Implementa exactamente la firma
  void postorden(ASTNode *n, char *buf, int cap);
que recorre el árbol en postorden (izquierdo, derecho, raíz) y escribe en buf la
notación polaca inversa (RPN). Invariantes:
 - No uses printf: escribe únicamente en el buffer.
 - Nunca escribas más de cap-1 caracteres más el terminador '\0'.
 - Los tokens van separados por un solo espacio y sin espacio final.
 - Un árbol NULL produce cadena vacía; el buffer siempre termina en '\0'.
 - Si buf es NULL o cap <= 0, no hagas nada.
 - No incluyas main(); no cambies la firma.

OUTPUT: Entrega solo el código en C, modular (puedes usar una función auxiliar
static), más un breve análisis de complejidad temporal y espacial.
```

---

## 3. Código Devuelto por el Modelo
```c
// Código devuelto por el LLM
#include "api_sesion05.h"
#include <stdio.h>
#include <string.h>

static void postorden_helper(ASTNode *n, char *buf, int cap, int *escritos) {
    if (n == NULL) return;

    // Recorrido postorden: Izquierda, Derecha, Raíz
    postorden_helper(n->izq, buf, cap, escritos);
    postorden_helper(n->der, buf, cap, escritos);

    // Agregar espacio antes del siguiente token si ya hay algo en el buffer
    if (*escritos > 0 && *escritos < cap - 1) {
        buf[*escritos] = ' ';
        buf[*escritos + 1] = '\0';
        (*escritos)++;
    }

    // Imprimir el valor del nodo en el espacio restante de forma segura
    int limite = cap - *escritos;
    if (limite > 0) {
        int n_char = 0;
        if (n->tipo == NODO_NUM) {
            n_char = snprintf(buf + *escritos, limite, "%d", n->valor);
        } else if (n->tipo == NODO_BINOP) {
            n_char = snprintf(buf + *escritos, limite, "%c", n->op);
        }
        if (n_char > 0) {
            *escritos += n_char;
        }
    }
}

void postorden(ASTNode *n, char *buf, int cap) {
    if (buf == NULL || cap <= 0) return;
    buf[0] = '\0';
    int escritos = 0;
    postorden_helper(n, buf, cap, &escritos);
}
```

---

## 4. Fase 3: El Auditor (Resultados del Test Harness)
*Pruebas ejecutadas:* Validación contra desbordamiento de pila

### Salida del harness
```text
=== AUDITORIA SESION 05 - Recorrido postorden y RPN ===
Caso                               esperado:              obtenido:              resultado
[ 1] hoja 42                            esperado: 42           obtenido: 42           OK
[ 2] 3+(5*2)                            esperado: 3 5 2 * +    obtenido: 3 5 2 * +    OK
[ 3] arbol NULL                         esperado:              obtenido:              OK
[ 4] (1+2)*(3+4)
     esperado: 1 2 + 3 4 + *
     obtenido: 1 2 + 3 4 + *
     -> OK
[ 5] 8-3                                esperado: 8 3 -        obtenido: 8 3 -        OK

RESULTADO: 5/5 pruebas superadas
TODO EN VERDE. Documenta tu auditoria en auditorias/auditoria_sesion05.md
```

* [x] Caso base y límites superados sin *Crash*.
* [x] No existen accesos fuera de límites ni *Memory Leaks*.
* [x] La complejidad teórica coincide con la implementada.

### Reporte Crítico
*¿Qué caso borde o problema omitió la IA y cómo lo solucionaste?*
> El código de la IA pasa los 5 casos del harness, pero omitió el caso borde de **truncamiento por capacidad pequeña**. Cuando `cap` es muy chico, el código agrega el espacio separador pero luego el siguiente token ya no cabe, y el buffer queda con un **espacio final** (viola el criterio "sin espacio final"). Lo detecté probando la función con el mismo árbol `1+(2*3)` y capacidades de 1 a 10: con `cap` = 3, 5, 7 y 9 la salida fue `"1 "`, `"1 2 "`, `"1 2 3 "` y `"1 2 3 * "`. Además, `snprintf` devuelve la longitud que *habría* escrito, no la que realmente escribió, así que el contador `escritos` puede quedar mayor a lo real al truncar. No hay desbordamiento de memoria (el `'\0'` siempre queda dentro de `cap`), pero el resultado truncado no es limpio. La corrección es verificar antes de escribir que el espacio **y** el token completo caben en el buffer, y si no caben, no escribir el espacio. También el harness no prueba árboles muy profundos (la recursión usa pila proporcional a la altura del árbol). Complejidad: tiempo O(n) (cada nodo se visita una vez) y espacio auxiliar O(h) por la pila de recursión, donde h es la altura del árbol.

---
**Firma digital:** [mariajeje2006]
