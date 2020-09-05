COVID-19 Preprints
================

# Background

This file contains code used to harvest metadata of COVID-19 related
preprints.

Currently, data are harvested from three sources:

1.  Crossref (using the
    [rcrossref](https://github.com/ropensci/rcrossref) package)
2.  DataCite (using the
    [rdatacite](https://github.com/ropensci/rcrossref) package)
3.  arXiv (using the [aRxiv](https://github.com/ropensci/aRxiv) package)
4.  RePEc (using the [oai](https://github.com/ropensci/oai) package)

A description of the methods for harvesting data sources is provided in
each relevant section below.

# Load required packages

``` r
# clear previous output
rm(list=ls())

library(aRxiv)
library(lubridate)
library(oai)
library(ratelimitr)
library(rcrossref)
library(rdatacite)
library(tidyverse)
library(rvest)
library(jsonlite)
```

# Set the latest sample date

``` r
sample_date <- "2020-08-30"
```

# Crossref

Harvesting of Crossref metadata is carried out using the
[rcrossref](https://github.com/ropensci/rcrossref) package for R. In
general, preprints are indexed in Crossref with the ‘type’ field given a
value of ‘posted-content’. The `cr_types_` function can therefore be
used to retrieve all metadata related to records with the type of
‘posted-content’, filtered for the dates of this analysis
(i.e. 2020-01-01 until present). Note that here, the ‘low level’
`cr_types_` function is used to return all metadata in list format, as
this also includes some fields (e.g. abstract) that are not returned by
the ‘high level’ `cr_types` function.

``` r
# Query posted content
cr_posted_content <- cr_types_(types = "posted-content",
                               works = TRUE, 
                               filter = c(from_posted_date = "2020-01-01", 
                                          until_posted_date = sample_date),
                               limit = 1000, 
                               cursor = "*",
                               parse = TRUE,
                               cursor_max = 1000000)
```

Relevant preprint metadata fields are parsed from the list format
returned in the previous step, to a more manageable data frame. Note the
‘institution’, ‘publisher’ and ‘group-title’ fields are retained, to be
later used to match preprints to specific preprint repositories.

``` r
# Function to parse posted "date parts" to more useful YYYY-MM-DD format
parseCrossrefPostedDate <- function(posted) {
  if(length(posted$`date-parts`[[1]]) == 3) {
    ymd(paste0(sprintf("%02d", unlist(posted$`date-parts`)), collapse = "-"))
  } else {
    NA
  }
}

# Function to parse Crossref preprint data to data frame
parseCrossrefPreprints <- function(item) {
  tibble(
    institution = if(length(item$institution$name)) item$institution$name else NA_character_,
    publisher = item$publisher,
    group_title = if(length(item$`group-title`)) item$`group-title` else NA_character_,
    cr_member_id = item$member,
    identifier = item$DOI,
    identifier_type = "DOI",
    title = item$title[[1]],
    # For posted-content, use the 'posted' date fields for the relevant date. 
    # For SSRN preprints, use the 'created' date
    posted_date = if(length(item$posted)) parseCrossrefPostedDate(item$posted) else parseCrossrefPostedDate(item$created),
    abstract = if(length(item$abstract)) item$abstract else NA
  )
}

# Iterate over posted-content list and build data frame
cr_posted_content_df <- map_dfr(cr_posted_content, 
                                ~ map_df(.$message$items, parseCrossrefPreprints))

rm(cr_posted_content)
```

In the final step, preprints are subsetted to include only those related
to COVID-19, and the respective preprint repository of each identified.

``` r
# Build a search string containing terms related to COVID-19
search_string <- "coronavirus|covid-19|sars-cov|ncov-2019|2019-ncov|hcov-19|sars-2"

cr_posted_content_covid <- cr_posted_content_df %>%
  # Filter COVID-19 related preprints
  filter(str_detect(title, regex(search_string, ignore_case = TRUE)) | 
         str_detect(abstract, regex(search_string, ignore_case = TRUE))) %>%
  # Rule-based matching of preprints to repositories. For CSHL repositories, the
  # repository name (bioRxiv/medRxiv) is contained in the 'institution' field. For
  # others we can use the 'publisher' field, except for any preprint servers 
  # hosted on OSF in which we should use the 'group_title' field to ensure we get
  # the right repository.
  mutate(source = case_when(
    institution == "bioRxiv" ~ "bioRxiv",
    institution == "medRxiv" ~ "medRxiv",
    publisher == "Research Square" ~ "Research Square",
    publisher == "MDPI AG" ~ "Preprints.org",
    publisher == "American Chemical Society (ACS)" ~ "ChemRxiv",
    publisher == "JMIR Publications Inc." ~ "JMIR",
    publisher == "WHO Press" ~ "WHO",
    publisher == "ScienceOpen" ~ "ScienceOpen",
    publisher == "SAGE Publications" ~ "SAGE",
    publisher == "FapUNIFESP (SciELO)" ~ "SciELO",
    publisher == "Institute of Electrical and Electronics Engineers (IEEE)" ~ "Techrxiv (IEEE)",
    publisher == "Authorea, Inc." ~ "Authorea",
    group_title == "PsyArXiv" ~ "PsyArXiv (OSF)",
    group_title == "NutriXiv" ~ "NutriXiv (OSF)",
    group_title == "SocArXiv" ~ "SocArXiv (OSF)",
    group_title == "EdArXiv" ~ "EdArXiv (OSF)",
    group_title == "MediArXiv" ~ "MediArXiv (OSF)",
    group_title == "AfricArXiv" ~ "AfricArXiv (OSF)",
    group_title == "EarthArXiv" ~ "EarthArXiv (OSF)",
    group_title == "IndiaRxiv" ~ "IndiaRxiv (OSF)",
    group_title == "EcoEvoRxiv" ~ "EcoEvoRxiv (OSF)",
    group_title == "INA-Rxiv" ~ "INA-Rxiv (OSF)",
    group_title == "MetaArxiV" ~ "MetaArXiV (OSF)",
    group_title == "engrXiv" ~ "engrXiv (OSF)",
    group_title == "SportRxiv" ~ "SportRxiv (OSF)",
    group_title == "LawArXiv" ~ "LawArXiv (OSF)",
    group_title == "Frenxiv" ~ "Frenxiv (OSF)",
    group_title == "MetaArXiV" ~ "MetaArXiV (OSF)",
    group_title == "AgriXiv" ~ "AgriXiv (OSF)",
    group_title == "BioHackrXiv" ~ "BioHackrXiv (OSF)",
    group_title == "Open Science Framework" ~ "OSF Preprints"
  )) %>%
  # Remove those that could not be unambiguously matched or do not seem to be
  # "true" preprints
  filter(!is.na(source)) %>%
  # Some preprints have multiple DOI records relating to multiple preprint
  # versions (mainly in ChemRxiv and Preprints.org). In these cases the DOI 
  # is usually appended with a version number, e.g. 10.1000/12345.v2. To ensure
  # only a single record is counted per preprint, the version number is
  # removed and only the earliest DOI record is kept
  mutate(doi_clean = str_replace(identifier, "\\.v.*|\\/v.*", "")) %>%
  group_by(doi_clean) %>%
  arrange(posted_date) %>%
  slice(1) %>%
  ungroup() %>%
  # Additionally filter preprints with the same title posted on the same server
  group_by(source, title) %>%
  arrange(posted_date) %>%
  slice(1) %>%
  ungroup() %>%
  # Select only relevant fields with unique values
  distinct(source, identifier, identifier_type, posted_date, title, abstract)
```

A side effect of the above procedure is that some preprint servers, most
notably SSRN, instead index their content with the ‘type’ set to
‘journal-article’, and are thus not included when querying only for
‘posted-content’ types. Metadata of SSRN preprints are thus instead
harvested using the `cr_works_` function for the ISSN of SSRN
(1556-5068).

``` r
# Query SSRN preprints
cr_ssrn <- cr_works_(filter = c(issn = "1556-5068",
                               from_created_date = "2020-01-01", 
                               until_created_date = sample_date), 
                    limit = 1000,
                    cursor = "*",
                    parse = TRUE,
                    cursor_max = 1000000)

# Iterate over SSRN list and build data frame
cr_ssrn_df <- map_df(cr_ssrn, ~ map_dfr(.x$message$items, parseCrossrefPreprints))

rm(cr_ssrn)
```

An inspection of the published dates of SSRN preprints indicates some
abnormalities, e.g. on 24th March 2020, more than 5000 SSRN preprints
were published according to dates from Crossref - the next highest day
only has \~250 published preprints. Manual inspection of a small number
suggests that the published date in Crossref does not correspond well to
the actual published date according to the SSRN website. Thus, we can
subset our set of SSRN preprints to those related to COVID-19 (to reduce
the number of articles), and harvest more accurate publication dates by
directly crawling the SSRN website (using the
[rvest](https://github.com/tidyverse/rvest) package).

``` r
getSSRNPublicationDate <- function(doi) {
  
  # Base URL for querying
  base_url <- "https://doi.org/"
  url <- paste0(base_url, doi)
  
  posted_date <- tryCatch({
    # Read page URL and select relevant node
    d <- read_html(url) %>%
      html_nodes("meta[name='citation_online_date'][content]") %>%
      html_attr('content')
    # Sometimes the doi resolves to an empty page - in these cases return NA
    if(length(d)) d else NA
  },
  error = function(e) {
    NA
  })
  
  return(posted_date)
}

# Create the final SSRN dataset
cr_ssrn_covid <- cr_ssrn_df %>%  
  # Filter COVID-19 related preprints. SSRN metadata does not contain abstracts
  filter(str_detect(title, regex(search_string, ignore_case = TRUE))) %>%
  # Retrieve 'real' posted dates from the SSRN website. Warning: slow
  mutate(posted_date = ymd(map_chr(identifier, getSSRNPublicationDate)),
         source = "SSRN") %>%
  # Keep only the first preprint where multiple preprints exist with the same title
  group_by(source, title) %>%
  arrange(posted_date) %>%
  slice(1) %>%
  ungroup() %>%
  distinct(source, identifier, identifier_type, posted_date, title, abstract)
```

The datasets derived from “posted-content” and from SSRN are merged to a
final Crossref dataset

``` r
cr_covid <- bind_rows(cr_posted_content_covid, cr_ssrn_covid)
```

# DataCite

For harvesting of Datacite metadata the
[rdatacite](https://github.com/ropensci/rcrossref) package for R was
used. In general, preprints are indexed in Datacite with the
‘resourceType’ field set to ‘Preprint’. This field is not strictly
controlled though, so not all preprints will be caught this way.

Results can be filtered on date of creation (year only), which works
fine for 2020 (but will need adaptation afterwards). Pagination is used
to get all records.

``` r
#define function to query Datacite API
getDataCite <- function(n){
  res <- dc_dois(query = "types.resourceType:Preprint", 
                 created = "2020",
                 limit = 1000,
                 page = n)
}  


#define function to add progress bar
getDataCite_progress <- function(n){
  pb$tick()$print()
  res <- getDataCite(n)
  
  return(res)
}

#initial query to get number of results for pagination
dois <- dc_dois(query = "types.resourceType:Preprint", 
                created = "2020",
                limit = 1)
total <- dois$meta$total

#create pagination sequence
seq <- c(1:ceiling(total/1000))

#set counter for progress bar
pb <- progress_estimated(length(seq))

#get datacite results
dc_preprints <- map(seq, getDataCite_progress)

rm(pb)
```

Next, relevant preprint metadata fields are parsed from the list format
returned in the previous step, to a more manageable data frame. Note
that specific preprint repositories are encoded in the field ‘client’,
and abstracts are included in the field ‘descriptions’. The resulting
columns ‘title’ and ‘descriptions’ are list columns and need to be
processed further to extract the needed information.

``` r
parseDataCiteDescription <- function(x) {
  if(length(x) > 0) {
    if(x$descriptionType == "Abstract") {
      return(str_to_sentence(str_c(x$description, collapse = "; ")))
    } else {
      return(NA_character_)
    }
  } else {
    return(NA_character_)
  }
}

parseDataCitePreprints <- function(item) {
  tibble(
    identifier = item$data$attributes$doi,
    identifier_type = "DOI",
    posted_date = as.Date(item$data$attributes$created),
    client = item$data$relationships$client$data$id,
    title = map_chr(item$data$attributes$titles, 
                    ~ str_to_sentence(str_c(.x$title, collapse = "; "))),
    abstract = map_chr(item$data$attributes$descriptions, 
                          function(x) parseDataCiteDescription(x)))
}

dc_preprints_df <- map_df(dc_preprints, parseDataCitePreprints)

rm(dc_preprints)
```

DataCite preprints are then subsetted to include only those related to
COVID-19, and the respective preprint repository of each identified.

``` r
dc_covid <- dc_preprints_df %>%
  # Filter COVID-19 related preprints
  filter(str_detect(title, regex(search_string, ignore_case = TRUE)) | 
         str_detect(abstract, regex(search_string, ignore_case = TRUE))) %>%
  # Rule-based matching of preprints to repositories.
  # Repository names are encoded in field 'client.
  mutate(source = case_when(
    client == "rg.rg" ~ "ResearchGate",
    client == "figshare.ars" ~ "Figshare",
    client == "cern.zenodo" ~ "Zenodo",
    TRUE ~ NA_character_)) %>%
  # Remove those that could not be unambiguously matched
  filter(!is.na(source)) %>%
  # Some preprints have multiple DOI records relating to multiple preprint
  # versions. It differs how this is reflected in the DOIs. 
  # In Figshare and ResearchGate, the DOI is appended with a version number
  #(e.g.10.1000/12345.v2 and 10.1000/12345/1, respectively)
  # To ensure only a single record is counted per preprint, the version number is
  # removed and only the earliest DOI record is kept
  mutate(doi_clean = str_replace(identifier, "\\.v.*|\\/[0-9]$|\\/[1-9][0-9]$", "")) %>%
  group_by(doi_clean) %>%
  arrange(posted_date) %>%
  slice(1) %>%
  ungroup() %>%
  # Additionally filter preprints with the same title posted on the same server
  # This will also address versioning on Zenodo, where consecutive DOIs are used
  group_by(source, title) %>%
  arrange(posted_date) %>%
  slice(1) %>%
  ungroup() %>%
  # Select only relevant fields with unique values
  distinct(source, identifier, identifier_type, posted_date, title, abstract)
```

# arXiv

ArXiv records are retrieved using the
[aRxiv](https://github.com/ropensci/aRxiv) package. aRxiv provides a
nice search functionality, so that we can search directly for our search
terms in the titles and abstracts of arXiv preprints, and return only
the relevant data

``` r
# For returning details of preprints on arXiv, we can use the aRxiv package and
# define title and abstract search strings
ar_covid <- arxiv_search('ti:coronavirus OR ti:covid OR ti:sars-cov OR ti:ncov-2019 ti:2019-ncov OR ti:hcov-19 OR ti:sars-2 OR abs:coronavirus OR abs:covid OR abs:sars-cov OR abs:ncov-2019 OR abs:2019-ncov OR abs:hcov-19 OR abs:sars-2', limit = 10000) %>% 
  mutate(source = "arXiv",
         identifier = id,
         identifier_type = "arXiv ID",
         posted_date = date(submitted)) %>%
  filter(posted_date >= ymd("2020-01-01")) %>%
  select(source, identifier, identifier_type, posted_date, title, abstract) %>%
  distinct()
```

# RePEc

RePEc is a repository for research papers in Economics, containing
preprints as well as other research outputs (e.g. journal papers, books,
software components). There exist a number of methods for accessing
RePEc metadata, see an overview
[here](https://ideas.repec.org/getdata.html). Here we access RePEc
metadata using the (OAI-PMH)\[<https://www.openarchives.org/pmh/>\]
inteface with the package [oai](https://github.com/ropensci/oai).

``` r
# Get a list of all RePEc records posting between our dates
repec_records <- oai::list_records("http://oai.repec.openlib.org", 
                                   from = "2020-01-01", 
                                   until = sample_date, 
                                   as = "list")
```

Relevant metadata is extracted from each record, and only records with
the “type” field of “preprint” are retained. Records contain either one
or multiple “description” fields, the first of which generally (but not
always) contains the abstract. For now we extract only this first
description field.

``` r
parseRepecPreprints <- function(item) {
    tibble(
      type = if(length(item$metadata$type)) item$metadata$type else NA_character_,
      source = "RePEc",
      identifier = if(length(item$headers$identifier)) str_remove(item$headers$identifier, "oai:") else NA_character_,
      identifier_type = "RePEc handle",
      posted_date = if(length(item$headers$datestamp)) item$headers$datestamp else NA_character_,
      title = if(length(item$metadata$title)) item$metadata$title else NA_character_,
      abstract = if(length(item$metadata$description[1])) item$metadata$description[1] else NA_character_
    )
}

repec_covid <- map_dfr(repec_records, function(x) parseRepecPreprints(x)) %>%
  filter(type == "preprint") %>%
  select(-type) %>%
  filter(str_detect(title, regex(search_string, ignore_case = TRUE)) | 
         str_detect(abstract, regex(search_string, ignore_case = TRUE))) %>%
  mutate(posted_date = ymd(posted_date)) %>%
  # Some duplicates are contained - here we group by title and select the earliest record
  group_by(title) %>%
  arrange(posted_date) %>%
  slice(1) %>%
  ungroup() %>%
  # Select only relevant fields with unique values
  distinct(source, identifier, identifier_type, posted_date, title, abstract)

rm(repec_records)
```

# Create final dataset ((bind Crossref, DataCite, arXiv and RePEc data)

``` r
covid_preprints <- bind_rows(cr_covid, dc_covid, ar_covid, repec_covid) %>%
  select(source, identifier, identifier_type, posted_date, title, abstract) %>%
  filter(posted_date <= ymd(sample_date))
  
#clean abstracts (remove jats-tags, remove leading string "Abstract" and similar)
covid_preprints <- covid_preprints %>%
  mutate(abstract = str_remove_all(abstract, "<jats:title>.*?</jats:title>")) %>%
  mutate(abstract = str_remove_all(abstract, "<.*?>"))  

covid_preprints %>%
  write_csv("data/covid19_preprints.csv")
```

# Create metadata file (json file with sample date and release date)

``` r
# Set system date as release date
release_date <- Sys.Date()

# Create metadata as list
metadata <- list()

metadata$release_date <- release_date
metadata$sample_date <- sample_date
metadata$url <- "https://github.com/nicholasmfraser/covid19_preprints/blob/master/data/covid19_preprints.csv?raw=true"

# Save as json file
metadata_json <- toJSON(metadata, pretty = TRUE, auto_unbox = TRUE)
write(metadata_json, "data/metadata.json")
```

# Visualizations

``` r
# Default theme options
theme_set(theme_minimal() +
          theme(text = element_text(size = 12),
          axis.text.x = element_text(angle = 90, vjust = 0.5),
          axis.title.x = element_text(margin = margin(20, 0, 0, 0)),
          axis.title.y = element_text(margin = margin(0, 20, 0, 0)),
          legend.key.size = unit(0.5, "cm"),
          legend.text = element_text(size = 8),
          plot.caption = element_text(size = 10, hjust = 0, color = "grey25", 
                                      margin = margin(20, 0, 0, 0))))

# Create a nice color palette
pal_1 <- colorspace::lighten(pals::tol(n = 10), amount = 0.2)
pal_2 <- colorspace::lighten(pals::tol(n = 10), amount = 0.4)
palette <- c(pal_1, pal_2)
```

``` r
# Minimum number of preprints to be included in graphs (otherwise too many
# categories/labels is confusing)
n_min <- 40

# Repositories with < min preprints
other <- covid_preprints %>%
  count(source) %>%
  filter(n < n_min) %>%
  pull(source)

fct_levels <- covid_preprints %>% 
  mutate(source = case_when(
           source %in% other ~ "Other*",
           T ~ source
        )) %>%
  count(source) %>%
  arrange(desc(n)) %>%
  mutate(source = c(source[source != "Other*"], "Other*")) %>%
  pull(source)

other_text = paste0("* 'Other' refers to preprint repositories  containing <",
                    n_min, " total relevant preprints. These include: ",
                    paste(other, collapse = ", "), ".")

other_caption <- str_wrap(other_text, width = 150)

# Daily preprint counts
covid_preprints %>%
  mutate(source = case_when(
    source %in% other ~ "Other*",
    T ~ source
  ),
  source = factor(source, levels = fct_levels)) %>%
  count(source, posted_date) %>%
  ggplot(aes(x = posted_date, y = n, fill = source)) +
  geom_col() +
  labs(x = "Posted Date", y = "Preprints", fill = "Source",
       title = "COVID-19 preprints per day",
       subtitle = paste0("(up until ", sample_date, ")"),
       caption = other_caption) +
  scale_x_date(date_breaks = "7 days",
               date_minor_breaks = "1 day",
               expand = c(0.01, 0),
               limits = c(ymd("2020-01-13"), ymd(sample_date) + 1)) +
  scale_fill_manual(values = palette) +
  ggsave("outputs/figures/covid19_preprints_day.png", width = 12, height = 6)
```

![](covid19_preprints_files/figure-gfm/unnamed-chunk-19-1.png)<!-- -->

``` r
# Weekly preprint counts
covid_preprints %>%
  mutate(
    source = case_when(
      source %in% other ~ "Other*",
      T ~ source
    ),
    source = factor(source, levels = fct_levels),
    posted_week = ymd(cut(posted_date,
                          breaks = "week",
                          start.on.monday = TRUE))) %>%
  count(source, posted_week) %>%
  ggplot(aes(x = posted_week, y = n, fill = source)) +
  geom_col() +
  labs(x = "Posted Date (week beginning)", y = "Preprints", fill = "Source",
       title = "COVID-19 preprints per week", 
       subtitle = paste0("(up until ", sample_date, ")"),
       caption = other_caption) +
  scale_x_date(date_breaks = "1 week",
               expand = c(0, 0),
               limits = c(ymd("2020-01-13"), ymd(sample_date))) +
  scale_fill_manual(values = palette) +
  ggsave("outputs/figures/covid19_preprints_week.png", width = 12, height = 6)
```

![](covid19_preprints_files/figure-gfm/unnamed-chunk-20-1.png)<!-- -->

``` r
# Cumulative daily preprint counts
covid_preprints %>%
  mutate(source = case_when(
      source %in% other ~ "Other*",
      T ~ source
    ),
    source = factor(source, levels = fct_levels)) %>%
  count(source, posted_date) %>%
  complete(posted_date, nesting(source), fill = list(n = 0)) %>%
  group_by(source) %>%
  arrange(posted_date) %>%
  mutate(cumulative_n = cumsum(n)) %>%
  ggplot() +
  geom_area(aes(x = posted_date, y = cumulative_n, fill = source)) +
  labs(x = "Posted Date", y = "Preprints", fill = "Source",
       title = "COVID-19 preprints (cumulative)", 
       subtitle = paste0("(up until ", sample_date, ")"),
       caption = other_caption) +
  scale_y_continuous(labels = scales::comma) +
  scale_x_date(date_breaks = "1 week",
               expand = c(0.01, 0),
               limits = c(ymd("2020-01-13"), ymd(sample_date) + 1)) +
  scale_fill_manual(values = palette) +
  ggsave("outputs/figures/covid19_preprints_day_cumulative.png", width = 12, height = 6)
```

![](covid19_preprints_files/figure-gfm/unnamed-chunk-21-1.png)<!-- -->
