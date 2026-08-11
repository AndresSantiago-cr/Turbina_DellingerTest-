# Validación CFD de un Tornillo de Arquímedes

**Semillero SIIM — Universidad Pontificia Bolivariana, Bucaramanga**

Modelado numérico multifásico de una turbina de tornillo de Arquímedes en
Ansys Fluent, con validación contra los datos experimentales de
Dellinger et al. (2018).

![Estado](https://img.shields.io/badge/estado-en_validaci%C3%B3n-orange)
![Solver](https://img.shields.io/badge/solver-Fluent_25.2_R2-blue)
![Modelo](https://img.shields.io/badge/modelo-VOF_multif%C3%A1sico-lightgrey)

---

## Objetivo

Reproducir numéricamente el punto de operación experimental de Dellinger
et al. y demostrar que el modelo es capaz de predecir el **torque en el eje**
y la **eficiencia hidráulica** dentro de las tolerancias de aceptación
definidas en el protocolo de validación:

| Métrica | Tolerancia |
|---|---|
| Torque en el eje | ≤ 10 % de error |
| Eficiencia η | ≤ 8 puntos porcentuales |
| Posición de η máxima | ± 15 % |

Una vez validado, el modelo se usará para dimensionar un tornillo adaptado
a las condiciones de caudal y salto disponibles en el sitio de interés.

---

## Estado actual

> **El modelo todavía NO está validado.** Hay un defecto geométrico
> identificado que bloquea la comparación con el experimento.

### Hallazgo principal

El dominio CAD contiene **2 hélices en lugar de 3**. El análisis del archivo
STEP encontró 24 superficies B-spline donde deberían existir 36, con fases
angulares en 73° y 193°, y la tercera posición (313°) vacía.

Esto se confirmó por dos vías independientes:

- **Geométrica.** Concentración de simetría rotacional: N=3 → 0.995,
  N=6 → 0.980, pero N=1/2/4 → ≈0.50.
- **Espectral.** El pico medido de la FFT del torque está en **4.244 Hz**.
  Con f_rotación = 2.170 Hz, dos álabes predicen 4.340 Hz (−2.2 %) y tres
  predicen 6.511 Hz (−34.8 %).

Con un tercio menos de superficie de empuje, el torque queda sistemáticamente
por debajo del valor experimental.

### Estado numérico

| Indicador | Valor | Criterio | ¿Cumple? |
|---|---|---|---|
| Torque (últimas 4 rev, solo `tornillo`) | ≈ 0.1979 N·m | 0.2370 ± 10 % | ❌ −16.5 % |
| Variación entre revoluciones consecutivas | hasta 31 % | < 0.5 % | ❌ |
| Error de conservación de masa (agua) | −5.3 % | < 1 % | ❌ |
| Balance de la fase aire | +0.000845 kg/s | ≈ 0 | ✅ |
| Interfaces de malla | 100 % solape | sin huecos | ✅ |
| Eje de rotación | 24.00° exacto | 24° | ✅ |

**Todavía no se alcanza el régimen estacionario.** Las medias por revolución
siguen oscilando sin estabilizarse, así que ningún promedio actual es
reportable como resultado.

---

## Configuración del caso

### Geometría de referencia (Dellinger et al., 2018)

| Parámetro | Símbolo | Valor |
|---|---|---|
| Radio exterior | R_a | 0.096 m |
| Radio interior del eje | R_i | 0.052 m |
| Paso | S | 0.192 m |
| Longitud del tornillo | L_B | 0.400 m |
| Número de álabes | N | **3** |
| Espesor del álabe | s_sp | 0.001 m |
| Ángulo de inclinación | β | 24° |

El STEP actual reproduce con exactitud todas las dimensiones
(R_i = 52 mm, artesa = 97 mm, S = 192.0 mm, β = 24.00°). El único
defecto es el número de álabes.

### Solver

| Parámetro | Valor |
|---|---|
| Velocidad de giro | −13.636 rad/s = **130.21 rpm** |
| Paso temporal | 0.00192299997434 s = 1.502°/paso |
| Pasos por revolución | 240 |
| Periodo de revolución | 0.46078 s |
| Avance simulado | 4.4383 s ≈ 9.63 revoluciones |
| Iteraciones máx. por paso | 20 |
| Movimiento de zona | Mesh Motion (no MRF) |
| Eje de rotación | (0.913545, 0, −0.406737) |

Los primeros 100 pasos se corrieron con dt = 0.0005 s para arrancar.

---

## Estructura del repositorio

```
.
├── README.md                                   Este archivo
├── Protocolo_validacion_modelado_CFD_tornillo.md   Protocolo completo
├── PROGRESO_validacion_CFD_tornillo.md             Bitácora de depuración
├── autopush.ps1                                Subida automática a GitHub
├── .gitignore                                  Excluye binarios pesados
│
├── geometria/
│   └── dominio_3cuerpos_V2.step                Dominio fluido
│
└── Workbench/
    └── <proyecto>_files/dp0/FFF/Fluent/
        ├── report-def-0-rfile.out              Torque (momento)
        ├── report-def-1-rfile.out              Torque (duplicado)
        ├── report-def-2-rfile.out
        ├── mdot_inlet-rfile.out                Flujo másico entrada
        ├── mdot_outlet-rfile.out               Flujo másico salida
        ├── mdot_intf-rfile.out                 Flujo en la interfaz
        ├── vol_agua_entrada-rfile.out          Volumen de agua
        ├── vol_agua_descarga-rfile.out
        └── Solution.trn                        Transcript
```

### Qué se versiona

Solo texto y geometría de intercambio. Los archivos `.h5`, `.dat`, `.cas`
y `.msh` están excluidos por el `.gitignore`: pesan cientos de MB y GitHub
rechaza cualquier archivo de más de 100 MB. Los monitores `.out` pesan
~50 KB, así que se pueden versionar indefinidamente.

---

## Formato de los monitores

Los archivos `.out` de Fluent tienen esta cabecera:

```
("Time Step" "<nombre>" "flow-time")
```

Son texto plano de tres columnas. Para leerlos:

```python
import numpy as np

def leer_out(ruta):
    datos = np.loadtxt(ruta, skiprows=3)
    return datos[:, 0], datos[:, 1], datos[:, 2]  # paso, valor, tiempo

paso, torque, t = leer_out("report-def-0-rfile.out")

# Promedio sobre las últimas 4 revoluciones (240 pasos cada una)
print(f"Torque medio: {torque[-960:].mean():.5f} N·m")
```

---

## Hoja de ruta

### Fase A — Corregir la geometría (bloqueante)

- [ ] Rehacer el CAD con patrón circular de **3 instancias** sobre 360°
- [ ] Verificar la tercera hélice en la fase **313°**
- [ ] Rehacer la operación booleana del dominio fluido
- [ ] Exportar STEP y confirmar **36 superficies B-spline**
- [ ] Volver a mallar

### Fase B — Corregir la configuración

- [ ] B1 — Quitar `artesa` de `report-def-0`; dejar solo `tornillo`
- [ ] B2 — `Average Over (Time Steps)` = **240**
- [ ] B3 — Redefinir `report-def-1` como torque **viscoso**
- [ ] B4 — Apuntar `mdot_intf` a `intf:01:entrada-...::int_rot_arriba`
- [ ] B5 — Subir `Max Iterations/Time Step` de 20 a **40**
- [ ] B6 — Revisar el esquema VOF (Explícito ⇒ Courant ≤ 0.25)
- [ ] B7 — Ejecutar `General → Check`
- [ ] B8 — `File → Write → Start Transcript` antes de correr

> **Nota sobre B1.** El reporte de momento actual incluye la pared `artesa`,
> que es estática. Su contribución cancela parte del torque real del rotor:
> 0.16621 N·m con artesa contra 0.18741 N·m solo con `tornillo`, una
> supresión del 11.3 %.

### Fase C — Correr hasta régimen

- [ ] dV/dt ≈ 0 con signo consistente
- [ ] Residual de continuidad < 10⁻⁵
- [ ] Error de conservación de agua < 1 %
- [ ] Activar `Data Sampling for Time Statistics` **solo ya en régimen**
- [ ] Promediar sobre **4 revoluciones**, con < 0.5 % entre consecutivas

### Fase D — Independencia de malla

- [ ] Tres mallas: ~0.7 M / 1.4 M / 2.2 M celdas, con r ≥ 1.3
- [ ] Extrapolación de Richardson + GCI (ASME V&V 20)
- [ ] Reportar el torque como **C ± GCI**

---

## Nota metodológica: la pregunta de la regresión

Surgió la propuesta de hacer una regresión lineal sobre los resultados y
proyectar el valor que se obtendría con más capacidad de cómputo. Conviene
precisar sobre qué variable.

**Sobre el tiempo de simulación no es válido.** Una serie temporal de torque
no tiende asintóticamente a un valor por extrapolación lineal: oscila
periódicamente alrededor de una media. Una regresión sobre 19 valores
consecutivos dio pendiente +0.001116/paso con **R² = 0.098** y t ≈ 1.36, es
decir, sin significancia estadística. No hay tendencia; hay ruido.

**Sobre el refinamiento de malla sí es el procedimiento correcto**, y es
precisamente la extrapolación de Richardson de la Fase D. Aplicada a la
tabla de mallas de Dellinger (2.4M/5.0M/8.8M → 0.2423/0.2370/0.2360):

```
r₂₁ = 1.207     p = 7.8 (fuera del rango asintótico)
C(h→0) = 0.2357 N·m     GCI₂₁ = 0.16 %
```

El refinamiento es **monótonamente decreciente**. Eso significa que refinar
más la malla no puede hacer subir el torque; converge por debajo. La
diferencia con el experimento no se cierra con más celdas: se cierra
corrigiendo la geometría.

---

## Referencias

1. **Dellinger, G., Garambois, P.-A., Dufresne, M., Terfous, A., Vazquez, J.,
   Ghenaim, A.** (2018). Numerical and experimental study of an Archimedean
   Screw Generator. *Renewable Energy*, **118**, 847–857.
   — Fuente de validación primaria.

2. **Rohmer, J., Knittel, D., Sturtzer, G., Flieller, D., Renaud, J.** (2016).
   Modeling and experimental results of an Archimedes screw turbine.
   *Renewable Energy*, **94**, 136–146.

3. **ASME V&V 20-2009.** Standard for Verification and Validation in
   Computational Fluid Dynamics and Heat Transfer.
   — Metodología GCI de la Fase D.

---

## Herramientas

- **Ansys Fluent 25.2 R2** — solver VOF multifásico
- **Ansys Workbench** — gestión del proyecto
- **SolidWorks** — geometría paramétrica
- **Python 3 + NumPy** — postproceso de los monitores
- **PowerShell** — automatización de la subida a GitHub

---

## Licencia y uso

Repositorio privado de trabajo académico del Semillero SIIM, UPB Bucaramanga.
Los datos experimentales de referencia pertenecen a sus autores originales y
se citan únicamente con fines de validación.
