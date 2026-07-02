# Productor/Consumidor — Análisis de errores de sincronización

Trabajo práctico de **Sistemas Operativos I** (UNICEN). El código de la carpeta
`src/edu/isistan/buffer` implementa un esquema productor/consumidor cuyo autor
**no sincronizó correctamente** las estructuras compartidas. Este documento
resume qué falla, por qué, y la evidencia observada al ejecutar el ejemplo con
distintas configuraciones.

> ⚠️ El código contiene errores introducidos **a propósito** con fines
> didácticos. No usar como referencia de código correcto.

---

## Componentes

| Clase | Rol |
|-------|-----|
| `Producer` | Thread que genera enteros consecutivos y los deposita con `buffer.add(...)`. Imprime en `System.err` (`"Generando..."`). |
| `Consumer` | Thread que saca elementos con `buffer.next()`. Imprime en `System.out` (`"El consumidor..."`). |
| `IBuffer` | Interfaz del buffer compartido. |
| `CircularBuffer` | Buffer circular de N posiciones (10 por defecto). **No sincronizado.** |
| `OneElementBuffer` | Buffer de un único elemento. **No sincronizado.** |
| `ProdConsMain` | `main` + variables estáticas de configuración (cantidad de threads, tiempos de espera, tamaño de buffer, detección de deadlocks). |

Cada `Producer`/`Consumer` ejecuta su bucle `for` una cantidad **fija** de veces
(`produce` / `consume`). **Ninguno le roba turnos al otro**: que uno vaya más
rápido no cambia *cuántas* operaciones hace, solo *cómo se entrelazan*.

---

## La causa raíz

`CircularBuffer` (igual que `OneElementBuffer`) **no usa `synchronized` ni
`wait()/notify()`**. Los threads leen y modifican `posNext`, `lastElem` y
`elements[]` **al mismo tiempo, sin exclusión mutua**. `volatile` garantiza
visibilidad de cada variable por separado, pero **no atomicidad** del conjunto
de operaciones. Además:

```java
public void add(T data) {
    while(((posNext+1)%length)==lastElem){ }  // espera activa (busy-wait) si está lleno
    int p = this.lastElem;
    this.lastElem = (this.lastElem+1) % length;  // ①  PUBLICA el índice primero
    this.elements[p] = data;                      // ②  ESCRIBE el dato después
}

public T next() {
    while(posNext == lastElem){ }   // espera activa si está vacío
    T e = elements[posNext];        // lee
    posNext = (posNext+1) % length; // avanza
    return e;
}
```

Dos problemas centrales:

1. **`add()` publica el índice antes de escribir el dato** (① antes que ②). En
   la ventana entre ambos, un consumidor cree que hay un elemento y lee la celda
   **antes** de que sea escrita.
2. **Espera activa (busy-wait)** en lugar de bloqueo: quema CPU y, sobre todo,
   la condición de "lleno/vacío" (`posNext == lastElem`) **no es confiable**
   cuando los índices se actualizan sin atomicidad.

Detalle adicional: `size()` usa `%` sobre una resta que **puede dar negativo**
(en Java el módulo conserva el signo del dividendo).

---

## Casos observados

Todos los logs provienen del **mismo código sin sincronizar**. Lo único que
cambia entre casos es la configuración (tiempos de espera y cantidad de
threads). Las anomalías de **orden** entre líneas (p. ej. `consumió 3` antes de
`Generando 3`) son un artefacto de mezclar `System.out` (consumidor) con
`System.err` (productor): **no** son el error. Los errores reales son de
**contenido**: `null`, duplicados, pérdidas y cuelgues.

### Caso 1 — Productor más rápido: lecturas viejas + pérdidas

Un productor, un consumidor. El productor le queda pegado al consumidor sobre la
misma celda reciclada del buffer.

**Firma:** el valor consumido es siempre el **generado menos el tamaño del
buffer** (`N − 10`):

```
Generando: Elemento: 79  →  consumió: Elemento: 69   (79 − 10)
Generando: Elemento: 81  →  consumió: Elemento: 71   (81 − 10)
Generando: Elemento: 82  →  consumió: Elemento: 72   (82 − 10)
```

**Por qué:** por ①→②, el consumidor lee la celda **antes** de que el productor
la sobrescriba, obteniendo el valor de hace 10 iteraciones. El elemento nuevo se
escribe tarde (cuando `posNext` ya pasó) → **se pierde**. Hay tantas pérdidas
como duplicados. `size() = 0` al final **no** prueba correctitud.

### Caso 2 — Consumidor más rápido: `null` + duplicados

Un productor, un consumidor, pero el consumidor corre más rápido y **sobrepasa**
al productor.

**Firma:** ráfagas de `null` de longitud ≈ tamaño del buffer, seguidas de una
relectura de los mismos elementos:

```
Generando: Elemento: 1  →  consumió: null   (× ~8)
                        →  consumió: Elemento: 0   (duplicado)
                        →  consumió: Elemento: 1..9 (relectura)
```

**Por qué:** las celdas arrancan en `null`. El consumidor lee posiciones **aún
no escritas**. Como `posNext` **sobrepasa** a `lastElem`, la condición de "vacío"
no vuelve a cumplirse hasta que `posNext` da toda la vuelta al buffer → por eso
la ráfaga de `null` mide casi una vuelta completa (≈10). Al completar la vuelta,
re-lee celdas viejas → duplicados; los que salieron como `null` **se perdieron**.

### Caso 3 — Bien alternados: "funciona"… por suerte

Un productor, un consumidor, tiempos tales que cuando uno toca el buffer el otro
está dormido en su `wait(...)`. Salida **perfecta**: 0..19 generados y
consumidos una sola vez, en orden, sin `null` ni duplicados, `size() = 0`.

**Lección clave:** es **exactamente el mismo código roto**. No se disparó la
condición de carrera porque **nunca coincidieron dentro de la ventana crítica**.

> Una race condition es **no determinística**. Que un programa concurrente
> "funcione" en una o mil corridas **no prueba que sea correcto**: solo prueba
> que el scheduler no cayó esta vez en el intervalo malo. Cambian los tiempos,
> la máquina o la carga → reaparece.

### Caso 4 — Varios consumidores: duplicación masiva + cuelgue (livelock)

Un productor, **10 consumidores** (IDs 28–37). Ahora los consumidores compiten
**también entre ellos**.

**Firma 1 — mismo elemento a muchos consumidores:**

```
consumidor 30 → Elemento: 0
consumidor 31 → Elemento: 0
consumidor 36 → Elemento: 0
consumidor 35 → Elemento: 0
consumidor 29 → Elemento: 0
consumidor 34 → Elemento: 0     ← seis consumidores, el mismo 0
```

`next()` es un **leer-modificar-escribir no atómico**: varios consumidores ven
el mismo `posNext`, leen la misma celda y recién después avanzan el índice
pisándose (lost update). Se suman los `null` de siempre.

**Firma 2 — el programa queda colgado (confirmado en la ejecución).** El
proceso **nunca terminó** y hubo que abortarlo manualmente. El log termina con
solo-producción y **sin** `"La cantidad de elementos en el buffer es 0"`:

```
Generando: Elemento: 34
...
Generando: Elemento: 43     (y corta)
```

**Por qué cuelga:** los consumidores gastaron su cuota consumiendo basura
(duplicados y `null`), así que **terminaron antes de vaciar el buffer**. Sin
consumidores vivos, el productor **llena las ~10 posiciones** y en el siguiente
`add()` gira **para siempre** en su busy-wait (buffer lleno, nadie consume).

**Detalle fino:** ese cuelgue es un **livelock**, no un deadlock de monitores.
El productor está *ejecutando* un bucle vacío, no *bloqueado* por un lock. Por eso
`deadlockDetection` (que usa `ThreadMXBean.findDeadlockedThreads()`) **no lo
detectaría**: ese detector solo ve threads bloqueados esperando locks/monitores.

---

## Tabla resumen

| Caso | Configuración | Síntoma | Mecanismo |
|------|---------------|---------|-----------|
| 1 | Productor rápido | Valores viejos (`N − tamañoBuffer`) + pérdidas | `add()` publica índice antes de escribir dato |
| 2 | Consumidor rápido | Ráfagas de `null` + duplicados | Lee celdas no escritas; `posNext` sobrepasa a `lastElem` |
| 3 | Bien alternados | "Funciona" (engañoso) | No coincidieron en la ventana crítica (suerte) |
| 4 | Varios consumidores | Mismo elemento a N consumidores + **cuelgue** | `next()` no atómico; buffer se llena sin consumidores → **livelock** |

---

## Conclusiones

- El problema **no** es de reparto de CPU ni de cuántas veces corre cada thread;
  es una **condición de carrera por falta de sincronización**.
- Cambiar los tiempos solo cambia **cómo se manifiesta** la corrupción (valores
  viejos, `null`, duplicados, cuelgue), **no la corrige**.
- `size() = 0` y una corrida sin errores visibles **no** son prueba de
  correctitud en código concurrente.
- Con múltiples consumidores el fallo escala de "corrompe datos" a "corrompe
  datos **y** cuelga el programa" (**livelock**, no detectable por el detector de
  deadlocks basado en monitores).

**Solución (pendiente):** hacer atómicas las secciones críticas de `add()` y
`next()` con `synchronized`, y reemplazar la espera activa por bloqueo real con
`wait()/notify()` (o `notifyAll()`), de modo que el comportamiento sea correcto
**independientemente** de los tiempos y de la cantidad de productores/consumidores.
