# Native Status Resolver (NSR)

Author: Brad Boyle (bboyle@email.arizona.edu)  

## Table of Contents

- [Overview](#Overview)
- [Data sources & database](#Data)
- [Components](#Components)
- [Software requirements](#Software)
- [Installation](#Installation)
  - [Database and batch application](#Core)
  - [Web service](#wsi)
- [Usage](#Usage)
  - [Build NSR database](#DB)
  - [Batch applicaton](#Batch)
  - [Batch input format](#Input)
  - [Batch output format](#Output)
  - [Web service](#ws)
- [Native Status Codes](#Native)

## <a name="Overview"></a>Overview

The Native Status Resolver (NSR) determines if a taxon is native or introduced within a political division of observation. The NSR accepts as input one or more observations of a taxon in a political division (country, plus optionally state/province or county/parish). For each observation, the NSR returns an opinion as to whether that taxon is native or introduced in the lowest political division submitted. Native status opinions are determined by consulting one or more species checklists for the region in question. If no checklist is available for the region submitted, the NSR returns no opinion. In some cases, even if no checklist is available for the political division of observation, the NSR may  be able to assign a status of "introduced" if the taxon is endemic to a different, non-overlapping political division.

If the taxon submitted is not present in any checklist for the region of observation, the NSR will also search for opinions for higher taxa, as appropriate. For example, if Poa annua var. supina is submitted, the NSR also checks for opinions for Poa annua and Poa. If a native status opinion is available for a higher taxon and it applies to the lower taxon as well, the NSR will report that opinion. In the case of Poa annua var. supina, a status of "introduced" for Poa annua is also assigned to the variety, but an opinion of "native" is not assigned (i.e., if the species is introduced then all lower taxa must also be introduced, whereas the fact that the species is native to a region does not indicate which variety is native to that region). In other words, introduced status propagates upward in the taxonomic hierarchy and native status propagates downward.

The NSR uses a similar hierarchical approach for political divisions, transferring native status opinions up or down the hierarchy as appropriate. For example, a species that is "introduced" according to a country-level checklist will also be flagged as introduced to a state in that country, even if no checklist is available for that state. Similarly, a species recorded as "native" in a county-level checklist will also be flagged as native to the containing state and country. In other words, introduced status propagates downward in the political division hierarchy and native status propagates upward.

## <a name="Data"></a>Data sources & database

The majority of the checklists consulted by the NSR are high-quality published species lists prepared by professional taxonomists as part of floras or other floristic projects. These checklists are imported and compiled within a PostgreSQL database. Assembly of the NSR database is performed by a separate pipeline of PHP and SQL scripts in subdirectory nsr_db/. See the separate README in this directory for more details.

## <a name="Components"></a>Components

#### nsr_db/nsr_db.php  
- Builds & populates MySQL database used by all NSR services
- Reference data not included 
- See separate README in nsr_db/ for details 

#### nsr.php   
- Core application, evaluates table of observations against reference tables and populates native status opinion columns.  
- Called by nsr_batch.php and nsr_ws.php  

#### nsr_batch.php    
- NSR batch processing application  
- Calls nsr.php  
- Processes multiple observations at once  
- Uploads observations as CSV file from data directory 
- Exports NSR results as TAB delimited file to data directory  
- Requires shell access to this server  

#### nsr_ws.php   
- Simple NSR web service
- Processes on observation per call  
- Calls nsr.php

## <a name="Software"></a>Software requirements
* OS: Runs on a unix or unix-like environment
* PHP 7.0 or greater (may work on earlier versions, but not tested)
* MySQL 5.5 or greater

## <a name="Installation"></a>Installation & setup

The following steps assume two installations: one in public_html for the web service, and a second installation elsewhere in the file system for creating the database and running the batch applications. Other configurations may be used as well.

### <a name="Core"></a>Database and batch application
1. Clone this repository to location of choice, using recursive option to include submodules:

```
git clone --recursive https://github.com/ojalaquellueva/nsr.git
```
2. Set up MySQL database
   * Create empty NSR database.
   * Create admin-level and select-only NSR database users, using user names and passwords of your choice.
3. Copy read-only database config file (db_config-example.php) as db_config.php to location outside the application directory and set the parameters.
4. Copy write-access database config file (db_configw-example.php) as db_configw.php to location outside the application directory and set the parameters.
5. Copy or rename example parameters file (params.example.php) to params.php to same location (inside the main application directory) and set the parameters.
6. Prepare NSR database checklist data sources and set database parameters as described in nsr_db/README.md
7. Build NSR database
8. The following file is used only by the web service and may be removed:

```
rm nsr_ws.php
```

### <a name="wsi"></a>NSR web service

The following instructions assumes:
* NSR database is installed and configured as described above
* Valid virtual host and port have been configured for the API root directory

1. Clone this repository to API root directory of your choice (e.g., /var/www/public_html/nsr/), using recursive option to include submodules:

```
git clone --recursive https://github.com/ojalaquellueva/nsr.git
```
2. Copy read-only database config file (db_config-example.php) as db_config.php to location outside public_html and set the parameters.
3. Copy write-access database config file (db_configw-example.php) as db_configw.php to location outside public_html and set the parameters.
4. Copy or rename parameters file (params.example.php) to params.php and set the parameters.
5. Adjust file permissions per your server settings.
6. The following files, directories and their contents are not used by web service and should be removed:

```
rm -rf nsr_batch_includes/
rm -rf nsr_db/
rm db_batch_connect.php
rm nsr_batch.php
```

## <a name="Usage"></a>Usage

### <a name="DB"></a>Build NSR database

Syntax:  
```
php nsr_db.php

```

See separate README in nsr_db/ for details.

### <a name="Batch"></a>Batch application

Syntax:  
```
php nsr_batch.php -e=<echo> -i=<interactive_mode_on> -f=<inputfile> -l=<line_endings> -t=<inputfile_type> -r=<replace_cache>

```

Options (default in __bold__):  
-e: terminal echo on [__true__,false]  
-i: interactive mode [true,__false__]  
-f: input file name ['__nsr_input.csv__']  
-l: line-endings [unix,__mac__,win]  
-t: inputfile type [__csv__,tab]  
-r: replace the cache [true,__false__]  

Example:  
```
php nsr_batch.php -i=true -f='my_observations.txt' -l=unix -t=tab

```

Notes:  
* Use -r=false to retain all previously cached results. Option -r=true is used only when NSR reference database has changed and previous results may not be valid.  
* When the NSR has finished running, results file will be saved to the NSR data directeory
* Results file has same base name as input file, plus suffix "_nsr_results.txt" 
* Results file is tab-delimitted, regardless of the format of the input file

#### <a name="Input">Batch input format

The NSR accepts as input a plain text file containing one or more observations of taxon in political division, formatted as follows (optional values in square brackets; if county_parish is included, state_province must be included as well):  

taxon,country[,state_province[,county_parish]]  
taxon,country[,state_province[,county_parish]]  
taxon,country[,state_province[,county_parish]]  

Taxon names can be of any of the following ranks: family, genus, species, subspecies, variety, forma. Do not include author.

Spellings of political division names in the NSR database are the plain ascii (unaccented) versions of English-language political division names in Geonames (www.geonames.org). Political division names in user input should therefore be standardized according to the same standard. 

#### <a name="Output">Batch output format

The NSR batch application returns original rows and values as submitted, plus columns indicating whether taxon is native in each level of observation within the political division hierarchy, an overall assessment of native status within the lowest political division of observation, a short explanation of how the decision was reached, and a list of checklist sources consulted.   


| Column	| Meaning (values)
| --------- | -------------------
| native_status_country	| Native status in country (see native status values, below)
| native_status_state_province	| Native status in state_province, if any (see native status values, below)
| native_status_county_parish	| Native status in county_parish, if any (see native status values, below)
| native_status	| Overall native status in lowest declared political division (see native status values, below)
| native_status_reason	| Reason native status was assigned
| native_status_sources	| Checklists used to determine native status
| isIntroduced	| Simplified overall native status (1=introduced;  0=native; blank=status unknown)
| isCultivatedNSR	| Species is known to be cultivated in declared region  (1=cultivated;  0=wild or status unknown)

###  <a name="ws">Web service

Syntax:  
```
[base_url]/nsr/nsr_ws.php?species=[Genus]%20[specific_epithet]&country=[country]&stateprovince=[state_province]&countyparish=[county_parish]&format=[output_format]
```

Example:
```
http://bien.nceas.ucsb.edu/bien/apps/nsr/nsr_ws.php?species=Pinus%20ponderosa&country=United%20States&stateprovince=Arizona&countyparish=Pima&format=json
```

Notes:  
* Accepts one species + political_division_of_observation at a time
* Parameters stateprovince, county parish and format are optional  
* Output format: xml (default),  json   

## <a name="Native">Native Status Codes

| Native status code	| Meaning 
| --------- | -------------------
| P	| Present in checklist for region of observation but no explicit status assigned
| N	| Native to region of observation
| Ne | Native and endemic to region of observation
| A	 | Absent from all checklists for region of observation
| I	| Introduced, as declared in checklist for region of observation
| Ie | Endemic to other region and therefore introduced in region of observation
| UNK | Unknown; no checklists available for region of observation and taxon not endemic elsewhere
