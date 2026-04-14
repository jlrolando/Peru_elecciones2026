ONPE 2026: Extrapolación Presidencial 🗳️
Este proyecto contiene un pipeline desarrollado en R para extraer, procesar y proyectar los resultados de las elecciones presidenciales de Perú 2026.

🛠️ Funcionalidades
Data Scraping: Conexión directa a la API de ONPE para resultados distritales y del extranjero.

Extrapolación Robusta: Imputación de distritos sin actas basada en promedios provinciales y departamentales.

Filtro de Sesgo: Exclusión de grandes ciudades en el cálculo de promedios para proteger la representatividad rural.

Proyección de Segunda Vuelta: Cálculo automático de márgenes entre los tres primeros candidatos.

💻 Stack Técnico
Lenguaje: R.

Librerías: httr, jsonlite, dplyr, readr.

Hardware: Optimizado para entornos de alto rendimiento (bioinformática/servidores).

📂 Estructura
fetch_data_analysis.R: Script principal de extracción y análisis.

Logs: Sistema de monitoreo en tiempo real para el seguimiento de las ~3,700 peticiones API.

Desarrollado para el análisis técnico de datos electorales.
