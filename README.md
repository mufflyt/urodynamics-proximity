# Urodynamics-Proximity
This is a deposit of the code used to map proximity to urodynamics testing centers across the United States.

Urodynamics testing (CPT codes 51726, 51727, 51728, 51729, 51741) is a key diagnostic tool for evaluating lower urinary tract dysfunction including urinary incontinence, voiding dysfunction, and neurogenic bladder. This analysis examines geographic disparities in access to urodynamics testing using Medicare provider data.

Please note that, for this project, you will need an active internet connection because some data is downloaded during the R session.

Additionally, the following files need to be downloaded manually as they will be imported during the analysis:
-  To access data for urodynamics services (Click on "Filter" and use codes 51726, 51727, 51728, 51729, and 51741 for the "HCPCS_Cd" column to filter to the necessary rows related to urodynamics; name the file: "Medicare Urodynamics Data.xlsx"): https://data.cms.gov/provider-summary-by-type-of-service/medicare-physician-other-practitioners/medicare-physician-other-practitioners-by-provider-and-service/data
-  To access data for census tract shape files (file named "tlgdb_2021_a_us_substategeo.gdb"): https://www2.census.gov/geo/tiger/TGRGDB21/
-  To download major metropolitan areas (file named "tl_2021_us_cbsa.shp"; Store contents in a directory subfolder called: "RU Data"): https://catalog.data.gov/dataset/tiger-line-shapefile-2021-nation-u-s-core-based-statistical-areas
-  US Decennial census (2020) data. Click on "Census Tract" on the Geographics panel and choose "All Census Tracts in the US". Then, choose the population variable "P1_001N" and store csv file as "US Census Tract 2020 Data.csv": https://data.census.gov/all?d=DEC+Demographic+and+Housing+Characteristics
