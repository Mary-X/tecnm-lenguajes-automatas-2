# Bitácora de Requerimientos y Auditoría IA — Sesión 02
**Materia:** Lenguajes y Autómatas II (SCD-1016)  
**Tema:** Gestión en GitHub Web y Nodos Dinámicos  
**Docente Titular:** MCC. Iván Márquez Larios (`ijmarquezl`)  
**Estudiante:** [Maria Jesus Fuentes Valdez]  
**Matrícula:** [24531315]  
**Track:** C (el harness evalúa C; la versión en Rust es opcional y no se califica con este harness)  

---

## 1. Fase 1: El Artesano (Dificultad Manual)
*Reto:* Lista enlazada de tokens en Heap  
*Describe qué problema o restricción de memoria/sintaxis identificaste al codificar a mano:*
> 
Lexema de tamaño fijo. NodoToken guarda el lexema en char lexema[16], así que como máximo caben 15 caracteres más el '\0'. Copiar con strcpy desbordaría el arreglo con un token largo. Usé snprintf(nuevo->lexema, sizeof(nuevo->lexema), "%s", lexema), que corta en 16 bytes y siempre termina la cadena en '\0'.

Lista vacía. Si cabeza == NULL, el nodo nuevo pasa a ser la cabeza y por eso la función devuelve el puntero a la cabeza (si no, el llamador nunca vería el primer nodo).

Enlazar al final. Hay que recorrer hasta el último nodo (while (actual->siguiente != NULL)) y asegurar que el siguiente del nodo nuevo sea NULL, porque malloc no inicializa la memoria.

Use-after-free al liberar. En liberar_lista, si hago free(actual) antes de leer actual->siguiente, pierdo el resto de la lista y accedo a memoria liberada. Por eso guardo siguiente en una variable temporal antes de liberar.

Entradas inválidas. Si lexema es NULL o malloc falla, la función devuelve la cabeza sin modificar la lista, para no caerse.
---

## 2. Fase 2: El Científico (Ingeniería de Requerimientos C-R-E-O)

### 2.1 Matriz de Especificación Formal
| Elemento C-R-E-O | Definición Técnica |
| :--- | :--- |
| **Contexto (C)** | C99 / GCC o Rust 2021 sin bibliotecas no estándar |
| **Rol (R)** | Ingeniero de software de sistemas y compiladores |
| **Especificación (E)** | Gestión de memoria dinámica C99/Rust |
| **Output (O)** | Solo código modular y análisis de complejidad |

### 2.2 Prompt Estructurado
```text
Contexto: Trabajo en C con GCC (-std=c11 -Wall -Wextra -Werror), sin bibliotecas no estándar. Mi archivo no debe tener main(), porque el main() es de un harness externo. Ya existe api_sesion02.h con este contrato:

typedef struct NodoToken { char lexema[16]; struct NodoToken *siguiente; } NodoToken;
NodoToken *agregar_token(NodoToken *cabeza, const char *lexema);
int contar_tokens(NodoToken *cabeza);
void liberar_lista(NodoToken *cabeza);

Rol: Actúa como ingeniero de software de sistemas y compiladores.

Especificación: Implementa las tres funciones con las firmas exactas. Cada nodo se reserva con malloc y el campo siguiente del último nodo vale NULL. agregar_token agrega el token al final y devuelve siempre la cabeza (si la lista estaba vacía, la cabeza es el nodo nuevo). El lexema se copia sin desbordar los 16 bytes y siempre termina en '\0'. contar_tokens(NULL) devuelve 0. liberar_lista libera cada nodo sin usar memoria ya liberada y liberar_lista(NULL) no debe caerse. Maneja lexema NULL y fallo de malloc sin fallo de segmentación.

Output: Solo el código del archivo .c (sin main), modular y con comentarios breves, y al final un análisis de complejidad en tiempo y espacio.[Pega aquí el prompt formal que redactaste para la IA]
```

---

## 3. Código Devuelto por el Modelo
```c
// Código devuelto por el LLM
/* ia_sesion02_c.c - Fase 2: El Cientifico */
#include "api_sesion02.h"
#include <stdlib.h>
#include <string.h>

NodoToken *agregar_token(NodoToken *cabeza, const char *lexema) {
    if (lexema == NULL) return cabeza;
    NodoToken *nuevo = (NodoToken *)malloc(sizeof(NodoToken));
    if (nuevo == NULL) return cabeza;
    strncpy(nuevo->lexema, lexema, sizeof(nuevo->lexema) - 1);
    nuevo->lexema[sizeof(nuevo->lexema) - 1] = '\0';
    nuevo->siguiente = NULL;
    if (cabeza == NULL) return nuevo;
    NodoToken *p = cabeza;
    while (p->siguiente != NULL) p = p->siguiente;
    p->siguiente = nuevo;
    return cabeza;
}

int contar_tokens(NodoToken *cabeza) {
    int n = 0;
    for (NodoToken *p = cabeza; p != NULL; p = p->siguiente) n++;
    return n;
}

void liberar_lista(NodoToken *cabeza) {
    while (cabeza != NULL) {
        NodoToken *sig = cabeza->siguiente;
        free(cabeza);
        cabeza = sig;
    }
}
// Código devuelto por el LLM
```

---

## 4. Fase 3: El Auditor (Resultados del Test Harness)
*Pruebas ejecutadas:* Prevención de Use-After-Free y cadenas nulas

### Salida del harness
```text
=== AUDITORIA SESION 02 - Lista enlazada de tokens en heap ===
Caso                               esperado:              obtenido:              resultado
[ 1] agregar_token(NULL,"if")           esperado: if           obtenido: if           OK
[ 2] contar_tokens(3 tokens)            esperado: 3            obtenido: 3            OK
[ 3] orden: segundo token               esperado: dos          obtenido: dos          OK
[ 4] ultimo->siguiente == NULL          esperado: 1            obtenido: 1            OK
[ 5] contar_tokens(NULL)                esperado: 0            obtenido: 0            OK
[ 6] liberar_lista(NULL)                esperado: sin crash    obtenido: sin crash    OK

RESULTADO: 6/6 pruebas superadas
TODO EN VERDE. Documenta tu auditoria en auditorias/auditoria_sesion02.mdPega aquí la salida de la celda del harness. Debe terminar con:
RESULTADO: n/n pruebas superadas
```

[x] Caso base y límites superados sin Crash (contar_tokens(NULL) = 0 y liberar_lista(NULL) no se cae).
[ ] No existen accesos fuera de límites ni Memory Leaks. (Las implementaciones liberan cada nodo y el lexema se copia sin pasar de 16 bytes, pero el harness no libera las listas de los casos 2, 3 y 4, así que el programa de prueba sí termina con nodos sin liberar. Verifica con valgrind --leak-check=full ./test_ia o gcc -fsanitize=address y marca la casilla solo si aplica.)
[x] La complejidad teórica coincide con la implementada: agregar_token es O(n) en tiempo (recorre hasta el último nodo) y O(1) en espacio adicional; contar_tokens y liberar_lista son O(n) en tiempo y O(1) en espacio. Construir una lista de n tokens con agregar_token es O(n²).
### Reporte Crítico
*¿Qué caso borde o problema omitió la IA y cómo lo solucionaste?*
> 
El código de la IA pasa los 6 casos y su lógica es igual a la del Artesano. La única diferencia es la copia del lexema: usó strncpy más un '\0' explícito, que es correcto, y yo usé snprintf. Lo que omitió no lo mide el harness:

Use-after-free en el llamador. liberar_lista(NodoToken *cabeza) recibe el puntero por valor, así que no puede poner en NULL el puntero del llamador. Después de liberar, c queda apuntando a memoria liberada, y un contar_tokens(c) o un agregar_token(c, ...) posterior sería un uso después de liberar. Lo detecté al revisar que el harness libera y nunca vuelve a usar el puntero. Solución: asignar c = NULL después de cada liberar_lista(c), o cambiar la firma a NodoToken ** (lo que cambiaría el contrato).
Truncamiento silencioso. Un lexema de más de 15 caracteres se corta sin avisar, y dos tokens distintos que comparten los primeros 15 caracteres quedarían iguales. Solución: validar strlen(lexema) < sizeof(nuevo->lexema) y rechazar o reportar el token.
Errores sin reportar. Con lexema == NULL o con malloc fallido devuelve la cabeza sin avisar, y el token se pierde sin que el llamador se entere.
Agregar al final es O(n). Cada inserción recorre toda la lista; con muchos tokens conviene guardar también un puntero a la cola para que sea O(1).
Fugas en la prueba. Los casos 2, 3 y 4 del harness crean listas que nunca se liberan. La IA no lo advirtió, ni el contrato lo exige.
---
**Firma digital:** [mariajeje2006]
