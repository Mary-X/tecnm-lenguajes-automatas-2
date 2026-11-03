# Bitácora de Requerimientos y Auditoría IA — Sesión 03
**Materia:** Lenguajes y Autómatas II (SCD-1016)  
**Tema:** Google Colab, Mágicos y Flujo de Entrega  
**Docente Titular:** MCC. Iván Márquez Larios (`ijmarquezl`)  
**Estudiante:** Maria Jesus Fuentes Valdez  
**Matrícula:** 24531315  
**Track:** C (el harness evalúa C; la versión en Rust es opcional y no se califica con este harness)  

---

## 1. Fase 1: El Artesano (Dificultad Manual)
*Reto:* Manipulación de archivos y llamadas OS  
*Describe qué problema o restricción de memoria/sintaxis identificaste al codificar a mano:*
> Al implementar `escribir_texto` y `contar_lineas` sin ayuda de la IA identifiqué tres puntos delicados:
>
> 1. **Cerrar el archivo en todos los caminos.** Cada `return` posterior a un `fopen` exitoso debe pasar por `fclose`; si `fputs` falla, hay que cerrar antes de devolver `-1`, o el descriptor queda abierto (fuga de recursos).
> 2. **Distinguir `EOF` de un carácter.** `fgetc` devuelve `int`, no `char`; guardar el resultado en un `char` impide detectar bien el fin de archivo.
> 3. **Contar la última línea sin salto final.** Contar solo los `'\n'` da 1 en `"a\nb"` en lugar de 2. Lo resolví recordando el último carácter leído: si existe y no es `'\n'`, se suma una línea. Esto además mantiene el archivo vacío en 0.
>
> También validé los punteros `NULL` y reporté los errores con `-1` en lugar de abortar, como pide el contrato de `api_sesion03.h`.

---

## 2. Fase 2: El Científico (Ingeniería de Requerimientos C-R-E-O)

### 2.1 Matriz de Especificación Formal
| Elemento C-R-E-O | Definición Técnica |
| :--- | :--- |
| **Contexto (C)** | C99 / GCC o Rust 2021 sin bibliotecas no estándar |
| **Rol (R)** | Ingeniero de software de sistemas y compiladores |
| **Especificación (E)** | Salida estándar y códigos de retorno ABI |
| **Output (O)** | Solo código modular y análisis de complejidad |

### 2.2 Prompt Estructurado
```text
Contexto: C11 (compatible con C99), compilado con gcc -std=c11 -Wall -Wextra -Werror,
solo biblioteca estándar, sin main() (el main es del harness). Incluye "api_sesion03.h".

Rol: Ingeniero de software de sistemas y compiladores.

Especificación: Implementa exactamente estas firmas, sin cambiarlas:
  int escribir_texto(const char *ruta, const char *texto);
  int contar_lineas(const char *ruta);
- escribir_texto devuelve 0 si escribió el texto completo y -1 ante cualquier error
  (ruta o texto NULL, ruta que no se puede abrir, error de escritura o de cierre).
- contar_lineas devuelve el número de líneas y -1 ante cualquier error
  (ruta NULL, archivo inexistente o no legible, error de lectura).
- Una última línea sin '\n' cuenta como línea; un archivo vacío cuenta 0.
- Usa fopen con fgetc o fgets, cierra siempre el archivo en todos los caminos
  y nunca abortes: los errores se reportan con -1.

Output: Solo código modular en C y un análisis de complejidad en tiempo y espacio.
Sin explicaciones adicionales.
```

---

## 3. Código Devuelto por el Modelo
```c
/* ia_sesion03_c.c - Fase 2: El Cientifico */
#include "api_sesion03.h"
#include <stdio.h>
#include <stdlib.h>

int escribir_texto(const char *ruta, const char *texto) {
    if (ruta == NULL || texto == NULL) {
        return -1;
    }

    FILE *archivo = fopen(ruta, "w");
    if (archivo == NULL) {
        return -1;
    }

    int resultado = fputs(texto, archivo);
    if (resultado == EOF) {
        fclose(archivo);
        return -1;
    }

    fclose(archivo);
    return 0;
}

int contar_lineas(const char *ruta) {
    if (ruta == NULL) {
        return -1;
    }

    FILE *archivo = fopen(ruta, "r");
    if (archivo == NULL) {
        return -1;
    }

    int lineas = 0;
    int caracter;
    int ultimo_caracter = EOF;

    while ((caracter = fgetc(archivo)) != EOF) {
        if (caracter == '\n') {
            lineas++;
        }
        ultimo_caracter = caracter;
    }

    /* Si el archivo contenía texto pero no terminaba con salto de línea */
    if (ultimo_caracter != EOF && ultimo_caracter != '\n') {
        lineas++;
    }

    fclose(archivo);
    return lineas;
}
```

**Análisis de complejidad:**
- `escribir_texto`: tiempo O(n) con n = longitud del texto; espacio O(1) adicional.
- `contar_lineas`: tiempo O(n) con n = tamaño del archivo en bytes (una lectura secuencial); espacio O(1), pues solo usa contadores y no carga el archivo en memoria.

---

## 4. Fase 3: El Auditor (Resultados del Test Harness)
*Pruebas ejecutadas:* Detección de errores de compilación GCC

Compilación: `gcc -std=c11 -Wall -Wextra -Werror harness_sesion03_c.c ia_sesion03_c.c -o test_ia && ./test_ia`

### Salida del harness
```text
=== AUDITORIA SESION 03 - Archivos: escribir y contar lineas ===
Caso                               esperado:              obtenido:              resultado
[ 1] escribir_texto(...)==0             esperado: 0            obtenido: 0            OK
[ 2] contar_lineas(3 lineas)            esperado: 3            obtenido: 3            OK
[ 3] contar_lineas(sin salto final)     esperado: 2            obtenido: 2            OK
[ 4] contar_lineas(archivo vacio)       esperado: 0            obtenido: 0            OK
[ 5] contar_lineas(inexistente)         esperado: -1           obtenido: -1           OK
[ 6] escribir_texto(ruta invalida)      esperado: -1           obtenido: -1           OK

RESULTADO: 6/6 pruebas superadas
TODO EN VERDE. Documenta tu auditoria en auditorias/auditoria_sesion03.md
```

La versión del Artesano (`artesano_sesion03_c.c`) obtuvo el mismo resultado: **6/6 pruebas superadas**.

* [x] Caso base y límites superados sin *Crash*.
* [ ] No existen accesos fuera de límites ni *Memory Leaks*.
* [x] La complejidad teórica coincide con la implementada.

### Reporte Crítico
*¿Qué caso borde o problema omitió la IA y cómo lo solucionaste?*
> El harness pasó 6/6, pero solo cubre seis casos. Escribí una prueba adicional (`extra_sesion03.c`, con su propio `main`) para dos situaciones que el harness no revisa, y la IA las omitió:
>
> 1. **Error al cerrar el archivo en `escribir_texto`.** La escritura va a un búfer, de modo que un error de disco puede aparecer solo en `fclose`. Con `escribir_texto("/dev/full", "hola")`, el `fopen` y el `fputs` tienen éxito, pero la función ignoraba el valor de `fclose` y devolvía `0` en vez de `-1`.
> 2. **Error de lectura en `contar_lineas`.** `fgetc` devuelve `EOF` tanto al llegar al final como ante un error de lectura, y el código no consultaba `ferror`. Con `contar_lineas(".")` (un directorio, que `fopen` abre en modo lectura en Linux), la función devolvía `0` en vez de `-1`.
>
> **Salida de la prueba adicional antes de corregir:**
> ```text
> [PEGA AQUÍ LA SALIDA DE ./extra]
> ```
>
> **Solución:** en `escribir_texto`, terminar con `return fclose(archivo) == 0 ? 0 : -1;`. En `contar_lineas`, después del ciclo, revisar `if (ferror(archivo)) { fclose(archivo); return -1; }` antes de contar la última línea. Tras el cambio, volví a ejecutar el harness (sigue en 6/6) y la prueba adicional (ahora devuelve `-1` en ambos casos).

---
**Firma digital:** mariajeje2006
