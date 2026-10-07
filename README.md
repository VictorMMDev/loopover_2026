# Loopover 4x4

Aplicación de línea de comandos que resuelve el puzzle **Loopover** (tablero 4×4, fichas 0–15) mediante búsqueda en espacio de estado, con soporte de movimientos encadenados, seis estrategias de búsqueda y ocho funciones de heurística.

---

## 1. El puzzle

El tablero es una rejilla 4×4 con 16 fichas numeradas del 0 al 15, una por casilla. Una **acción** desplaza circularmente una fila y, acto seguido, una columna:

| Signo | Fila | Columna |
|-------|------|---------|
| `+`   | a la derecha | hacia abajo |
| `-`   | a la izquierda | hacia arriba |

Hay 4 filas × 4 columnas × 2 signos = **32 acciones posibles**. Cada acción se escribe como `fila` `columna` `signo`, por ejemplo `01+`, `33-`.

El estado resuelto tiene la ficha *i* en la casilla *i*:

```
 0  1  2  3
 4  5  6  7
 8  9 10 11
12 13 14 15
```

---

## 2. Requisitos

- JDK 17 o superior (compilado con `maven.compiler.release=17`).
- Maven 3.8 o superior.

```powershell
java -version
mvn -version
```

---

## 3. Compilación y empaquetado

```powershell
mvn -q package
```

Genera el jar ejecutable autocontenido en `target/loopover.jar` (maven-shade-plugin, clase principal `LoopoverApp`). El código fuente reside en `src/` sin paquete declarado (`sourceDirectory` = `src`).

Para iterar durante la implementación de la Tarea 1, también es posible compilar sin empaquetar y ejecutar directamente por classpath:

```powershell
mvn -q compile
mvn -q dependency:build-classpath "-Dmdep.outputFile=cp.txt"
java -cp "target/classes;$(Get-Content cp.txt)" LoopoverApp --help
Remove-Item cp.txt
```

Comprobación rápida:

```powershell
mvn -q compile
java -jar target/loopover.jar --help
```

---

## 4. Uso

La aplicación expone tres comandos: la orden raíz `loopover` (opción `-v/--verbose`; muestra el mensaje inicial con el identificador de versión `loopover 0.1.0`), y los subcomandos `verify` y `solve`. `--help` y `--version` están habilitados en todos ellos.

Tras implementar la Tarea 1, **solo `verify` estará funcional**; `solve` requerirá haber completado la Tarea 2.

### 4.1 `verify` — comprobar estados y aplicar acciones

```powershell
java -jar target/loopover.jar verify -s <estado> [-a <acciones>]
```

| Opción | Descripción |
|--------|-------------|
| `-s`   | Estado como cadena de **32 dígitos** (obligatoria). |
| `-a`   | Lista de acciones separadas por comas, p. ej. `21+,03-`. |

Sin `-a` imprime los 32 sucesores del estado. Con `-a` aplica la secuencia en orden y muestra el estado resultante.

Tras completar `Estado.java`, el resultado de mostrar sucesores tendrá el formato `(acción,estado,costo)`, por ejemplo `(00+,00010203040506070809101112131415,1.0)`. La secuencia resultante con `-a` es la cadena de 32 dígitos del estado final.

```powershell
# Los 32 sucesores del estado resuelto
java -jar target/loopover.jar verify -s 00010203040506070809101112131415

# Aplicar dos acciones y ver el resultado
java -jar target/loopover.jar verify -s 00010203040506070809101112131415 -a 00+,01+
```

### 4.2 `solve` — resolver un estado

```powershell
java -jar target/loopover.jar solve -s <estado> [opciones]
```

La salida estándar es la **secuencia de acciones** que lleva del estado dado al estado resuelto, en el mismo formato que acepta `verify -a`.

| Opción | Descripción | Por defecto |
|--------|-------------|-------------|
| `-s` | Estado de 32 dígitos (obligatoria). | — |
| `-e`, `--estrategia` | Estrategia de búsqueda. | `A_ESTRELLA` |
| `-h`, `--heuristica` | Función de heurística. | `MANHATTAN` |
| `-p`, `--profundidad` | Profundidad máxima de búsqueda. | `1000` |
| `-c`, `--capacidad` | Nodos máximos del árbol en memoria (SMA*). Obligatorio con `A_ESTRELLA_ACOTADA`. | `10000000` |
| `-m`, `--max-visitados` | Límite de estados visitados antes de abortar. `0` = ilimitado. | `0` |
| `-v`, `--verbose` | Imprime estadísticas tras la solución. | desactivado |

**Estrategias** (`-e`):

- `PROFUNDIDAD` — DFS.
- `ANCHURA` — BFS.
- `COSTO_UNIFORME` — uniform cost (Dijkstra).
- `VORAZ` — greedy best-first.
- `A_ESTRELLA` — A*.
- `A_ESTRELLA_ACOTADA` — SMA* con límite de nodos del árbol (`-c`, obligatorio).

**Heurísticas** (`-h`):

- `MANHATTAN` — Manhattan toroidal (por defecto).
- `MANHATTAN_ADMISIBLE` — Manhattan admisible.
- `CERO` — h = 0; la búsqueda se comporta en anchura.
- `PARES_8`, `IMPARES_8` — pattern database de 8 piezas (pares / impares).
- `PBD_8` — máximo de ambas bases (admisible).
- `PBD_8_SUMA` — suma de ambas bases (**no admisible**: no garantiza optimalidad).
- `PERMUTACIONES` — heurística de permutaciones.

```powershell
# A* con la heurística por defecto, con estadísticas
java -jar target/loopover.jar solve -s 01000203040506070809101112131415 -v

# Búsqueda en anchura limitada en profundidad
java -jar target/loopover.jar solve -s 01000203040506070809101112131415 -e ANCHURA -p 20

# SMA* con 500000 nodos en el árbol y pattern database
java -jar target/loopover.jar solve -s 01000203040506070809101112131415 -e A_ESTRELLA_ACOTADA -c 500000 -h PBD_8
```

Si el estado ya está resuelto, `solve` imprime `(ya resuelto)`. Los códigos de salida son `0` (éxito) y `1` (error de estado, de configuración o sin solución).

---

## 5. Representación de los estados

### 5.1 Cadena de 32 dígitos

Cada casilla se codifica con **dos dígitos decimales**, recorriendo el tablero en orden de casilla 0 a 15 (fila `i/4`, columna `i%4`):

```
00010203040506070809101112131415   →   estado resuelto
```

### 5.2 Bitboard

`Estado` almacena el tablero en un `long` con **4 bits por ficha**: la casilla *i* ocupa los bits `[i*4, i*4+3]`, quedando la casilla 0 en los bits menos significativos. El estado resuelto vale `0xFEDCBA98_76543210L`.

Las operaciones de fila y columna se hacen con máscaras y desplazamientos de bits, con retorno circular:

- **Fila**: bloque de 16 bits a partir de `fila * 16`; rotación circular de 4 bits.
- **Columna**: 4 nibbles separados 16 bits; rotación circular de una posición (16 bits).

`Estado` es **inmutable**: `aplicar` devuelve un nuevo objeto.

### 5.3 Codificación de acciones (5 bits)

| Bits | Campo |
|------|-------|
| 0–1 | fila (0–3) |
| 2–3 | columna (0–3) |
| 4   | signo (`1` = `+`) |

La tabla estática `Estado.ACCIONES` recoge las 32 acciones: posiciones 0–15 con signo `+`, 16–31 con signo `-`.

---

## 6. Estructura del proyecto

```
loopover_2026/
├── pom.xml            # Maven: Java 17, picocli, fastutil, guava, shade
├── src/
│   ├── LoopoverApp.java      # Punto de entrada, comando raíz (picocli)
│   ├── ComandoVerificar.java # Subcomando verify
│   ├── ComandoBusqueda.java  # Subcomando solve (mapeo estrategia/heurística)
│   ├── Estado.java           # Espacio de estados: bitboard, acciones, sucesores
│   ├── Sucesor.java          # Tupla (acción, estado, costo)
│   ├── Busqueda.java         # Motor de búsqueda (estrategias)
│   ├── Nodo.java             # Nodo del árbol de búsqueda
│   ├── Frontera.java         # Frontera (heap min-max, SMA*)
│   ├── Visitados.java        # Tabla de estados visitados
│   ├── Heuristicas.java      # Manhattan toroidal/admisible, permutaciones
│   ├── HeuristicasPBD.java   # Patrones de 8 piezas (pares/impares)
│   └── PBD.java              # Pattern database de 8 piezas
└── target/            # Artefactos de compilación (ignorado por git)
```

### Dependencias

| Dependencia | Versión | Uso |
|-------------|---------|-----|
| `info.picocli:picocli` | 4.7.7 | Parseo de argumentos y subcomandos. |
| `it.unimi.dsi:fastutil` | 8.5.16 | Listas de nodos en el camino de solución. |
| `com.google.guava:guava` | 33.7.1-jre | Declarada en el pom; sin uso en el código fuente actual. |

---

## 7. Estado de implementación

| Tarea | Alcance | Clases |
|-------|---------|--------|
| **1** | Representación del espacio de estados | `Estado` |
| **2** | Motor de búsqueda (estrategias informadas y no informadas) | `Busqueda`, `Nodo`, `Frontera`, `Visitados` |
| **3** | Heuristicas y pattern databases | `Heuristicas`, `PBD`, `HeuristicasPBD`, `Frontera.eliminarPeorHojaNoRaiz` |

**Tarea 1 (activa/pendiente):** implementar `Estado.java`. Tras completar la Tarea 1, **`verify` funcionará al completo** (mostrar sucesores y aplicar acciones), mientras que **`solve` seguirá fallando** hasta completar la Tarea 2. Las clases aún no implementadas lanzan `UnsupportedOperationException("TODO: Tarea n")` en los métodos pendientes; `Sucesor`, `ComandoVerificar`, `ComandoBusqueda` y `LoopoverApp` están completos. `Heuristicas` devuelve `0` provisionalmente para que las estrategias no informadas funcionen sin heurística.

---

## 8. Solución y verificación

La solución devuelta por `solve` puede revalidarse con `verify`:

```powershell
$sol = java -jar target/loopover.jar solve -s 01000203040506070809101112131415
java -jar target/loopover.jar verify -s 01000203040506070809101112131415 -a $sol
```

El segundo comando debe imprimir `00010203040506070809101112131415`.
