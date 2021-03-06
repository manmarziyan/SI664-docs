# Meeting 5 Exercise (macOS)

## 1.0 Back-end

### 1.1 Back up the database
Run the MySQL `mysqldump` utility to back up the unesco_heritage_sites database.  The file will 
be saved in the directory in which you issue the command unless you specify path (relative or 
absolute) for placing the file elsewhere. Save the *.sql dump file in a safe place.

:warning: Do not skip this step.  

```commandline
$ mysqldump -uarwhyte unesco_heritage_sites > unesco_heritage_sites.sql
```

### 1.2 Add new database entities
If you are Git/Github savvy, fork the [SI664-scripts](https://github.com/UMSI-SI664-2018Fall/SI664-scripts) repo and clone it to your laptop.  Otherwise download the *.zip file of the code.  If you have already forked and cloned the repo do a `git pull` to update your local working directory.  

### 1.3 Create a virtual environment
Create a virtual environment for the scripts and then run `pip` to install package dependencies 
listed in the `requirements.txt` file.

##### macOS
```commandline
$ cd path/to/SI664/scripts
$ source venv/bin/activate
(venv) $ pip3 install -r requirements.txt
```

##### Windows
```commandline
> cd path/to/SI664/scripts
> venv\Scripts\activate
(venv) > pip install -r requirements.txt
```

### 1.4 Create a .yaml configuration file
Create a `unesco_heritage_sites.yaml` configuration file. The `run_mysql_script.py` script reads 
this file in order to retrieve the database connection settings. Add the following database 
connection settings to your `unesco_heritage_sites.yaml` file.  Make sure you set `user` and 
`passwd` variables to the correct values. 

```yaml
################
#   Database   #
################

mysql:
  host: localhost
  user: [yer MySQL user]
  passwd: [yer MySQL user password]
  db: unesco_heritage_sites
  local_infile: True
```

:bulb: Git excludes this file when you push changes to your Github remote repo (see `.gitignore`)
 so you can save it in your fork of SI664-scripts without fear of exposing your connection settings to the outside world.
 
### 1.5 Add new entities to the database
The `unesco_heritage_sites` database needs to be modified as follows:

* add new tables 
  - `planet`
  - `location`
* add new properties
  - `region.planet_id` (INT, NN)
  - `country_area.location_id` (INT, NN)
* add new foreign key constraints
  - region: `region_fk_planet_id`
  - country_area: `country_area_fk_location_id`
  
The above changes will be handled by executing the `unesco_heritage_sites_add_location.sql` 
script. Review the script; understand what it does. 
  
Clean up is required once the above changes are in place.  Final changes will be handled by running 
the `unesco_heritage_sites_trim_country_area.sql` script.

* drop foreign key constraints
  - FK constraint on region_id
  - FK constraint on sub_region_id
  - FK constraint on intermediate_region_id

* drop columns
  - region_id
  - sub_region_id
  - intermediate_region_id

:bulb: The above changes can be handled in a single *.sql script but I want you to be able to 
confirm for yourself that querying on `country_area.location_id` returns the same hierarchy of 
regional affiliations as a querying on ye olde `country_area.region_id`, `country_area.sub_region_id`, and `country_area.intermediate_region_id`.

The `run_mysql_script.py` takes two arguments: `-c` for the config file path and `-p` for the 
path to the SQL script: 

```commandline
(venv) $ python3 run_mysql_script.py --help
usage: run_mysql_script.py [-h] -c CONFIG -p PATH

This python script is designed to process MySQL scripts. The script requires a
valid database connection. After opening a connection and creating a cursor,
the script creates a list of SQL statements after splitting the SQL script on
each semi-colon encountered (;). The script then loops through the statements,
attempting to execute each. If successful, the script commits the changes,
closes the cursor and then closes the connection. Otherwise, it rolls back the
transaction and reports the error encountered.

optional arguments:
  -h, --help            show this help message and exit
  -c CONFIG, --config CONFIG path to config file
  -p PATH, --path PATH path to script
```

Run as follows, tailoring the *.yaml and *.sql file paths as necessary:

##### macOS
```commandline
(venv) $ python3 run_mysql_script.py -c ./config/unesco_heritage_sites.yaml -p ./input/sql/unesco_heritage_sites_add_location.sql
```

#### Windows
```commandline
(venv) $ python run_mysql_script.py -c ./config/unesco_heritage_sites.yaml -p ./input/sql/unesco_heritage_sites_add_location.sql
```

### 1.6 Confirm country_area/location regional affiliations 
Start a new terminal session and invoke the MySQL shell.  Run the following query:

```mysql
USE unesco_heritage_sites;

SELECT CONCAT('CA: ', 
       ca.country_area_name, ' ',
       IFNULL(ca.region_id, 0), ' ',
       IFNULL(ca.sub_region_id, 0), ' ',
       IFNULL(ca.intermediate_region_id, 0)),
       CONCAT('LOC: ',
       IFNULL(l.region_id, 0), ' ',
       IFNULL(l.sub_region_id, 0), ' ',
       IFNULL(l.intermediate_region_id, 0))
  FROM country_area ca
       LEFT JOIN location l
              ON ca.location_id = l.location_id
 WHERE IFNULL(ca.region_id, 0) = IFNULL(l.region_id, 0)
   AND IFNULL(ca.sub_region_id, 0) = IFNULL(l.sub_region_id, 0)
   AND IFNULL(ca.intermediate_region_id, 0) = IFNULL(l.intermediate_region_id, 0)
 ORDER BY ca.region_id, ca.sub_region_id, ca.intermediate_region_id, ca.country_area_name\G
```

Note that the query is terminated with `\G` rather than `;`.  This instructs the shell to use the
 vertical output format. For this query at least, the vertical output simplifies the visual 
 comparison of the matching regional affiliations drawn from joining on the `location` (LOC) table
  vs the `country_area.region_id`, `country_area.sub_region_id`, and `country_area.intermediate_region_id` foreign key values in the `country_area` (CA) table.
 
```commandline
*************************** 1. row ***************************
CONCAT('CA: ',
       ca.country_area_name, ' ',
       IFNULL(ca.region_id, 0), ' ',
       IFNULL(ca.sub_region_id, 0), ' ',
       IFNULL(ca.intermediate_region_id, 0)): CA: Antarctica 0 0 0 <-- ORIGINAL
                                     CONCAT('LOC: ',
       IFNULL(l.region_id, 0), ' ',
       IFNULL(l.sub_region_id, 0), ' ',
       IFNULL(l.intermediate_region_id, 0)): LOC: 0 0 0  <-- NEW (a match)
*************************** 2. row ***************************
CONCAT('CA: ',
       ca.country_area_name, ' ',
       IFNULL(ca.region_id, 0), ' ',
       IFNULL(ca.sub_region_id, 0), ' ',
       IFNULL(ca.intermediate_region_id, 0)): CA: Algeria 1 8 0 <-- ORIGINAL
                                     CONCAT('LOC: ',
       IFNULL(l.region_id, 0), ' ',
       IFNULL(l.sub_region_id, 0), ' ',
       IFNULL(l.intermediate_region_id, 0)): LOC: 1 8 0  <-- NEW (a match)

. . .

*************************** 249. row ***************************
CONCAT('CA: ',
       ca.country_area_name, ' ',
       IFNULL(ca.region_id, 0), ' ',
       IFNULL(ca.sub_region_id, 0), ' ',
       IFNULL(ca.intermediate_region_id, 0)): CA: Wallis and Futuna Islands 5 11 0  <-- ORIGINAL
                                     CONCAT('LOC: ',
       IFNULL(l.region_id, 0), ' ',
       IFNULL(l.sub_region_id, 0), ' ',
       IFNULL(l.intermediate_region_id, 0)): LOC: 5 11 0  <-- NEW (a match)

``` 

You can also run the query you have previously run to return a list of Asian sub 
regions ordered by heritage site hectare totals, tweeked of course, to reflect the new changes to the `unesco_heritage_sites` database.

```mysql
SELECT sr.sub_region_name AS 'sub region', ROUND(CAST(SUM(hs.area_hectares) AS DECIMAL(10,1))) AS
            area_hectares
       FROM heritage_site hs
            LEFT JOIN heritage_site_jurisdiction hsj
                   ON hs.heritage_site_id = hsj.heritage_site_id
            LEFT JOIN country_area ca
                   ON hsj.country_area_id = ca.country_area_id
            LEFT JOIN location l
                   ON ca.location_id = l.location_id
            LEFT JOIN sub_region sr
                   ON l.sub_region_id = sr.sub_region_id
      WHERE sr.sub_region_name LIKE '%asia%'
      GROUP BY sr.sub_region_name
      ORDER BY area_hectares DESC;
```
```commandline
mysql> SELECT sr.sub_region_name AS 'sub region', ROUND(CAST(SUM(hs.area_hectares) AS DECIMAL(10,1))) AS
    ->             area_hectares
    ->        FROM heritage_site hs
    ->             LEFT JOIN heritage_site_jurisdiction hsj
    ->                    ON hs.heritage_site_id = hsj.heritage_site_id
    ->             LEFT JOIN country_area ca
    ->                    ON hsj.country_area_id = ca.country_area_id
    ->             LEFT JOIN location l
    ->                    ON ca.location_id = l.location_id
    ->             LEFT JOIN sub_region sr
    ->                    ON l.sub_region_id = sr.sub_region_id
    ->       WHERE sr.sub_region_name LIKE '%asia%'
    ->       GROUP BY sr.sub_region_name
    ->       ORDER BY area_hectares DESC;
+--------------------+---------------+
| sub region         | area_hectares |
+--------------------+---------------+
| Eastern Asia       |       8547176 |
| South-eastern Asia |       7149778 |
| Southern Asia      |       4080770 |
| Central Asia       |       3150430 |
| Western Asia       |        833377 |
+--------------------+---------------+
5 rows in set (0.01 sec)
```

The result set confirms that the database modifications have not introduced data integrity 
issues with respect to country/area regional affiliations.

### 1.7 country_area table cleanup
When satisfied that the `country_area` records retain their original regional affiliations via the 
new `country_area.location_id` foreign key, drop the now redundant `country_area.region_id`, 
`country_area.sub_region_id`, and `country_area.intermediate_region_id` columns.  

However, before you can drop these columns you _must first_ drop the foreign key constraints 
associated with each column.  To drop the foreign keys you need to know their constraint names. You can retrieve the constraint names by querying the `information_schema.key_column_usage` table.
 
 You can perform these operations either manually by following the directions embedded in the `unesco_heritage_sites_trim_country_area.sql` file or by executing the script using `run_mysql_query.py`. 
 
Above the expected `ALTER TABLE` statement `unesco_heritage_sites_trim_country_area.sql` 
includes instructions for retrieving the names of foreign key constraints associated with the 
`country_area table`.

```mysql
-- WARNING: before running this script confirm that the foreign key names in the ALTER TABLE
-- statement are correct before executing this script.
--
-- Run the following SQL SELECT statement against the information_schema database to ascertain the
-- foreign key constraint names for country_area.region_id, country_area.sub_region_id, and
-- country_area.intermediate_region_id. Then adjust the names accordingly in the DROP FOREIGN KEY
-- statements below. Foreign key constraints MUST be dropped before the associated column is
-- dropped.

/*
mysql> SELECT table_name, column_name, constraint_name,
    ->        referenced_table_name, referenced_column_name
    ->  FROM information_schema.key_column_usage
    -> WHERE table_name = 'country_area';
+--------------+------------------------+-----------------------------+-----------------------+------------------------+
| table_name   | column_name            | constraint_name             | referenced_table_name | referenced_column_name |
+--------------+------------------------+-----------------------------+-----------------------+------------------------+
| country_area | country_area_id        | PRIMARY                     | NULL                  | NULL                   |
| country_area | country_area_id        | country_area_id             | NULL                  | NULL                   |
| country_area | country_area_name      | country_area_name           | NULL                  | NULL                   |
| country_area | region_id              | country_area_ibfk_1         | region                | region_id              |
| country_area | sub_region_id          | country_area_ibfk_2         | sub_region            | sub_region_id          |
| country_area | intermediate_region_id | country_area_ibfk_3         | intermediate_region   | intermediate_region_id |
| country_area | dev_status_id          | country_area_ibfk_4         | dev_status            | dev_status_id          |
| country_area | location_id            | country_area_fk_location_id | location              | location_id            |
+--------------+------------------------+-----------------------------+-----------------------+------------------------+
8 rows in set, 2 warnings (0.00 sec)
*/

-- Drop country_area region-related foreign keys and columns
ALTER TABLE country_area
       DROP FOREIGN KEY country_area_ibfk_1,
       DROP COLUMN region_id,
       DROP FOREIGN KEY country_area_ibfk_2,
       DROP COLUMN sub_region_id,
       DROP FOREIGN KEY country_area_ibfk_3,
       DROP COLUMN intermediate_region_id;
```

In the example above the relevant constraint names are:

* country_area_ibfk_1 (`region_id`) 
* country_area_ibfk_2 (`sub_region_id`)
* country_area_ibfk_3 (`intermediate_region_id`)

Once you have identified the correct constraint names update the `ALTER TABLE` statement 
appropriately.  Then either execute it manually in the MySQL shell or run `run_mysql_script.py` against 
the updated `unesco_heritage_sites_trim_country_area.sql` script tailoring the *.yaml and *.sql 
file paths as necessary:

##### macOS
```commandline
(venv) $ python3 run_mysql_script.py -c ./config/unesco_heritage_sites.yaml -p ./input/sql/unesco_heritage_sites_trim_country_area.sql
```

#### Windows
```commandline
(venv) $ python run_mysql_script.py -c ./config/unesco_heritage_sites.yaml -p ./input/sql/unesco_heritage_sites_trim_country_area.sql
```

:bulb: If you encounter any SQL-related errors that you are not able to resolve, post a message to the class using the Canvas Discussion tool under the relevant topic ("Django", "MySQL", etc.).  Note your operating system version (macOS 10.13.6 (High Sierra), Windows 10, Ubuntu 18, etc.), Python version (3.7.0, 3.6.6, etc.). Describe your problem and include the traceback. 

## 2.0 Front-end

### 2.1 Update heritagesites/models.py
With the `unesco_heritage_sites` database updated you will next need to make adjustments to 
`heritagesites/models.py` to match the changes to the `unesco_heritage_sites` database.  

You, not Django, will manage all tables associated with the UNSD/UNESCO data sets.

First add two new models that represent the new tables that were added to the database: 
* `unesco_heritage_sites.planet` 
* `unesco_heritage_sites.location`

:bulb: Review the `CREATE TABLE` and `ALTER TABLE` statements in the 
`unesco_heritage_sites_add_location.sql` script to gather needed information about column names, 
data types and other constraints.

Since `models.py` is populated with existing `Model` classes that serve as ready-made examples, I am 
opting for the slimmest of mockups: 

```
class SomeName(models.Model):
    """
    New model based on Mtg 5 refactoring of the database.
    """
    some_id = models.AutoField(primary_key=True)
    # define additional properties as needed    

    class Meta:
        managed = False   <-- YOU MUST SET managed TO FALSE
        db_table = 'some_name'
        ordering = ['some_variable_name']
        verbose_name = 'some singular string'
        verbose_name_plural = 'some plural string'

    def __str__(self):
        return self.some_name <-- MUST RETURN A STRING
```

Second, add new variables and assign the proper [Field types](https://docs.djangoproject.com/en/2
.1/ref/models/fields/#model-field-types) to the `Region` and `CountryArea` models so that they 
match the changes made `unesco_heritage_sites.region` and `unesco_heritage_sites.country_area` 
tables. The variable assignments should be easy to figure out based on a review of the `unesco_heritage_sites_add_location.sql` script.

Don't forget to comment out the following `CountryArea` model variable assignments as 
they are now redundant:

```python
# region = models.ForeignKey('Region', models.DO_NOTHING, blank=True, null=True)
# sub_region = models.ForeignKey('SubRegion', models.DO_NOTHING, blank=True, null=True)
# intermediate_region = models.ForeignKey('IntermediateRegion', models.DO_NOTHING, blank=True, null=True)
```

### 2.2 Update heritagesites/admin.py
Next, turn your attention to `heritagesites/admin.py`. You need to register two new `Model`
 classes with the admin site as well as make adjustments to a couple of existing `ModelAdmin` 
 classes in order to update the Django [Admin site](https://docs.djangoproject.com/en/2.1/ref/contrib/admin/) interface with new and adjusted views. 
 
First, register your new `Model` classes created in `models.py` using the admin register decorator 
(`@admin.register(SomeModel)`). Since `admin.py` is populated with existing `ModelAdmin` classes that serve as ready-made examples, I am again opting for the slimmest of mockups:

```
@admin.register(models.SomeName)
class SomeNameAdmin(admin.ModelAdmin):
	"""
	New class added as a result of Mtg 5 database refactoring.
	"""
	fields = ['some_name', 'some_other_name']
	list_display = ['some_name', 'some_other_name']
	ordering = ['some_name', 'some_other_name']
```

Second, make adjustments to both `CountryAreaAdmin` and `RegionAdmin` classes adding new elements 
and commenting out newly redundant elements in relevant `fields`, `fieldsets`, 
`display_list`, `list_filter`, and `ordering` properties as the following example illustrates.

```
list_display = [
    'country_area_name',
	# 'region',                 <- comment out
	# 'sub_region',             <- comment out
	# 'intermediate_region',    <- comment out
	'm49_code',
	'iso_alpha3_code',
	'location',                 <- new (a hint)
	'dev_status'
]

```

### 2.3 Check the heritagesites Admin site
With the edits to `models.py` and `admin.py` in place, start up the Django development server:

#### macOS
```commandline
(venv) $ python3 manage.py runserver
Performing system checks...

System check identified no issues (0 silenced).
October 01, 2018 - 20:37:25
Django version 2.1.1, using settings 'mysite.settings'
Starting development server at http://127.0.0.1:8000/
Quit the server with CONTROL-C.
```

#### Windows
```commandline
(venv) > python manage.py runserver
Performing system checks...

System check identified no issues (0 silenced).
October 01, 2018 - 20:37:25
Django version 2.1.1, using settings 'mysite.settings'
Starting development server at http://127.0.0.1:8000/
Quit the server with CONTROL-C.
``` 

Log in as the superuser and check the `heritagesites` app admin pages.  You should encounter two new pages based on the new `ModelAdmin` classes you registered.

:bulb: If you encounter startup errors, recheck your work. If you find yourself stumped, post a 
message to the class using the Canvas Discussion tool under the relevant topic ("Django", "MySQL", etc.).  Note your 
operating system version (macOS 10.13.6 (High Sierra), Windows 10, Ubuntu 18, etc.), Python 
version (3.7.0, 3.6.6, etc.). Describe your problem and include the traceback.

## 3.0 Exercises
Now that the `unesco_heritage_sites` database has been updated and the `heritagesites` app 
updated with new as well as updated `Model` and `ModelAdmin` classes, a few tasks remain for 
completing the assignment.

### 3.1 Model classes cut-and-paste
Copy the following `Model` classes from your _updated_ `heritagesites/models.py` file:

* CountryArea
* Location
* Region

Paste the Python code into a .txt file named
 
 `<uniqname>-heritage_sites_mtg5.txt`
 
 
### 3.2 Largest protected area in Africa
Start the MySQL shell and issue a SQL SELECT statement that returns the largest protected area in 
Africa. This can be done using `MAX()`, along with a subquery. Alias each column as follows:
 
 * region
 * subregion
 * country / area
 * heritage site
 * area (hectares)

Print out the result set in the MySQL shell in vertical output form.  Then cut and paste the 
shell output into a *.txt file named
 
 `<uniqname>-heritage_sites_mtg5.txt`
 
 __Please include _both_ the SQL statement and the result set in the *.txt file.__
 
:bulb: Remember to point the MySQL shell at the unesco_heritage_sites database before issuing your 
 SELECT statement.
 
```commandline
mysql> USE unesco_heritage_sites;
Database changed
```
 
### 3.3 Developed vs Developing countries in Asia
Start the Django interactive Python shell and generate a QuerySet comprised of two dictionaries; 
the first providing a count of "developed" countries/areas in Asia; the second providing a count 
of "developing" countries/areas in Asia. 

```commandline
>>> from heritagesites.models import Location, Region, CountryArea, DevStatus
>>> from django.db.models import Count
>>> from django.db.models import F

>>> loc = Location.objects. . . 

>>> for l in loc:
...     print(l)
...
...
{'region_name': 'Asia', 'dev_status': 'Developed', 'count': x}
{'region_name': 'Asia', 'dev_status': 'Developing', 'count': Y}
```

 Append your Python code output to the *.txt 
 file named
 
 `<uniqname>-heritage_sites_mtg5.txt`
 
 __Please include _both_ the Python code and the result set in the *.txt file.__
 
### 3.4 Submit assignment 
Submit `<uniqname>-china_heritage_sites.txt` to Canvas via the assignment page.
  