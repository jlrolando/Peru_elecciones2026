# ONPE 2026: Extrapolación Presidencial 🗳️

Este proyecto contiene un pipeline desarrollado en **R** para extraer, procesar y proyectar los resultados de las elecciones presidenciales de Perú 2026 en tiempo real.

### 🛠️ Funcionalidades
* **Data Scraping**: Conexión directa a la API de ONPE para resultados distritales y del extranjero.
* **Extrapolación Robusta**: Imputación de distritos sin actas basada en promedios provinciales y departamentales.
* **Filtro de Sesgo**: Exclusión de grandes ciudades en el cálculo de promedios para proteger la representatividad rural.
* **Proyección de Segunda Vuelta**: Cálculo automático de márgenes entre los candidatos líderes.

### 💻 Stack Técnico
* **Lenguaje**: R.
* **Librerías**: `httr`, `jsonlite`, `dplyr`, `readr`.
* **Entorno**: Optimizado para procesamiento en servidores de alto rendimiento (16 CPUs / 128 GB RAM).

### 📂 Estructura
* `fetch_data_analysis.R`: Script principal de extracción y análisis.
* **Logs**: Sistema de monitoreo para el seguimiento de las ~3,700 peticiones API necesarias.

---
*Desarrollado para el análisis técnico y procesamiento de datos electorales.*
Tip adicional: Si quieres que los emojis no se vean raros, asegúrate de que al guardar el archivo en tu editor (o en GitHub) la codificación sea UTF-8.
