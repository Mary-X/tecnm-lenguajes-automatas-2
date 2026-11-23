# Bitácora de Requerimientos y Auditoría IA — Sesión 01

**Materia:** Lenguajes y Autómatas II (SCD-1016)  
**Tema:** Encuadre, Diagnóstico de Punteros y Metodología IA  
**Docente Titular:** MCC. Iván Márquez Larios (`ijmarquezl`)  
**Estudiante:** Maria Jesus Fuentes Valdez  
**Matrícula:** 24531315  
**Track:** C (el harness evalúa C; la versión en Rust es opcional y no se califica con este harness)

---

## 1. Fase 1: El Artesano (Dificultad Manual)

*Reto:* Inversión de buffer in-place (`char*`).

*¿Qué dificultades encontraste al invertir una cadena directamente en memoria y qué aspectos tuviste que considerar para evitar errores?*

> Al realizar este ejercicio, me di cuenta de que invertir una cadena requiere modificar sus caracteres sin crear otra cadena para guardar el resultado. Para hacerlo, se pueden utilizar dos punteros: uno que apunte al inicio y otro al final, intercambiando sus valores hasta llegar al centro.
>
> También tuve que considerar qué sucede cuando la cadena está vacía, porque no existe un último carácter al que podamos acceder. Si se calcula esa posición sin revisar primero la longitud, se pueden producir errores al trabajar con el tipo `size_t`.
>
> Otro aspecto importante es comprobar que el puntero no sea `NULL`, ya que intentar leer su contenido podría provocar que el programa falle. Además, cuando la cadena tiene un solo carácter, no se necesita realizar ningún intercambio.
>
> Por eso, antes de invertir la cadena, es importante revisar las condiciones iniciales y después realizar los intercambios únicamente mientras los punteros no se hayan cruzado.

---

## 2. Fase 2: El Científico (Ingeniería de Requerimientos C-R-E-O)

### 2.1 Matriz de Especificación Formal

| Elemento C-R-E-O | Definición Técnica |
| :--- | :--- |
| **Contexto (C)** | C99 / GCC o Rust 2021 sin bibliotecas no estándar |
| **Rol (R)** | Ingeniero de software de sistemas y compiladores |
| **Especificación (E)** | Manejo de punteros, tiempo O(n) y memoria adicional O(1) |
| **Output (O)** | Función modular en C y explicación de su complejidad |

### 2.2 Prompt Estructurado

```text
CONTEXTO:
Estoy trabajando en una práctica de Lenguajes y Autómatas II
en la que debo invertir una cadena en lenguaje C.
El programa se compila con GCC utilizando C11 y las opciones
-Wall -Wextra -Werror.
El archivo se integra con un harness que ya contiene main().

ROL:
Actúa como un programador especializado en C, punteros
y administración de memoria.

REQUERIMIENTOS:
Crea una función llamada void invertir(char *s).

La función debe cumplir con las siguientes condiciones:

1. Modificar la cadena original sin crear otra cadena
   ni utilizar malloc.
2. Utilizar dos punteros para recorrer la cadena desde
   ambos extremos y una variable temporal para intercambiar
   los caracteres.
3. Tener una complejidad temporal O(n) y espacial O(1).
4. No provocar errores si el argumento recibido es NULL.
5. Funcionar con una cadena vacía y con una cadena
   que contenga solamente un carácter.
6. Evitar accesos inválidos al calcular la posición
   del último carácter.
7. Producir los siguientes resultados:
   "HOLA" -> "ALOH"
   "abc"  -> "cba"
   "AB"   -> "BA"
   "a"    -> "a"
   ""     -> ""
8. Entregar únicamente el código necesario para la función
   y una explicación de su complejidad.
9. No agregar main(), porque el harness ya lo incluye.
```

---

## 3. Código Devuelto por el Modelo

```c
#include "api_sesion01.h"
#include <string.h>

void invertir(char *s) {
    /* Protege contra NULL y cadena vacía */
    if (s == NULL || *s == '\0') {
        return;
    }

    char *inicio = s;
    char *fin = s + strlen(s) - 1;

    /* Intercambia los extremos hasta que se cruzan */
    while (inicio < fin) {
        char tmp = *inicio;
        *inicio = *fin;
        *fin = tmp;

        inicio++;
        fin--;
    }
}
```

---

## 4. Fase 3: El Auditor (Resultados del Test Harness)

*Pruebas ejecutadas:* Cadenas normales, cadena vacía, un solo carácter y puntero `NULL`.

### Salida del harness

```text
=== AUDITORIA SESION 01 - Inversion de buffer in-place ===
Caso                              esperado:         obtenido:         resultado
[ 1] invertir("HOLA")             esperado: ALOH    obtenido: ALOH    OK
[ 2] invertir("abc")              esperado: cba     obtenido: cba     OK
[ 3] invertir("")                 esperado:         obtenido:         OK
[ 4] invertir("AB")               esperado: BA      obtenido: BA      OK
[ 5] invertir("a")                esperado: a       obtenido: a       OK
[ 6] invertir(NULL)               esperado: sin crash obtenido: sin crash OK

RESULTADO: 6/6 pruebas superadas
```

- [x] La función responde correctamente a las entradas de prueba, incluidos los casos especiales.
- [x] No se detectaron accesos fuera de límites ni fugas de memoria en las pruebas realizadas.
- [x] El uso de memoria y el tiempo de ejecución corresponden con la complejidad solicitada.

### Reporte Crítico

*¿Qué aspectos de la solución tuviste que revisar para determinar si era correcta y qué aprendiste de las pruebas?*

> Al analizar la solución generada por la IA, revisé principalmente la forma en que se calculaba la posición final de la cadena y las condiciones utilizadas para detener el algoritmo. La comprobación inicial permite que la función termine cuando recibe `NULL` o una cadena vacía.
>
> También observé que los punteros avanzan desde los extremos hacia el centro. En cada repetición se intercambian dos caracteres, por lo que no es necesario recorrer toda la cadena dos veces ni reservar espacio adicional para almacenar el resultado.
>
> Un detalle que debe tenerse en cuenta es que la función solo puede modificar buffers que estén almacenados en memoria modificable y que contengan una cadena terminada en `'\0'`. Si se intenta cambiar directamente un literal como `"HOLA"`, el comportamiento es indefinido. Esta condición depende de cómo se utilice la función.
>
> Las pruebas permiten comprobar el comportamiento con diferentes entradas, pero también es importante revisar el código para entender por qué funciona. En este caso, la complejidad temporal es O(n), ya que el número de operaciones aumenta proporcionalmente a la longitud de la cadena. La complejidad espacial adicional es O(1), porque únicamente se utilizan dos punteros y una variable temporal.
>
> Esta actividad me ayudó a comprender que una respuesta generada por IA no debe aceptarse sin revisarla. Es necesario establecer condiciones claras, probar entradas especiales y comprobar que el código respete las restricciones de memoria y los requisitos solicitados.

---

**Firma digital:** [Mary-X]
