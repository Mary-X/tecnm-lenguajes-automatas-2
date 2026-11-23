# Bitácora de Requerimientos y Auditoría IA — Sesión 16
**Materia:** Lenguajes y Autómatas II (SCD-1016)  
**Tema:** Firmas de Función y Parámetros  
**Docente Titular:** MCC. Iván Márquez Larios (`ijmarquezl`)  
**Estudiante:** Maria Jesus Fuentes Valdez  
**Matrícula:** 24531315  
**Track:** C (el harness evalúa C; la versión en Rust es opcional y no se califica con este harness)  

---

## 1. Fase 1: El Artesano (Dificultad Manual)
*Reto:* Registro de prototipos y aridad  
*Describe qué problema o restricción de memoria/sintaxis identificaste al codificar a mano:*
> Aquí el reto fue distinguir bien los tres resultados: 1 si la llamada es válida, 0 si el nombre existe pero la cantidad de argumentos no coincide, y -1 si la función no existe. Al principio regresaba 0 cuando no encontraba el nombre, y entonces "función no registrada" y "aridad incorrecta" se veían igual. Lo resolví recorriendo **todo** el catálogo y regresando -1 solo después del ciclo, cuando ya revisé todas las firmas. También tuve que cuidar el caso de `main` con cero parámetros: como comparo `aridad == nargs` exacto, una función de 0 solo acepta 0 argumentos. Al inicio valido catálogo `NULL`, nombre `NULL` y `nargs` negativo, y dentro del ciclo me salto las entradas que tengan `nombre` en `NULL` para no hacer `strcmp` con `NULL`.

---

## 2. Fase 2: El Científico (Ingeniería de Requerimientos C-R-E-O)

### 2.1 Matriz de Especificación Formal
| Elemento C-R-E-O | Definición Técnica |
| :--- | :--- |
| **Contexto (C)** | C99 / GCC o Rust 2021 sin bibliotecas no estándar |
| **Rol (R)** | Ingeniero de software de sistemas y compiladores |
| **Especificación (E)** | Coincidencia estricta de parámetros |
| **Output (O)** | Solo código modular y análisis de complejidad |

### 2.2 Prompt Estructurado
```text
CONTEXTO: C11 con GCC (-Wall -Wextra -Werror), solo libc. La cabecera
api_sesion16.h define:
  typedef struct Funcion { const char *nombre; int aridad; } Funcion;
  int verificar_llamada(const Funcion *catalogo, int total,
                        const char *nombre, int nargs);

ROL: Ingeniero de software de sistemas y compiladores.

ESPECIFICACIÓN: Verificador de prototipos con coincidencia estricta.
 - Busca el nombre exacto en todo el catálogo.
 - Devuelve 1 si la aridad coincide, 0 si no coincide y -1 si la función no
   existe.
 - Catálogo NULL, nombre NULL o nargs negativo devuelven -1.
 - Una función de cero parámetros solo acepta cero argumentos.
 - Sin main() y sin cambiar la firma.

OUTPUT: Solo código C modular y un análisis breve de complejidad.
```

---

## 3. Código Devuelto por el Modelo
```c
// Código devuelto por el LLM
#include "api_sesion16.h"
#include <string.h>
#include <stddef.h>

int verificar_llamada(const Funcion *catalogo, int total, const char *nombre, int nargs) {
    // Validaciones defensivas y de robustez
    if (catalogo == NULL || nombre == NULL || total <= 0 || nargs < 0) {
        return -1;
    }

    // Busqueda lineal en el catalogo de simbolos
    for (int i = 0; i < total; i++) {
        if (catalogo[i].nombre != NULL && strcmp(catalogo[i].nombre, nombre) == 0) {
            // Validar coincidencia de aridad exacta
            return (catalogo[i].aridad == nargs) ? 1 : 0;
        }
    }

    // No registrado en la tabla de simbolos
    return -1;
}
```

---

## 4. Fase 3: El Auditor (Resultados del Test Harness)
*Pruebas ejecutadas:* Llamadas con número incorrecto de args

### Salida del harness
```text
=== AUDITORIA SESION 16 - Registro de prototipos y aridad ===
Caso                               esperado:              obtenido:              resultado
[ 1] suma con 3 argumentos              esperado: 1            obtenido: 1            OK
[ 2] suma con 2 argumentos              esperado: 0            obtenido: 0            OK
[ 3] suma con 4 argumentos              esperado: 0            obtenido: 0            OK
[ 4] funcion no registrada              esperado: -1           obtenido: -1           OK
[ 5] main sin argumentos                esperado: 1            obtenido: 1            OK
[ 6] nombre NULL                        esperado: -1           obtenido: -1           OK

RESULTADO: 6/6 pruebas superadas
TODO EN VERDE. Documenta tu auditoria en auditorias/auditoria_sesion16.md
```

* [x] Caso base y límites superados sin *Crash*.
* [x] No existen accesos fuera de límites ni *Memory Leaks*.
* [x] La complejidad teórica coincide con la implementada.

### Reporte Crítico
*¿Qué caso borde o problema omitió la IA y cómo lo solucionaste?*
> La IA pasó los 6 casos, pero se queda con **la primera coincidencia** y no detecta catálogos con el mismo nombre repetido. Lo probé con el catálogo `{"f", 1}, {"f", 2}` y la llamada `f` con 2 argumentos: regresó 0 (aridad incorrecta), aunque la segunda entrada sí acepta 2. Lenguajito no tiene sobrecarga, así que lo correcto sería que ese catálogo nunca existiera, pero ni la función ni el registro lo impiden, y el error que se reporta confunde al programador. Lo solucionaría de dos formas: al registrar una función, rechazar un nombre que ya está (error de redefinición); o, en `verificar_llamada`, recorrer todo y regresar 1 si **alguna** firma coincide. Otro detalle es que `nargs` negativo regresa -1, el mismo código que "no existe", aunque la función sí esté registrada. Complejidad: O(n) en tiempo, con n = total, y O(1) en espacio.

---
**Firma digital:** [mariajeje2006]