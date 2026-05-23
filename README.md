# IELE756 – Preparación y Análisis de Datos
## Proyecto Final: Una Anomalía, Defendida

**Integrante:** Guillermo Barañao  
**Comuna:** Santiago (código 13101)  
**Región:** Metropolitana  

---

## La anomalía

En Santiago, los egresos hospitalarios del capítulo CIE-10 *"Embarazo, parto y puerperio"* muestran una sobrerrepresentación extranjera muy superior a la esperada: aunque las personas extranjeras representan cerca del 39% de la población comunal y aproximadamente el 47% de las mujeres de 15 a 49 años, concentran alrededor del **76% de los egresos obstétricos GRD** entre 2022 y 2024. El patrón no se replica en el total de egresos GRD de la comuna, donde la proporción extranjera ronda el 35%, lo que descarta que sea un efecto de escala demográfica general.

---

## Estructura del repositorio

```
iele756-region-4/
│
├── README.md
├── requirements.txt
│
├── notebooks/
│   ├── tarea0.ipynb
│   ├── tarea1.ipynb
│   ├── tarea2.ipynb
│   ├── tarea3.ipynb
│   └── final_anomaly.ipynb        ← notebook del proyecto final
│
├── data/                          ← rutas o scripts de descarga (sin archivos grandes)
│
└── figs/
    ├── headline.png               ← figura principal usada en el video
    └── ...                        ← figuras de apoyo y diagnóstico
```

---

## La anomalía en detalle

| Indicador | Valor |
|---|---|
| Comunas analizadas | Santiago (13101) |
| Variable principal | Egresos GRD capítulo CIE-10 O00–O99 (Embarazo, parto y puerperio) |
| Período | 2022–2024 |
| % población extranjera en la comuna | ~39% |
| % mujeres extranjeras 15–49 años | ~47% |
| % egresos obstétricos de personas extranjeras | **~76%** |
| % egresos GRD totales de personas extranjeras | ~35% |

El notebook que produce la figura principal es **`notebooks/final_anomaly.ipynb`**.  
Tiempo estimado de ejecución: ~2–3 minutos (carga tablas precomputadas, no re-ejecuta el pipeline completo).

---

## Cómo reproducir

### 1. Clonar el repositorio

```bash
git clone https://github.com/gbaranaos/iele756-region-4.git
cd iele756-region-4
```

### 2. Instalar dependencias

```bash
pip install -r requirements.txt
```

### 3. Ejecutar el notebook

```bash
jupyter notebook notebooks/final_anomaly.ipynb
```

> **Nota sobre los datos:** Los archivos de datos no se incluyen directamente en el repositorio por su tamaño. Ver la carpeta `data/` para instrucciones de descarga o rutas locales esperadas.

---

## Declaración de uso de IA

Durante el desarrollo de este proyecto se utilizó **ChatGPT** como herramienta de asistencia. Su uso incluyó: apoyo en la redacción de este README, sugerencias de estructura para el notebook `final_anomaly.ipynb`, y debugging de código Python. Toda decisión analítica, interpretación de resultados y validación de cifras fue realizada de forma independiente por el autor. El uso de IA se declara de forma transparente conforme a la política del curso.

---

## Video

🎥 [Link al video (por agregar)]()

---



- IELE756 – Preparación y Análisis de Datos, Leo Ferres, PhD
- Entrega: viernes 22 de mayo de 2026
