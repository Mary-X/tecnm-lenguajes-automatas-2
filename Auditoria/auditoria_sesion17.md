# Bitácora de Requerimientos y Auditoría IA — Sesión 17
**Materia:** Lenguajes y Autómatas II (SCD-1016)  
**Tema:** Comprobación Semántica de Retornos  
**Docente Titular:** MCC. Iván Márquez Larios (`ijmarquezl`)  
**Estudiante:** Maria Jesus Fuentes Valdez  
**Matrícula:** 24531315  
**Track:** C (el harness evalúa C; la versión en Rust es opcional y no se califica con este harness)  

---

## 1. Fase 1: El Artesano (Dificultad Manual)
*Reto:* Validación de tipo de sentencia return  
*Describe qué problema o restricción de memoria/sintaxis identificaste al codificar a mano:*
> Lo difícil fue decidir **en qué orden** revisar las cosas, porque puede haber dos errores a la vez. Decidí revisar primero el tipo y después las rutas: si una función `void` regresa un valor, eso es error de tipo (1) aunque también falten rutas. Para comparar tipos me hice una tabla de "anchos" (char 1, int 2, long 3, float 4, double 5) y la regla es que el destino tiene que ser igual o más ancho que el origen; por eso `double` acepta `int` pero `int` no acepta `double`. Luego, si alguna ruta no termina en `return` (`rutas_con_return < rutas_totales`), regreso 2. Al inicio rechazo las firmas `NULL` y los conteos incoherentes: cero rutas, o más returns que rutas, regresan -1.

---

## 2. Fase 2: El Científico (Ingeniería de Requerimientos C-R-E-O)

### 2.1 Matriz de Especificación Formal
| Elemento C-R-E-O | Definición Técnica |
| :--- | :--- |
| **Contexto (C)** | C99 / GCC o Rust 2021 sin bibliotecas no estándar |
| **Rol (R)** | Ingeniero de software de sistemas y compiladores |
| **Especificación (E)** | Rutas de ejecución con retorno garantizado |
| **Output (O)** | Solo código modular y análisis de complejidad |

### 2.2 Prompt Estructurado
```text
CONTEXTO: C11 con GCC (-Wall -Wextra -Werror), solo libc. La cabecera
api_sesion17.h declara:
  int diagnosticar_retorno(const char *tipo_funcion, const char *tipo_expresion,
                           int rutas_totales, int rutas_con_return);

ROL: Ingeniero de software de sistemas y compiladores.

ESPECIFICACIÓN: Validación semántica de return.
 - 0 si todo está bien, 1 si el tipo es incompatible, 2 si hay una ruta sin
   return.
 - La conversión solo se acepta si el destino es igual o más ancho que el
   origen (char < int < long < float < double).
 - Devolver un valor en una función void siempre es error de tipo.
 - Firma NULL, cero rutas o más returns que rutas devuelven -1.
 - Sin main() y sin cambiar la firma.

OUTPUT: Solo código C modular y un análisis breve de complejidad.
```

---

## 3. Código Devuelto por el Modelo
```c
// Código devuelto por el LLM
#include "api_sesion17.h"
#include <string.h>

typedef struct { const char *nombre; int ancho; } TipoInfo;

static const TipoInfo TIPOS[] = {
    {"char", 1}, {"int", 2}, {"long", 3}, {"float", 4}, {"double", 5}
};

static int ancho_de(const char *t) {
    for (size_t i = 0; i < sizeof TIPOS / sizeof TIPOS[0]; i++) {
        if (strcmp(TIPOS[i].nombre, t) == 0)
            return TIPOS[i].ancho;
    }
    return -1;
}

int diagnosticar_retorno(const char *tipo_funcion, const char *tipo_expresion,
                         int rutas_totales, int rutas_con_return) {
    if (!tipo_funcion || !tipo_expresion)
        return -1;
    if (rutas_totales <= 0 || rutas_con_return < 0 || rutas_con_return > rutas_totales)
        return -1;

    int f_void = strcmp(tipo_funcion, "void") == 0;
    int e_void = strcmp(tipo_expresion, "void") == 0;

    if (f_void)
        return e_void ? 0 : 1;
    if (e_void)
        return 1;

    int destino = ancho_de(tipo_funcion);
    int origen = ancho_de(tipo_expresion);
    if (destino < 0 || origen < 0)
        return -1;
    if (destino < origen)
        return 1;

    return rutas_con_return < rutas_totales ? 2 : 0;
}
```

---

## 4. Fase 3: El Auditor (Resultados del Test Harness)
*Pruebas ejecutadas:* Funciones sin retorno o retornos mixtos

### Salida del harness
```text
=== AUDITORIA SESION 17 - Validacion de retornos y rutas ===
Caso                               esperado:              obtenido:              resultado
[ 1] int con int y dos rutas            esperado: 0            obtenido: 0            OK
[ 2] void devuelve valor                esperado: 1            obtenido: 1            OK
[ 3] ruta sin retorno                   esperado: 2            obtenido: 2            OK
[ 4] double acepta int                  esperado: 0            obtenido: 0            OK
[ 5] int no acepta double               esperado: 1            obtenido: 1            OK
[ 6] cero rutas de ejecucion            esperado: -1           obtenido: -1           OK

RESULTADO: 6/6 pruebas superadas
TODO EN VERDE. Documenta tu auditoria en auditorias/auditoria_sesion17.md
```

* [x] Caso base y límites superados sin *Crash*.
* [x] No existen accesos fuera de límites ni *Memory Leaks*.
* [x] La complejidad teórica coincide con la implementada.

### Reporte Crítico
*¿Qué caso borde o problema omitió la IA y cómo lo solucionaste?*
> La IA pasó los 6 casos, pero su tabla de tipos solo conoce `char, int, long, float, double`, y cualquier otro tipo regresa **-1, el mismo código que "firma incoherente"**. Lo probé con una función `int` que devuelve `short`: dio -1, cuando debería ser 0, porque `short` cabe en `int`. El analizador lo reportaría como si la llamada estuviera mal armada y no como un tipo válido. También revisé `float` que devuelve `long`: dio 0 (sin problema), pero un `long` de 64 bits no cabe exacto en un `float`, así que el orden `long < float` de la tabla no es del todo cierto en precisión. Lo solucionaría agregando `short` (y los demás tipos del lenguaje) a la tabla, y separando el código de "tipo desconocido" del de "conteo incoherente". Mi versión manual tiene la misma tabla, así que también hay que cambiarla. Complejidad: O(1) en tiempo y espacio, porque la tabla tiene tamaño fijo.

---
**Firma digital:** [mariajeje2006]