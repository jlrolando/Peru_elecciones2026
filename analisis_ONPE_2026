#!/usr/bin/env Rscript
# ==============================================================================
# ONPE 2026 — Extrapolation from Browser-Fetched CSV (v2)
#
# CHANGES FROM v1:
#   - Extranjero data is now at CITY level (not country level)
#   - Imputation hierarchy for zero-acta cities voto extranjero:
#       1. Country average (other cities in same country with data)
#       2. Continent average (other cities in same continent)
#   - est_valid uses any candidate with data, not just Fujimori
#   - Breakdown now shows per-country totals within each continent
#
# DOMESTIC:
#   Unchanged — province → department fallback with big-city exclusion
#
# OUTPUT:
#   - onpe_2026_extrapolated_full.csv
#   - onpe_extrapolation_output.txt
# ==============================================================================

library(readr)

setwd("")

# --- Log setup ----------------------------------------------------------------
options(cli.num_colors = 0, crayon.enabled = FALSE, cli.unicode = FALSE,
        readr.show_progress = FALSE)

LOG_FILE <- "onpe_extrapolation_output.txt"
writeLines("", LOG_FILE)

msg <- function(...) {
  txt <- paste0(...)
  cat(txt, "\n")
  clean <- gsub("\033\\[[0-9;]*[A-Za-z]", "", txt)
  clean <- gsub("[^[:print:]\n\t]", "", clean)
  con <- file(LOG_FILE, open = "a")
  writeLines(clean, con)
  close(con)
}

# --- Config -------------------------------------------------------------------

candidates <- c("Fujimori", "Lopez_Aliaga", "Nieto", "Belmont", "Sanchez")

labels <- c(
  Fujimori     = "Keiko Fujimori (Fuerza Popular)",
  Lopez_Aliaga = "Rafael Lopez Aliaga (Renovacion Popular)",
  Nieto        = "Jorge Nieto (Buen Gobierno)",
  Belmont      = "Ricardo Belmont (Civico Obras)",
  Sanchez      = "Roberto Sanchez (Juntos por el Peru)"
)

short <- c(Fujimori = "FUJ", Lopez_Aliaga = "RLA", Nieto = "NTO",
           Belmont = "BLM", Sanchez = "RSP")

# ==============================================================================
# LOAD DATA
# ==============================================================================

#IMPORTANT! CHANGE CSV FILE IF UPDATED
CSV_FILE <- "onpe_2026_complete_20260416T1209.csv"

msg(paste(rep("=", 70), collapse = ""))
msg("  LOADING BROWSER-FETCHED DATA")
msg(paste(rep("=", 70), collapse = ""))

raw <- read_csv(CSV_FILE, col_types = cols(.default = "c"))

num_cols <- c("pct_actas", "actas_contabilizadas", "total_actas",
              "votos_Fujimori", "pct_Fujimori",
              "votos_Lopez_Aliaga", "pct_Lopez_Aliaga",
              "votos_Nieto", "pct_Nieto",
              "votos_Belmont", "pct_Belmont",
              "votos_Sanchez", "pct_Sanchez")
for (col in num_cols) {
  if (col %in% names(raw)) raw[[col]] <- as.numeric(raw[[col]])
}
raw$pct_actas[is.na(raw$pct_actas)] <- 0

msg(sprintf("Total rows: %d", nrow(raw)))

df  <- raw[raw$type == "domestic", ]
ext <- raw[raw$type == "extranjero", ]

msg(sprintf("Domestic districts: %d (with data: %d, zero actas: %d)",
            nrow(df), sum(df$pct_actas > 0), sum(df$pct_actas == 0)))
msg(sprintf("Extranjero cities:  %d (with data: %d, zero: %d)",
            nrow(ext), sum(ext$pct_actas > 0), sum(ext$pct_actas == 0)))

# ==============================================================================
# PART A: EXTRANJERO EXTRAPOLATION (city-level, hierarchical imputation)
# ==============================================================================

msg("\n", paste(rep("=", 70), collapse = ""))
msg("  PART A: EXTRANJERO EXTRAPOLATION")
msg("  Hierarchy: direct > country avg > continent avg")
msg(paste(rep("=", 70), collapse = ""))

# In the new CSV:
#   prov_code  = country code (e.g., 920200 for Argentina)
#   dept_code  = continent code (e.g., 920000 for AMERICA)
#   provincia  = country name
#   departamento = continent name (EXT-AMERICA, etc.)
ext$country_code   <- ext$prov_code
ext$continent_code <- ext$dept_code
ext$continent      <- gsub("^EXT-", "", ext$departamento)

# --- Estimate valid votes per city (robust: try all candidates) ---------------

est_valid_row <- function(row_df, i) {
  for (c in candidates) {
    v <- row_df[[paste0("votos_", c)]][i]
    p <- row_df[[paste0("pct_", c)]][i]
    if (!is.na(v) && !is.na(p) && p > 0) return(v / (p / 100))
  }
  return(0)
}

ext$est_valid <- sapply(seq_len(nrow(ext)), function(i) est_valid_row(ext, i))

# --- Compute averages at country and continent level -------------------------
# compute_avg: from a subset of rows, compute pooled VPA and vote shares

compute_avg <- function(sub) {
  has <- sub$pct_actas > 0 & !is.na(sub$pct_actas)
  if (!any(has)) return(NULL)
  tv <- sum(sub$est_valid[has], na.rm = TRUE)
  ta <- sum(sub$actas_contabilizadas[has], na.rm = TRUE)
  if (ta == 0) return(NULL)
  vpa <- tv / ta
  cv <- sapply(candidates, function(c)
    sum(sub[[paste0("votos_", c)]][has], na.rm = TRUE))
  tc <- sum(cv)
  if (tc == 0) return(NULL)
  return(list(vpa = vpa, shares = cv / tc, n_cities = sum(has), n_actas = ta))
}

# Country-level averages
country_avgs <- list()
for (ctry in unique(ext$country_code)) {
  avg <- compute_avg(ext[ext$country_code == ctry, ])
  if (!is.null(avg)) country_avgs[[ctry]] <- avg
}

# Continent-level averages
continent_avgs <- list()
for (cont in unique(ext$continent)) {
  avg <- compute_avg(ext[ext$continent == cont, ])
  if (!is.null(avg)) continent_avgs[[cont]] <- avg
}

# --- Log averages ---
msg("\n--- Country averages (countries with data) ---")
for (ctry in names(country_avgs)) {
  avg <- country_avgs[[ctry]]
  cname <- ext$provincia[ext$country_code == ctry][1]
  msg(sprintf("  %s [%s]: vpa=%.1f, %d cities, %d actas",
              cname, ctry, avg$vpa, avg$n_cities, avg$n_actas))
  for (c in candidates) {
    msg(sprintf("    %-4s: %.2f%%", short[c], avg$shares[c] * 100))
  }
}

msg("\n--- Continent averages ---")
for (cont in names(continent_avgs)) {
  avg <- continent_avgs[[cont]]
  msg(sprintf("  %s: vpa=%.1f, %d cities, %d actas",
              cont, avg$vpa, avg$n_cities, avg$n_actas))
  for (c in candidates) {
    msg(sprintf("    %-4s: %.2f%%", short[c], avg$shares[c] * 100))
  }
}

# --- Extrapolate each city ----------------------------------------------------

for (c in candidates) ext[[paste0("extrap_", c)]] <- 0
ext$method <- ""

for (i in seq_len(nrow(ext))) {
  pct  <- ext$pct_actas[i]
  ta   <- ext$total_actas[i]; if (is.na(ta)) ta <- 0
  ctry <- ext$country_code[i]
  cont <- ext$continent[i]

  if (!is.na(pct) && pct > 0) {
    # DIRECT: scale counted votes to 100%
    for (c in candidates) {
      v <- ext[[paste0("votos_", c)]][i]; if (is.na(v)) v <- 0
      ext[[paste0("extrap_", c)]][i] <- v * (100 / pct)
    }
    ext$method[i] <- "direct"

  } else if (ta > 0 && ctry %in% names(country_avgs)) {
    # IMPUTE FROM COUNTRY: use other cities in same country
    avg <- country_avgs[[ctry]]
    ev <- avg$vpa * ta
    for (c in candidates) ext[[paste0("extrap_", c)]][i] <- ev * avg$shares[c]
    ext$method[i] <- "imputed_country"

  } else if (ta > 0 && cont %in% names(continent_avgs)) {
    # IMPUTE FROM CONTINENT: fallback when country has no data
    avg <- continent_avgs[[cont]]
    ev <- avg$vpa * ta
    for (c in candidates) ext[[paste0("extrap_", c)]][i] <- ev * avg$shares[c]
    ext$method[i] <- "imputed_continent"

  } else {
    ext$method[i] <- "skipped"
  }
}

# --- Log per-city results ---
msg("\n--- Extranjero extrapolated (by city) ---")

# Print grouped by country for readability
for (ctry in unique(ext$country_code)) {
  sub <- ext[ext$country_code == ctry, ]
  cname <- sub$provincia[1]
  cont  <- sub$continent[1]
  msg(sprintf("\n  %s (%s):", cname, cont))
  for (j in seq_len(nrow(sub))) {
    msg(sprintf("    %-20s [%-18s] FUJ:%7s RLA:%7s NTO:%7s BLM:%7s RSP:%7s",
                sub$distrito[j], sub$method[j],
                format(round(sub$extrap_Fujimori[j]), big.mark = ","),
                format(round(sub$extrap_Lopez_Aliaga[j]), big.mark = ","),
                format(round(sub$extrap_Nieto[j]), big.mark = ","),
                format(round(sub$extrap_Belmont[j]), big.mark = ","),
                format(round(sub$extrap_Sanchez[j]), big.mark = ",")))
  }
}

# --- Extranjero totals ---
ext_totals <- sapply(candidates, function(c)
  sum(ext[[paste0("extrap_", c)]], na.rm = TRUE))

msg("\n  EXTRANJERO TOTAL:")
for (c in candidates[order(-ext_totals)]) {
  msg(sprintf("    %-45s %10s", labels[c], format(round(ext_totals[c]), big.mark = ",")))
}

# Method counts
msg(sprintf("\n  Methods: direct=%d, imputed_country=%d, imputed_continent=%d, skipped=%d",
            sum(ext$method == "direct"),
            sum(ext$method == "imputed_country"),
            sum(ext$method == "imputed_continent"),
            sum(ext$method == "skipped")))

# --- Total valid votes extranjero ---
ext_total_valid <- 0
for (i in seq_len(nrow(ext))) {
  pct <- ext$pct_actas[i]; ta <- ext$total_actas[i]
  ev <- ext$est_valid[i]
  if (!is.na(pct) && pct > 0 && ev > 0) {
    ext_total_valid <- ext_total_valid + ev * (100 / pct)
  } else if (!is.na(ta) && ta > 0) {
    ctry <- ext$country_code[i]
    cont <- ext$continent[i]
    # Use same hierarchy as imputation
    vpa <- if (ctry %in% names(country_avgs)) country_avgs[[ctry]]$vpa
           else if (cont %in% names(continent_avgs)) continent_avgs[[cont]]$vpa
           else 0
    ext_total_valid <- ext_total_valid + vpa * ta
  }
}

msg(sprintf("\n  Est. total valid votes (extranjero): %s",
            format(round(ext_total_valid), big.mark = ",")))

# ==============================================================================
# PART B: DOMESTIC DISTRICT EXTRAPOLATION (unchanged from v1)
# ==============================================================================

msg("\n", paste(rep("=", 70), collapse = ""))
msg("  PART B: DOMESTIC DISTRICT EXTRAPOLATION")
msg(paste(rep("=", 70), collapse = ""))

msg(sprintf("Loaded %d districts", nrow(df)))
msg(sprintf("  With data:  %d", sum(df$pct_actas > 0)))
msg(sprintf("  Zero actas: %d", sum(df$pct_actas == 0)))

# --- Flag big-city districts --------------------------------------------------

df$is_big_city <- FALSE
for (prov in unique(df$prov_code)) {
  idx <- which(df$prov_code == prov & df$pct_actas > 0)
  if (length(idx) >= 4) {
    threshold <- quantile(df$total_actas[idx], 0.75, na.rm = TRUE)
    df$is_big_city[idx[df$total_actas[idx] >= threshold]] <- TRUE
  } else if (length(idx) >= 2) {
    df$is_big_city[idx[which.max(df$total_actas[idx])]] <- TRUE
  }
}
msg(sprintf("Big-city districts excluded from imputation: %d", sum(df$is_big_city)))

# --- Estimate valid votes per district ----------------------------------------

estimate_valid <- function(v, p) {
  ifelse(!is.na(p) & p > 0 & !is.na(v), v / (p / 100), 0)
}
df$est_valid <- estimate_valid(df$votos_Fujimori, df$pct_Fujimori)

# --- Province rates (excl big cities) -----------------------------------------

impute_pool <- df[df$pct_actas > 0 & !df$is_big_city, ]

prov_rates <- list()
for (prov in unique(impute_pool$prov_code)) {
  sub <- impute_pool[impute_pool$prov_code == prov, ]
  tv <- sum(sub$est_valid, na.rm = TRUE)
  ta <- sum(sub$actas_contabilizadas, na.rm = TRUE)
  cv <- sapply(candidates, function(c) sum(sub[[paste0("votos_", c)]], na.rm = TRUE))
  tc <- sum(cv)
  if (ta > 0 && tc > 0) prov_rates[[prov]] <- list(vpa = tv / ta, shares = cv / tc)
}

# --- Department rates (excl big cities) ---------------------------------------

dept_rates <- list()
for (dept in unique(impute_pool$dept_code)) {
  sub <- impute_pool[impute_pool$dept_code == dept, ]
  tv <- sum(sub$est_valid, na.rm = TRUE)
  ta <- sum(sub$actas_contabilizadas, na.rm = TRUE)
  cv <- sapply(candidates, function(c) sum(sub[[paste0("votos_", c)]], na.rm = TRUE))
  tc <- sum(cv)
  if (ta > 0 && tc > 0) dept_rates[[dept]] <- list(vpa = tv / ta, shares = cv / tc)
}

msg(sprintf("Provinces with rates: %d", length(prov_rates)))
msg(sprintf("Departments with rates: %d", length(dept_rates)))

# --- Extrapolate each district ------------------------------------------------

for (c in candidates) df[[paste0("extrap_", c)]] <- 0
df$method <- ""
counts <- c(direct = 0, province = 0, department = 0, skipped = 0)

for (i in seq_len(nrow(df))) {
  pct  <- df$pct_actas[i]
  prov <- df$prov_code[i]
  dept <- df$dept_code[i]
  ta   <- df$total_actas[i]; if (is.na(ta)) ta <- 0

  if (!is.na(pct) && pct > 0) {
    for (c in candidates) {
      v <- df[[paste0("votos_", c)]][i]; if (is.na(v)) v <- 0
      df[[paste0("extrap_", c)]][i] <- v * (100 / pct)
    }
    df$method[i] <- "direct"; counts["direct"] <- counts["direct"] + 1

  } else if (ta > 0 && prov %in% names(prov_rates)) {
    ev <- prov_rates[[prov]]$vpa * ta
    for (c in candidates) df[[paste0("extrap_", c)]][i] <- ev * prov_rates[[prov]]$shares[c]
    df$method[i] <- "province"; counts["province"] <- counts["province"] + 1

  } else if (ta > 0 && dept %in% names(dept_rates)) {
    ev <- dept_rates[[dept]]$vpa * ta
    for (c in candidates) df[[paste0("extrap_", c)]][i] <- ev * dept_rates[[dept]]$shares[c]
    df$method[i] <- "department"; counts["department"] <- counts["department"] + 1

  } else {
    df$method[i] <- "skipped"; counts["skipped"] <- counts["skipped"] + 1
  }
}

msg(sprintf("\n  Direct:     %d", counts["direct"]))
msg(sprintf("  Province:   %d", counts["province"]))
msg(sprintf("  Department: %d", counts["department"]))
msg(sprintf("  Skipped:    %d", counts["skipped"]))

# ==============================================================================
# PART C: COMBINE DOMESTIC + EXTRANJERO
# ==============================================================================

dom_totals <- sapply(candidates, function(c)
  sum(df[[paste0("extrap_", c)]], na.rm = TRUE))

national <- dom_totals + ext_totals

# Total valid (all candidates)
total_valid_domestic <- 0
for (dept in unique(df$dept_code)) {
  sub <- df[df$dept_code == dept, ]
  sub_d <- sub[sub$pct_actas > 0, ]
  if (nrow(sub_d) > 0) {
    cv <- sum(sub_d$est_valid, na.rm = TRUE)
    ca <- sum(sub_d$actas_contabilizadas, na.rm = TRUE)
    ta <- sum(sub$total_actas, na.rm = TRUE)
    if (ca > 0) total_valid_domestic <- total_valid_domestic + cv * (ta / ca)
  }
}
total_valid <- total_valid_domestic + ext_total_valid
pct_national <- national / total_valid * 100
sorted_cands <- candidates[order(-national)]

# ==============================================================================
# PART D: PRINT RESULTS
# ==============================================================================

msg("\n", paste(rep("=", 70), collapse = ""))
msg("  ONPE 2026 - EXTRAPOLATED PRESIDENTIAL RESULTS")
msg("  City-level extranjero, hierarchical imputation (country > continent)")
msg(paste(rep("=", 70), collapse = ""))

msg(sprintf("\n  %-45s %15s %10s", "Candidate", "Extrap.Votes", "% Valid"))
msg(paste(rep("-", 72), collapse = ""))
for (c in sorted_cands) {
  msg(sprintf("  %-45s %13s %9.2f%%",
              labels[c], format(round(national[c]), big.mark = ","), pct_national[c]))
  msg(sprintf("    (domestic: %s + extranjero: %s)",
              format(round(dom_totals[c]), big.mark = ","),
              format(round(ext_totals[c]), big.mark = ",")))
}
msg(sprintf("\n  Est. total valid votes: %s (domestic: %s + ext: %s)",
            format(round(total_valid), big.mark = ","),
            format(round(total_valid_domestic), big.mark = ","),
            format(round(ext_total_valid), big.mark = ",")))

# --- Segunda vuelta -----------------------------------------------------------

msg("\n", paste(rep("=", 70), collapse = ""))
msg("  SEGUNDA VUELTA PROJECTION")
msg(paste(rep("=", 70), collapse = ""))
msg(sprintf("\n  1st: %s", labels[sorted_cands[1]]))
msg(sprintf("       %s votes (%.2f%%)",
            format(round(national[sorted_cands[1]]), big.mark = ","), pct_national[sorted_cands[1]]))
msg(sprintf("\n  2nd: %s", labels[sorted_cands[2]]))
msg(sprintf("       %s votes (%.2f%%)",
            format(round(national[sorted_cands[2]]), big.mark = ","), pct_national[sorted_cands[2]]))
gap23 <- national[sorted_cands[2]] - national[sorted_cands[3]]
msg(sprintf("\n  Margin 2nd-3rd: %s votes (%.2f pp)",
            format(round(gap23), big.mark = ","), gap23 / total_valid * 100))
msg(sprintf("  3rd: %s (%s votes, %.2f%%)",
            labels[sorted_cands[3]],
            format(round(national[sorted_cands[3]]), big.mark = ","), pct_national[sorted_cands[3]]))

# --- Breakdown: departments + extranjero by country --------------------------

msg("\n", paste(rep("=", 70), collapse = ""))
msg("  BREAKDOWN BY DEPARTMENT")
msg(paste(rep("=", 70), collapse = ""))

dept_extrap <- list()
for (dept in sort(unique(df$departamento))) {
  sub <- df[df$departamento == dept, ]
  vals <- sapply(candidates, function(c) sum(sub[[paste0("extrap_", c)]], na.rm = TRUE))
  dept_extrap[[dept]] <- vals
}

msg(sprintf("\n  %-20s %-4s %10s %10s %10s %10s %10s",
            "Region", "Win", "FUJ", "RLA", "NTO", "BLM", "RSP"))
msg(paste(rep("-", 78), collapse = ""))
for (nm in names(dept_extrap)) {
  d <- dept_extrap[[nm]]
  winner <- names(which.max(d))
  msg(sprintf("  %-20s %-4s %10s %10s %10s %10s %10s",
              nm, short[winner],
              format(round(d["Fujimori"]), big.mark = ","),
              format(round(d["Lopez_Aliaga"]), big.mark = ","),
              format(round(d["Nieto"]), big.mark = ","),
              format(round(d["Belmont"]), big.mark = ","),
              format(round(d["Sanchez"]), big.mark = ",")))
}

# --- Breakdown: extranjero by country within continent -----------------------

msg("\n", paste(rep("=", 70), collapse = ""))
msg("  BREAKDOWN BY COUNTRY (EXTRANJERO)")
msg(paste(rep("=", 70), collapse = ""))

msg(sprintf("\n  %-6s %-25s %-4s %10s %10s %10s %10s %10s",
            "Cont.", "Country", "Win", "FUJ", "RLA", "NTO", "BLM", "RSP"))
msg(paste(rep("-", 100), collapse = ""))

for (cont in sort(unique(ext$continent))) {
  cont_sub <- ext[ext$continent == cont, ]
  for (ctry in sort(unique(cont_sub$country_code))) {
    sub <- cont_sub[cont_sub$country_code == ctry, ]
    cname <- sub$provincia[1]
    vals <- sapply(candidates, function(c) sum(sub[[paste0("extrap_", c)]], na.rm = TRUE))
    tv <- sum(vals)
    if (tv == 0) next
    winner <- names(which.max(vals))
    msg(sprintf("  %-6s %-25s %-4s %10s %10s %10s %10s %10s",
                cont, cname, short[winner],
                format(round(vals["Fujimori"]), big.mark = ","),
                format(round(vals["Lopez_Aliaga"]), big.mark = ","),
                format(round(vals["Nieto"]), big.mark = ","),
                format(round(vals["Belmont"]), big.mark = ","),
                format(round(vals["Sanchez"]), big.mark = ",")))
  }
  # Continent subtotal
  vals <- sapply(candidates, function(c)
    sum(cont_sub[[paste0("extrap_", c)]], na.rm = TRUE))
  winner <- names(which.max(vals))
  msg(sprintf("  %-6s %-25s %-4s %10s %10s %10s %10s %10s",
              "", paste0("--- ", cont, " TOTAL ---"), short[winner],
              format(round(vals["Fujimori"]), big.mark = ","),
              format(round(vals["Lopez_Aliaga"]), big.mark = ","),
              format(round(vals["Nieto"]), big.mark = ","),
              format(round(vals["Belmont"]), big.mark = ","),
              format(round(vals["Sanchez"]), big.mark = ",")))
  msg("")
}

# ==============================================================================
# PART E: SAVE
# ==============================================================================

ext_out <- data.frame(
  departamento = ext$departamento,
  provincia    = ext$provincia,
  distrito     = ext$distrito,
  pct_actas    = ext$pct_actas,
  total_actas  = ext$total_actas,
  method       = ext$method,
  is_big_city  = FALSE,
  stringsAsFactors = FALSE
)
for (c in candidates) {
  ext_out[[paste0("extrap_", c)]] <- ext[[paste0("extrap_", c)]]
}

out_cols <- c("departamento", "provincia", "distrito", "pct_actas", "total_actas",
              "method", "is_big_city", paste0("extrap_", candidates))

df_out <- rbind(df[, out_cols], ext_out[, out_cols])

write_csv(df_out, "onpe_2026_extrapolated_full.csv")
msg("\nSaved to onpe_2026_extrapolated_full.csv")
msg("Done.")
