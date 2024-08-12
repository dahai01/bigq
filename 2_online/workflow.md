# Geoprocessing Replica Data on BigQuery:

## 1. Prepare Your Geo Data.

   Ensure your file with geometry information is in Newline Delimited JSON (NDJSON or JSONL) format, using the WGS84 coordinate system (EPSG:4326) for compatibility. Below is an example Python code to convert a shapefile into NDJSON format:

   ```python
   import geopandas as gpd
   import json
   
   def convert_shapefile_to_ndjson(input_shapefile, output_ndjson, target_crs="EPSG:4326"):
       gdf = gpd.read_file(input_shapefile)
       
       if gdf.crs != target_crs:
           gdf = gdf.to_crs(target_crs)
       
       with open(output_ndjson, 'w') as ndjson_file:
           for _, row in gdf.iterrows():
               row_dict = row.to_dict()
               row_dict['geometry'] = row.geometry.to_wkt()
               ndjson_file.write(json.dumps(row_dict) + '\n')
   
   convert_shapefile_to_ndjson(shapefile_path, ndjson_path)
   ```

## 2. Go to BigQuery. Create Your Own Project.

   Access BigQuery and create your project. Google provide free usage of BigQuery in sandbox mode. For more information, refer to [Google's documentation](https://cloud.google.com/bigquery/docs/sandbox).

## 3. Create Your Own Dataset in the Project.

   Click on the three dots next to your project in the Explorer and select "Create dataset." Ensure to set the region as multiregion/US for further use, and set an expiration time within 60 days if in sandbox mode (free of charge).

## 4. Create Your Own Table in the Dataset.

   Click on the three dots next to your dataset in the Explorer and select "Create table." Choose the "Upload" option, select your JSON file, set the format to "JSONL," and complete the necessary fields.

## 5. Convert String to Geometry.


   Once the data is uploaded, the "geometry" field will be of STRING type. Alter your table to add a new column of type GEOGRAPHY. Then, update the table to populate this column with geometry data converted from STRING format.

   > Work in a project with bill info.

```sql
-- Add a new column for GEOGRAPHY type
ALTER TABLE `project_id.dataset_name.table_name` 
ADD COLUMN geo_gg GEOGRAPHY;

-- Populate the new column with geometry data
UPDATE `project_id.dataset_name.table_name` 
SET geo_gg = ST_GEOGFROMTEXT(geometry) 
WHERE geometry IS NOT NULL;
```

## 6. Conduct Your Geoprocessing as Needed.

   Write SQL queries to perform geoprocessing tasks. Join your table with the necessary tables from the Replica dataset and perform geo-processing using [the geography functions provided by Google BigQuery](https://cloud.google.com/bigquery/docs/reference/standard-sql/geography_functions). 


```sql
-- Example SQL to join tables and conduct geo-processing
SELECT
  polygons_table.taz_id AS taz_id,
  COUNT(points_table.person_id) AS population
FROM
  `project_id.dataset_name.table_name` AS polygons_table
JOIN
  `replica-customer.northeast.northeast_2023_Q4_population` AS points_table
ON
  ST_CONTAINS(polygons_table.geo_gg, ST_GEOGPOINT(points_table.lng, points_table.lat))
WHERE 
  points_table.lat IS NOT NULL
GROUP BY
  polygons_table.taz_id;
```
