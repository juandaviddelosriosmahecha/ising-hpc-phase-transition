# ising-hpc-phase-transition

Simulación por Monte Carlo de la **transición de fase del modelo de Ising 2D**,
implementada con tres enfoques de cómputo de alto rendimiento (HPC) para comparar
algoritmos y plataformas de paralelización: **C++ secuencial**, **OpenMP (CPU
multinúcleo)** y **CUDA (GPU)**.

## ¿Qué es esto?

El [modelo de Ising](https://es.wikipedia.org/wiki/Modelo_de_Ising) es un modelo de
mecánica estadística de espines (`+1` / `-1`) sobre una red. En dos dimensiones
presenta una **transición de fase de segundo orden** a la temperatura crítica
`T_c ≈ 2.269` (en unidades de `J/k_B`), donde el sistema pasa de un estado
ordenado (ferromagnético) a uno desordenado (paramagnético).

Este proyecto simula una red cuadrada `L×L` con condiciones de frontera periódicas
y mide, en función de la temperatura, los observables termodinámicos que
caracterizan la transición:

- **Magnetización por sitio** `⟨|M|⟩`
- **Energía por sitio** `E`
- **Susceptibilidad magnética** `χ`
- **Calor específico** `C`
- **Cumulante de Binder** `U`

Cada implementación recorre un rango de temperaturas, termaliza el sistema, toma
muestras decorrelacionadas y vuelca los resultados a un archivo CSV que luego se
grafica con `src/graf.py`.

## Algoritmos e implementaciones

| Carpeta       | Algoritmo       | Plataforma         | Descripción |
|---------------|-----------------|--------------------|-------------|
| `src/`        | **Wolff**       | C++ secuencial     | Algoritmo de cluster que voltea un cúmulo de espines por paso. Reduce el *critical slowing down* cerca de `T_c`. Es la implementación de referencia. |
| `OpenMP/`     | **Swendsen-Wang** | C++ + OpenMP (CPU) | Algoritmo de cluster que particiona toda la red con *union-find*; paralelizado sobre varios hilos de CPU. |
| `Metropolis/` | **Metropolis**  | CUDA (GPU)         | Actualización local tipo *checkerboard* (tablero de ajedrez) ejecutada en GPU con `curand` para los números aleatorios. |

Los núcleos físicos son equivalentes en los tres casos; cambian el algoritmo de
muestreo y la estrategia de paralelización.

## Estructura del repositorio

```
ising-hpc-phase-transition/
├── Makefile                 # Compila la versión Wolff (C++ secuencial)
├── Ising.ipynb              # Notebook de exploración/análisis
├── src/                     # Implementación de referencia (Wolff, C++)
│   ├── main.cpp             # Punto de entrada y configuración de la simulación
│   ├── wolf.h / wolf.cpp    # Algoritmo de Wolff y cálculo de observables
│   ├── SquareLattice.h/.cpp # Red cuadrada de espines
│   └── graf.py              # Graficación de resultados (Python/Matplotlib)
├── OpenMP/
│   └── Swendsen-Wang.cpp    # Versión paralela en CPU (OpenMP)
├── Metropolis/
│   └── cuda.cu              # Versión en GPU (CUDA, Metropolis checkerboard)
└── Resultados/              # CSV de salida ya generados
    ├── CUDA/                # Metropolis en GPU para L = 32, 64, 128, 256
    └── OPENMP/              # Swendsen-Wang en CPU para L = 32
```

## Requisitos

- **C++17** y `make` (versión Wolff).
- **OpenMP** (incluido en GCC/Clang recientes) para la versión de CPU paralela.
- **CUDA Toolkit** y una GPU NVIDIA (`nvcc`) para la versión de GPU.
- **Python 3** con `numpy`, `matplotlib`, `scipy` y `pandas` para graficar.

## Compilación y ejecución

### 1. Wolff — C++ secuencial (referencia)

```bash
make            # genera el ejecutable ./ising_simulation
make run        # compila y ejecuta
make clean      # elimina objetos y binario
```

Los parámetros de la simulación (`J`, `L`, rango de temperatura, iteraciones) se
configuran al inicio de `src/main.cpp`. El resultado se guarda en
`ising_results_L<L>.csv`. Con `save_snapshots = true` también se generan archivos
de evolución temporal por temperatura.

> Nota: `src/wolf.h` incluye `SquareLattice.h` con una ruta absoluta
> (`/content/...`) heredada de un entorno tipo Google Colab. Si compilas fuera de
> ese entorno, cámbiala por `#include "SquareLattice.h"`.

### 2. Swendsen-Wang — OpenMP (CPU)

```bash
g++ -std=c++17 -O3 -fopenmp OpenMP/Swendsen-Wang.cpp -o swendsen_wang
OMP_NUM_THREADS=8 ./swendsen_wang
```

Los parámetros se definen en la función `main()` de `OpenMP/Swendsen-Wang.cpp`.
La salida es `swendsen_wang_results_L<L>.csv`.

### 3. Metropolis — CUDA (GPU)

```bash
nvcc -O3 Metropolis/cuda.cu -o ising_cuda
./ising_cuda
```

Los parámetros (`L`, temperaturas, pasos de termalización, iteraciones) son
`#define` al inicio de `Metropolis/cuda.cu`. La salida es `ising_results.csv`.

## Formato de salida (CSV)

Todas las implementaciones producen un CSV con las mismas columnas:

```
Temperature,Magnetization,Energy,Susceptibility,SpecificHeat,BinderCumulant
```

Cada fila corresponde a una temperatura del barrido.

## Visualización de resultados

```bash
python3 src/graf.py <archivo.csv> -o salida --L 32 --format png
```

Genera dos figuras:

- `salida_phase_transition.<fmt>`: magnetización, susceptibilidad, calor específico
  y cumulante de Binder frente a la temperatura, con la línea crítica `T_c ≈ 2.269`.
- `salida_energy_analysis.<fmt>`: energía por sitio y su derivada `∂E/∂T`.

Opciones útiles: `--J`, `--Tmin`, `--Tmax`, `--iter` para anotar los parámetros, y
`--format {pdf,png,svg}` para el formato de salida (PDF por defecto).

## Resultados incluidos

La carpeta `Resultados/` contiene CSV ya generados para distintos tamaños de red,
útiles para graficar sin necesidad de volver a ejecutar las simulaciones:

- `Resultados/CUDA/`: Metropolis en GPU para `L = 32, 64, 128, 256`.
- `Resultados/OPENMP/`: Swendsen-Wang en CPU para `L = 32`.
