{
 "cells": [
  {
   "cell_type": "markdown",
   "id": "8dab8dc1",
   "metadata": {},
   "source": [
    "# Final Project: One Anomaly, Defended\n",
    "\n",
    "**Curso:** IELE756 – Preparación y Análisis de Datos  \n",
    "**Equipo:** Guillermo Barañao y [NOMBRE COMPAÑERO/A]  \n",
    "**Comuna analizada:** Santiago (`13101`)  \n",
    "**GitHub repository:** [PEGAR LINK DEL REPOSITORIO]  \n",
    "**Video:** [PEGAR LINK DEL VIDEO]\n",
    "\n",
    "Este notebook defiende una sola anomalía detectada en la integración de Censo 2024 y GRD 2022–2024. La idea no es volver a ejecutar todo el pipeline de las tareas anteriores, sino cargar tablas ya procesadas, aislar el hallazgo principal y revisar explicaciones alternativas."
   ]
  },
  {
   "cell_type": "markdown",
   "id": "46e53108",
   "metadata": {},
   "source": [
    "## Anomalía\n",
    "\n",
    "En Santiago, los egresos hospitalarios del capítulo CIE-10 **“Embarazo, parto y puerperio”** muestran una sobrerrepresentación extranjera muy superior a la esperada: aunque las personas extranjeras son cerca del 39% de la población comunal y cerca del 47% de las mujeres de 15 a 49 años, representan aproximadamente el 76% de los egresos obstétricos GRD entre 2022 y 2024. La anomalía no es simplemente que Santiago tenga alta inmigración: en todos los egresos GRD la proporción extranjera bordea el 35%, por lo que el patrón aparece concentrado en el capítulo obstétrico."
   ]
  },
  {
   "cell_type": "markdown",
   "id": "03309bf2",
   "metadata": {},
   "source": [
    "## 1. Librerías y rutas\n",
    "\n",
    "Se usan tablas preprocesadas del pipeline anterior. El notebook busca los archivos tanto si se ejecuta desde la raíz del repositorio como si se ejecuta desde la carpeta `notebooks/`."
   ]
  },
  {
   "cell_type": "code",
   "execution_count": null,
   "id": "1f92af37",
   "metadata": {},
   "outputs": [],
   "source": [
    "from pathlib import Path\n",
    "import pandas as pd\n",
    "import numpy as np\n",
    "import matplotlib.pyplot as plt\n",
    "\n",
    "plt.rcParams[\"figure.figsize\"] = (10, 6)\n",
    "plt.rcParams[\"axes.titlesize\"] = 13\n",
    "plt.rcParams[\"axes.labelsize\"] = 11\n",
    "\n",
    "ROOT = Path.cwd()\n",
    "if not (ROOT / \"data\").exists() and (ROOT.parent / \"data\").exists():\n",
    "    ROOT = ROOT.parent\n",
    "\n",
    "DATA = ROOT / \"data\"\n",
    "FIGS = ROOT / \"figs\"\n",
    "FIGS.mkdir(exist_ok=True)\n",
    "\n",
    "print(\"Carpeta raíz usada:\", ROOT)\n",
    "print(\"Archivos de datos disponibles:\")\n",
    "for p in sorted(DATA.glob(\"*.csv\")):\n",
    "    print(\"-\", p.name)"
   ]
  },
  {
   "cell_type": "markdown",
   "id": "58252eb2",
   "metadata": {},
   "source": [
    "## 2. Carga de tablas preprocesadas\n",
    "\n",
    "La tabla GRD ya viene filtrada a Santiago para 2022–2024, con nacionalidad agrupada y capítulo CIE-10 unido. La tabla de Censo resume los denominadores poblacionales necesarios para comparar contra la composición demográfica de la comuna."
   ]
  },
  {
   "cell_type": "code",
   "execution_count": null,
   "id": "af18c8f0",
   "metadata": {},
   "outputs": [],
   "source": [
    "gr = pd.read_csv(DATA / \"grd_santiago_2022_2024_processed.csv\", low_memory=False)\n",
    "denoms = pd.read_csv(DATA / \"census_santiago_denominators.csv\")\n",
    "\n",
    "# Asegurar tipos y variables necesarias\n",
    "for col in [\"FECHA_INGRESO\", \"FECHAALTA\", \"FECHA_NACIMIENTO\"]:\n",
    "    gr[col + \"_dt\"] = pd.to_datetime(gr[col], errors=\"coerce\", format=\"mixed\", dayfirst=True)\n",
    "\n",
    "gr[\"los\"] = (gr[\"FECHAALTA_dt\"] - gr[\"FECHA_INGRESO_dt\"]).dt.days\n",
    "gr = gr[(gr[\"los\"].isna()) | (gr[\"los\"] >= 0)].copy()\n",
    "gr[\"age_at_admission\"] = (gr[\"FECHA_INGRESO_dt\"] - gr[\"FECHA_NACIMIENTO_dt\"]).dt.days / 365.25\n",
    "gr[\"sev\"] = pd.to_numeric(gr[\"IR_29301_SEVERIDAD\"], errors=\"coerce\")\n",
    "gr[\"nat_group\"] = np.where(gr[\"NACIONALIDAD\"].astype(str).str.upper().eq(\"CHILE\"), \"Chilean\", \"Foreign\")\n",
    "\n",
    "ob_chapter = [c for c in gr[\"Capitulo\"].dropna().unique() if \"EMBARAZO\" in str(c).upper() or \"PARTO\" in str(c).upper()][0]\n",
    "ob = gr[gr[\"Capitulo\"].eq(ob_chapter)].copy()\n",
    "\n",
    "print(\"Total egresos GRD Santiago 2022-2024:\", len(gr))\n",
    "print(\"Capítulo obstétrico identificado:\", ob_chapter)\n",
    "print(\"Egresos obstétricos:\", len(ob))\n",
    "display(denoms)"
   ]
  },
  {
   "cell_type": "markdown",
   "id": "66ba949b",
   "metadata": {},
   "source": [
    "## 3. Figura principal: tamaño de la anomalía\n",
    "\n",
    "Esta figura compara cuatro referencias: composición total de la comuna, composición de mujeres 15–49 del Censo, composición de todos los egresos GRD y composición de los egresos obstétricos. Si la anomalía se explicara solo por la alta presencia migrante de Santiago, esperaríamos que el último porcentaje estuviera cerca del 39% o, de forma más exigente, cerca del 47% de mujeres 15–49. Sin embargo, el valor observado está cerca del 76%."
   ]
  },
  {
   "cell_type": "code",
   "execution_count": null,
   "id": "37e9e51e",
   "metadata": {},
   "outputs": [],
   "source": [
    "all_counts = gr[\"nat_group\"].value_counts()\n",
    "ob_counts = ob[\"nat_group\"].value_counts()\n",
    "\n",
    "headline = pd.DataFrame([\n",
    "    {\n",
    "        \"Indicador\": \"Censo: población total\",\n",
    "        \"foreign_share\": denoms.loc[denoms[\"metric\"].eq(\"Población total Censo 2024\"), \"foreign_share\"].iloc[0]\n",
    "    },\n",
    "    {\n",
    "        \"Indicador\": \"Censo: mujeres 15-49\",\n",
    "        \"foreign_share\": denoms.loc[denoms[\"metric\"].eq(\"Mujeres 15-49 Censo 2024\"), \"foreign_share\"].iloc[0]\n",
    "    },\n",
    "    {\n",
    "        \"Indicador\": \"GRD: todos los egresos\",\n",
    "        \"foreign_share\": all_counts.get(\"Foreign\", 0) / len(gr)\n",
    "    },\n",
    "    {\n",
    "        \"Indicador\": \"GRD: embarazo/parto\",\n",
    "        \"foreign_share\": ob_counts.get(\"Foreign\", 0) / len(ob)\n",
    "    },\n",
    "])\n",
    "headline[\"foreign_share_pct\"] = headline[\"foreign_share\"] * 100\n",
    "\n",
    "display(headline)\n",
    "\n",
    "plt.figure(figsize=(10, 6))\n",
    "x = np.arange(len(headline))\n",
    "plt.bar(x, headline[\"foreign_share_pct\"])\n",
    "plt.xticks(x, [\"Censo\n",
    "población total\", \"Censo\n",
    "mujeres 15-49\", \"GRD\n",
    "todos los egresos\", \"GRD\n",
    "embarazo/parto\"])\n",
    "plt.ylabel(\"Porcentaje extranjero (%)\")\n",
    "plt.title(\"Anomalía en Santiago: sobrerrepresentación extranjera en egresos obstétricos\")\n",
    "for i, v in enumerate(headline[\"foreign_share_pct\"]):\n",
    "    plt.text(i, v + 1.2, f\"{v:.1f}%\", ha=\"center\")\n",
    "plt.ylim(0, headline[\"foreign_share_pct\"].max() + 12)\n",
    "plt.tight_layout()\n",
    "plt.savefig(FIGS / \"headline.png\", dpi=180)\n",
    "plt.show()"
   ]
  },
  {
   "cell_type": "markdown",
   "id": "bd1013d1",
   "metadata": {},
   "source": [
    "**Lectura de la figura.** La diferencia clave no está entre Santiago y otra comuna, sino entre el patrón demográfico esperado y el patrón hospitalario observado. Las personas extranjeras son una parte importante de Santiago, pero su participación en egresos obstétricos es mucho mayor que su peso en la población total, que su peso entre mujeres de 15 a 49 años y que su peso en los egresos hospitalarios generales."
   ]
  },
  {
   "cell_type": "markdown",
   "id": "f3f2cc19",
   "metadata": {},
   "source": [
    "## 4. Código de aislamiento del número principal\n",
    "\n",
    "Este bloque contiene el cálculo mínimo que produce el número principal de la anomalía: conteos de egresos obstétricos por nacionalidad y comparación contra denominadores censales."
   ]
  },
  {
   "cell_type": "code",
   "execution_count": null,
   "id": "cce201f9",
   "metadata": {},
   "outputs": [],
   "source": [
    "isolation = pd.DataFrame({\n",
    "    \"grupo\": [\"Chilean\", \"Foreign\"],\n",
    "    \"egresos_obstetricos\": [int(ob_counts.get(\"Chilean\", 0)), int(ob_counts.get(\"Foreign\", 0))],\n",
    "    \"mujeres_15_49_censo\": [\n",
    "        int(denoms.loc[denoms[\"metric\"].eq(\"Mujeres 15-49 Censo 2024\"), \"chilean\"].iloc[0]),\n",
    "        int(denoms.loc[denoms[\"metric\"].eq(\"Mujeres 15-49 Censo 2024\"), \"foreign\"].iloc[0]),\n",
    "    ]\n",
    "})\n",
    "isolation[\"tasa_obstetrica_por_10k_3_anios\"] = isolation[\"egresos_obstetricos\"] / isolation[\"mujeres_15_49_censo\"] * 10000\n",
    "rate_ratio = isolation.loc[isolation[\"grupo\"].eq(\"Foreign\"), \"tasa_obstetrica_por_10k_3_anios\"].iloc[0] / isolation.loc[isolation[\"grupo\"].eq(\"Chilean\"), \"tasa_obstetrica_por_10k_3_anios\"].iloc[0]\n",
    "\n",
    "display(isolation)\n",
    "print(f\"Razón de tasas Foreign/Chilean: {rate_ratio:.2f}x\")"
   ]
  },
  {
   "cell_type": "markdown",
   "id": "74e0ea60",
   "metadata": {},
   "source": [
    "**Interpretación.** En términos de tasa bruta acumulada 2022–2024, los egresos obstétricos por cada 10.000 mujeres de 15 a 49 años son varias veces mayores en el grupo extranjero que en el grupo chileno. Esta comparación no prueba causalidad individual, pero sí muestra que la anomalía persiste al ajustar por el denominador más relevante disponible."
   ]
  },
  {
   "cell_type": "markdown",
   "id": "8d47b932",
   "metadata": {},
   "source": [
    "## 5. Chequeo alternativo 1: ¿es solo un efecto de un año específico?\n",
    "\n",
    "Si la anomalía se debiera a un evento puntual, el porcentaje extranjero del capítulo obstétrico debería aparecer concentrado en un solo año. El chequeo muestra que no ocurre eso: la sobrerrepresentación aparece en 2022, 2023 y 2024."
   ]
  },
  {
   "cell_type": "code",
   "execution_count": null,
   "id": "3069190e",
   "metadata": {},
   "outputs": [],
   "source": [
    "by_year_ob = ob.groupby([\"year\", \"nat_group\"]).size().unstack(fill_value=0)\n",
    "by_year_ob[\"obstetric_foreign_share\"] = by_year_ob.get(\"Foreign\", 0) / (by_year_ob.get(\"Foreign\", 0) + by_year_ob.get(\"Chilean\", 0))\n",
    "\n",
    "by_year_all = gr.groupby([\"year\", \"nat_group\"]).size().unstack(fill_value=0)\n",
    "by_year_all[\"all_grd_foreign_share\"] = by_year_all.get(\"Foreign\", 0) / (by_year_all.get(\"Foreign\", 0) + by_year_all.get(\"Chilean\", 0))\n",
    "\n",
    "check_year = by_year_ob[[\"Chilean\", \"Foreign\", \"obstetric_foreign_share\"]].join(by_year_all[[\"all_grd_foreign_share\"]])\n",
    "display(check_year)\n",
    "\n",
    "plt.figure(figsize=(9, 5))\n",
    "plt.plot(check_year.index, check_year[\"obstetric_foreign_share\"] * 100, marker=\"o\", label=\"Embarazo/parto/puerperio\")\n",
    "plt.plot(check_year.index, check_year[\"all_grd_foreign_share\"] * 100, marker=\"o\", label=\"Todos los egresos GRD\")\n",
    "plt.ylabel(\"Porcentaje extranjero (%)\")\n",
    "plt.xlabel(\"Año\")\n",
    "plt.title(\"Chequeo 1: la anomalía no depende de un solo año\")\n",
    "plt.xticks(check_year.index)\n",
    "plt.legend()\n",
    "plt.grid(True, alpha=0.3)\n",
    "plt.tight_layout()\n",
    "plt.savefig(FIGS / \"check_by_year.png\", dpi=180)\n",
    "plt.show()"
   ]
  },
  {
   "cell_type": "markdown",
   "id": "2e5761d8",
   "metadata": {},
   "source": [
    "**Resultado del chequeo 1.** La participación extranjera en egresos obstétricos se mantiene sobre 73% en los tres años, mientras que en el total de egresos GRD se mantiene cerca de 35%. Por lo tanto, el patrón no parece ser un salto aislado de un año."
   ]
  },
  {
   "cell_type": "markdown",
   "id": "e7b97b39",
   "metadata": {},
   "source": [
    "## 6. Chequeo alternativo 2: ¿se explica solo porque hay más mujeres extranjeras en edad fértil?\n",
    "\n",
    "Una explicación razonable es que la población extranjera de Santiago sea más joven y tenga mayor proporción de mujeres en edad reproductiva. Por eso, el chequeo compara contra mujeres de 15 a 49 años del Censo y calcula tasas por 10.000 mujeres."
   ]
  },
  {
   "cell_type": "code",
   "execution_count": null,
   "id": "02ba4b8e",
   "metadata": {},
   "outputs": [],
   "source": [
    "rates = isolation.copy()\n",
    "display(rates)\n",
    "\n",
    "plt.figure(figsize=(8, 5))\n",
    "plt.bar(rates[\"grupo\"], rates[\"tasa_obstetrica_por_10k_3_anios\"])\n",
    "plt.ylabel(\"Egresos obstétricos por 10.000 mujeres 15-49 (2022-2024)\")\n",
    "plt.title(\"Chequeo 2: tasa ajustada por mujeres 15-49\")\n",
    "for i, v in enumerate(rates[\"tasa_obstetrica_por_10k_3_anios\"]):\n",
    "    plt.text(i, v + 20, f\"{v:.1f}\", ha=\"center\")\n",
    "plt.tight_layout()\n",
    "plt.savefig(FIGS / \"check_rates.png\", dpi=180)\n",
    "plt.show()"
   ]
  },
  {
   "cell_type": "markdown",
   "id": "2be0dad6",
   "metadata": {},
   "source": [
    "**Resultado del chequeo 2.** Incluso usando como denominador solo mujeres de 15 a 49 años, la tasa acumulada de egresos obstétricos del grupo extranjero es muy superior a la del grupo chileno. Esto no elimina todas las diferencias de composición —por ejemplo, fecundidad, aseguramiento o patrones de uso hospitalario—, pero descarta que el hallazgo sea solamente consecuencia mecánica de tener más población extranjera en Santiago."
   ]
  },
  {
   "cell_type": "markdown",
   "id": "35125bd0",
   "metadata": {},
   "source": [
    "## 7. Chequeo alternativo 3: ¿son pocos casos extremos o casos más severos?\n",
    "\n",
    "Otra posibilidad es que la anomalía sea causada por pocos egresos extranjeros de mucha complejidad o estadías largas. Para revisar esto, comparamos estadía hospitalaria y severidad media dentro del mismo capítulo obstétrico."
   ]
  },
  {
   "cell_type": "code",
   "execution_count": null,
   "id": "b39e20df",
   "metadata": {},
   "outputs": [],
   "source": [
    "los_sev = ob.groupby(\"nat_group\").agg(\n",
    "    egresos=(\"nat_group\", \"size\"),\n",
    "    los_promedio=(\"los\", \"mean\"),\n",
    "    los_mediana=(\"los\", \"median\"),\n",
    "    los_max=(\"los\", \"max\"),\n",
    "    severidad_promedio=(\"sev\", \"mean\"),\n",
    "    severidad_mediana=(\"sev\", \"median\"),\n",
    ").reset_index()\n",
    "\n",
    "display(los_sev)\n",
    "\n",
    "plt.figure(figsize=(8, 5))\n",
    "plt.bar(los_sev[\"nat_group\"], los_sev[\"los_promedio\"])\n",
    "plt.ylabel(\"Estadía promedio (días)\")\n",
    "plt.title(\"Chequeo 3: estadía promedio en capítulo obstétrico\")\n",
    "for i, v in enumerate(los_sev[\"los_promedio\"]):\n",
    "    plt.text(i, v + 0.05, f\"{v:.2f}\", ha=\"center\")\n",
    "plt.tight_layout()\n",
    "plt.savefig(FIGS / \"check_los.png\", dpi=180)\n",
    "plt.show()"
   ]
  },
  {
   "cell_type": "markdown",
   "id": "50c3dd95",
   "metadata": {},
   "source": [
    "**Resultado del chequeo 3.** La anomalía no parece explicarse por pocos casos extremos de mayor estadía o severidad. De hecho, dentro del capítulo obstétrico, el grupo extranjero tiene una estadía promedio levemente menor y una severidad promedio similar o menor que el grupo chileno. Esto sugiere que el hallazgo se relaciona más con volumen relativo de egresos obstétricos que con complejidad hospitalaria."
   ]
  },
  {
   "cell_type": "markdown",
   "id": "cbd47e8c",
   "metadata": {},
   "source": [
    "## 8. Discusión: qué sugiere y qué no puede decir\n",
    "\n",
    "Este resultado sugiere que, en Santiago, la demanda hospitalaria asociada a embarazo, parto y puerperio está mucho más concentrada en población extranjera de lo que se esperaría mirando solo la composición comunal. Para política pública, el hallazgo apunta a la necesidad de mirar salud materna, acceso oportuno a controles, continuidad de atención y barreras administrativas o culturales en comunas con alta migración. Sin embargo, este análisis es ecológico y usa egresos, no personas únicas: no permite afirmar que ser extranjera cause más hospitalizaciones obstétricas. Tampoco observa nacimientos fuera del sistema GRD ni atención privada, por lo que debe interpretarse como una señal para profundizar, no como una conclusión causal."
   ]
  },
  {
   "cell_type": "markdown",
   "id": "82e7639f",
   "metadata": {},
   "source": [
    "## 9. Conclusión para el video\n",
    "\n",
    "La anomalía principal es que Santiago no solo tiene alta presencia extranjera: en los egresos hospitalarios de embarazo, parto y puerperio, la participación extranjera llega a cerca de tres cuartos de los casos. El patrón se mantiene en 2022, 2023 y 2024, no aparece con la misma fuerza en los egresos generales y persiste al usar como denominador mujeres de 15 a 49 años. Por eso, el hallazgo es defendible como una anomalía de salud materna a nivel comunal, aunque no debe leerse como causalidad individual. Con una semana más, el siguiente paso sería comparar con comunas vecinas de la Región Metropolitana y separar nacimientos, abortos, complicaciones y controles para distinguir fecundidad, acceso y severidad clínica."
   ]
  }
 ],
 "metadata": {},
 "nbformat": 4,
 "nbformat_minor": 5
}
