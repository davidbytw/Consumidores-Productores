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

---

# Explicación intuitiva (paso a paso, sin tecnicismos)

Esta sección explica el **mismo** problema de arriba, pero contado en cámara lenta
y con analogías. Es la parte que más cuesta entender la primera vez.

## 1. Cómo funciona el buffer por dentro (el anillo de cajas)

El `CircularBuffer` es un **anillo de 10 cajas** numeradas del 0 al 9:

```
        caja:  0   1   2   3   4   5   6   7   8   9
```

Hay **dos marcadores** que se persiguen alrededor del anillo:

- **`lastElem` = marcador de ESCRITURA** → dónde el productor va a dejar el
  próximo elemento.
- **`posNext` = marcador de LECTURA** → de dónde el consumidor va a sacar el
  próximo elemento.

**Reglas del juego:**

- El productor deja su elemento en **su** caja (`lastElem`) y mueve **su**
  marcador una caja adelante.
- El consumidor saca el elemento de **su** caja (`posNext`) y mueve **su**
  marcador una caja adelante.
- Si los **dos marcadores están en la MISMA caja → el buffer está VACÍO** → el
  consumidor **espera**.
- Como es un anillo, después de la caja 9 se vuelve a la caja 0 (por eso el
  `% length` en el código: `(indice + 1) % 10`).

**Idea central que hay que retener:** el consumidor **NO agarra "lo último que se
agregó"**. El consumidor **siempre lee de SU marcador (`posNext`)**, sea lo que
sea que haya en esa caja. Y las cajas **nunca se borran**: cuando un elemento se
consume, su texto **queda ahí** hasta que alguien lo **sobrescriba** una vuelta
después.

## 2. El bug, en una sola frase

En `add()`, el productor hace dos cosas en el **orden equivocado**:

```java
public void add(T data) {
    ...
    this.lastElem = (this.lastElem+1) % length;  // ①  AVISA "hay uno nuevo" (mueve el marcador)
    this.elements[p] = data;                      // ②  RECIÉN ACÁ escribe el valor
}
```

Primero **avisa** (①) y después **escribe** (②). Y como no hay candado, el
consumidor puede colarse **justo en el medio**, cuando ya avisaste pero todavía
no escribiste.

## 3. Analogía del mozo y la campanita

Un mozo (productor) le deja un plato a un cliente (consumidor):

- **Forma correcta:** apoya el plato 🍽️ y **después** toca la campanita 🔔. El
  cliente escucha, mira la mesa, el plato está. Todo bien.
- **Forma del código (rota):** toca la campanita 🔔 **primero** y **después** va
  a buscar el plato 🍽️. Si el cliente es rápido y mete la mano apenas suena la
  campanita, **el plato todavía no está** → se lleva lo que hubiera de antes (un
  plato viejo) o nada.

El "avisar antes de escribir" es exactamente tocar la campanita antes de apoyar
el plato.

## 4. Caso 1 en cámara lenta — de dónde sale el "69" en vez del "79"

Recordá: el buffer tiene 10 cajas, así que el elemento **79** y el **69** caen en
la **misma caja** (`79 % 10 = 9` y `69 % 10 = 9`). Cada 10 elementos se reusa la
misma caja física.

**Situación de partida:** ya se consumió hasta el 78. Los dos marcadores quedaron
**juntos en la caja 9** → buffer VACÍO → el consumidor está esperando. La caja 9
**todavía dice "69"** (basura vieja de 10 elementos atrás, nunca borrada).

```
caja 9 contiene: "69"      ← basura vieja
R (posNext)  = 9           ← consumidor esperando acá
W (lastElem) = 9           ← productor a punto de escribir acá
R == W  →  VACÍO  →  el consumidor espera
```

**El productor ejecuta `add("79")`:**

**Paso ① — avisa (mueve el marcador de escritura):**
```
W pasa de 9  →  0
Ahora:  R = 9,  W = 0   →   R ≠ W   →   el buffer "parece que tiene algo"
```
⚠️ Pero la caja 9 **todavía dice "69"**. El 79 NO está escrito aún.

**Paso — el consumidor, que esperaba, se despierta:**
```java
while(posNext == lastElem){ }   // 9 == 0 ? NO → deja de esperar
T e = elements[9];              // lee la caja 9 → agarra "69"   ❌ (viejo)
posNext = 0;                    // mueve su marcador
```

**Paso ② — recién ahora el productor escribe:**
```
caja 9 = "79"     ← tarde: el consumidor ya pasó de la caja 9
```

**Resultado de ese único cruce:**
- El **69** se consume **por segunda vez** (duplicado).
- El **79** queda guardado en la caja 9 pero **nadie lo lee nunca**: para cuando
  `posNext` vuelva a la caja 9 (una vuelta después), ya fue **sobrescrito por el
  89**. → el **79 se pierde**.

Por eso en el log hay **tantos duplicados como pérdidas**, y por eso aparece
`Generando: 79` pero **nunca** `consumió 79`.

## 5. Aclaración importante: NO es un "desfasaje" constante

Un error común es pensar que todo se **corre una posición** de forma permanente
(leo 70 donde iba 71, 71 donde iba 72, etc.). **No es así.**

Cada lectura mala es un **evento aislado**: el consumidor pasó por **una** caja
justo en el instante malo (marcador movido, valor sin escribir) y se llevó lo
viejo de **esa** caja puntual. La caja siguiente puede leerla **bien**.

Evidencia en el log real:
```
Generando: Elemento: 78  →  consumió: Elemento: 78    ✅ bien
Generando: Elemento: 79  →  consumió: Elemento: 69    ❌ viejo (79 − 10)
Generando: Elemento: 80  →  consumió: Elemento: 80    ✅ bien
Generando: Elemento: 81  →  consumió: Elemento: 71    ❌ viejo (81 − 10)
```

Fijate: después del **69** (malo) tomó **80** (bien), no 70. El 70 ya se había
consumido bien en su propia vuelta, antes. **Buenos y malos aparecen mezclados**,
no corridos parejo. Cada anomalía es un "swap" independiente: **un valor viejo
re-leído + un valor nuevo perdido**, en esa caja, en ese instante.

## 6. Los otros casos, contados con la misma analogía

- **Caso 2 (consumidor rápido → `null`):** misma película pero **al revés**. El
  consumidor se adelanta a una caja que el productor **nunca escribió todavía**.
  En vez de encontrar "un plato viejo", la caja está **realmente vacía** → en
  Java eso es `null`. Como el marcador de lectura llega a **sobrepasar** al de
  escritura, la condición de "vacío" no se cumple hasta dar toda la vuelta al
  anillo → por eso salen **ráfagas de `null`** de largo ≈ 10 (el tamaño del
  anillo).

- **Caso 3 (bien alternados → "funciona"):** exactamente el mismo código roto.
  Simplemente los tiempos hicieron que cuando uno tocaba el buffer, el otro
  estaba dormido en su `wait(...)`, así que **nunca se cruzaron en el instante
  malo**. Salió bien **de casualidad**, no porque el código sea correcto. Lección:
  *que un programa concurrente "funcione" una o mil veces NO prueba que sea
  correcto.*

- **Caso 4 (varios consumidores → cuelgue):** ahora los consumidores se pelean
  **entre ellos**. Varios miran el mismo marcador `posNext` a la vez, leen la
  **misma caja** y recién después mueven el marcador, pisándose → el **mismo
  elemento se entrega a muchos consumidores**. Como consumen basura (duplicados y
  `null`), **terminan su cuota antes de vaciar el buffer** y se mueren. Sin
  consumidores vivos, el productor **llena las 10 cajas** y se queda **girando
  para siempre** en el busy-wait de `add()` (buffer lleno, nadie consume). El
  programa **nunca termina** (se confirmó en la ejecución: hubo que abortarlo).
  Es un **livelock**, no un deadlock de locks → el detector automático NO lo
  agarra.

## 7. La causa de TODO, en una línea

El productor **avisa que hay un elemento nuevo (mueve el marcador) ANTES de
escribirlo**, y como no hay sincronización (`synchronized` / `wait` / `notify`),
el consumidor se cuela en ese instante y se lleva lo que había en la caja: un
valor **viejo** (Caso 1), **vacío/`null`** (Caso 2), o **repetido** entre varios
consumidores (Caso 4). Cambiar los tiempos solo cambia **cuál** de estos síntomas
aparece; no arregla nada.
