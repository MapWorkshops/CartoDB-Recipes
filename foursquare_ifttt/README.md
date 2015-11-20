How to use an IFTTT recipe to put your checkins in a CartoDB map

1. Enable this recipe in IFTTT. This will add, for each 4SQ checkin, a new row
in a Google Drive sheet named IFTTT/Foursquare check-ins

2. After a checkin, you can see the content of the sheet. Need a couple of modifications

* A header line, with these values: DT|SHOUT|VENUE URL|MAP URL
* Leave just the url in the last field (starts with =IMAGE(...))


3. Now, create a new sync table in CartoDB, using Google Drive connector, and select the created sheet.
**Important**: Enable a sync period. 1 hour, for example.

4. A new table will be created, with null geometry field. We'll make 2 modifications:

* Transform the date string into a real date: TODO query
* The most important thing, a trigger to update the geometry column after insert/update:

```sql
CREATE OR REPLACE FUNCTION update_geom_column()
RETURNS trigger AS
$BODY$
DECLARE
  coords_array varchar[] := ARRAY[2];
BEGIN
  if new.map_url is null then
    raise exception 'map_url cannot be null';
  end if;

  -- Get coords using regexp_matches
  select regexp_matches(NEW.map_url, 'center=(-?[\d]*\.[\d]*),(-?[\d]*\.[\d]*)&') into coords_array;
  NEW.the_geom = CDB_LatLng(coords_array[1]::numeric, coords_array[2]::numeric);

  return NEW;
END;
$BODY$
LANGUAGE plpgsql VOLATILE
COST 100;

drop trigger if exists update_geom_column_trigger on foursquare_checkins;
CREATE TRIGGER update_geom_column_trigger
BEFORE INSERT OR UPDATE
ON foursquare_checkins
FOR EACH ROW
EXECUTE PROCEDURE update_geom_column();
```

  So, every time the IFTTT recipe adds a new file to the Google Drive sheet, this file will be included
  in the next synchronization (depending on the period you chose, this will happen every hour, every day...).
  And every time the table is synchronized, the_geom field will be updated. NEED TESTING