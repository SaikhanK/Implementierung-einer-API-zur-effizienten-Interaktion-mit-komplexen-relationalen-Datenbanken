# Implementierung-einer-API-zur-effizienten-Interaktion-mit-komplexen-relationalen-Datenbanken

# Goal

In production we need a logic to dynamically `view` and `update` the database. This should be highly generic to adapt to database structure changes and changing user demands. The update feature enables simple spreadsheet like updates of the table while in the background the changes are atomically passed to the respective tables, logged and updates of related tables are triggered. Ideally, it also enables an `undo` feature.


# Acceptance Criteria

## `View` feature

* [ ] An API endpoint is implemented with the following features
  * [ ] Database tables are returned as json if requested (query parameter is implemented)
  * [ ] The available database tables are returned as options
  * [ ] Distributed data are shown in one table if requested (query parameter is implemented). Distributed data are data that are not related in one table but related via `one-to-one`, `one-to-many` or many-to-many` relations.
  * [ ] The available (constructed) tables are returned as options
  * [ ] (Optionally, if a table is returned the available columns are returned as options)
  * [ ] (Optionally, only requested columns are returned and a list of columns not returned) 
  * [ ] The requested table can be filtered as requested.
    * [ ] A query parameter is implemented to pass a filtering command as a string in simple pseudo language (e.g. <br> '(`Pump` is in [`PN81`, `PN82`] and `Impeller Material`=`IB`) or (`Pump`!=`PL08-0120 and `Stages` > `20`)' <br> )
    * [ ] (Frontend translates the column representations to the actual column names)
    * [ ] There is a meaningful response if the pseudo language filter command could not be interpreted
    * [ ] The full table is returned if the filter command could not be interpreted
    * [ ] (Optionally, frontend has features to add simple "equal" filters to filter command by click or more obvious means)  
  * [ ] The requested table can be ordered with multi column ordering (query parameter is implemented)
  * [ ] (Optionally, column elements can be grouped)
  * [ ] The access is restricted to `ENGINEERING` group
  * [ ] The access is restricted to certain groups of tables (no user data!)
  * [ ] (Optionally, previous states of the database can be selected)
  * [ ] Pagination is implemented
* [ ] A test is implemented for each feature


## `Update` feature

* [ ] Prerequisites are met
  * [ ] The database tables are modeled to enable below features (e.g. activation and deactivation timestamps exist for `catalog_` tables)
  * [ ] Unnecessary tables are removed (e.g. `catalog_pump_characteristic` and `catalog_pump` seem to be redundant)
  * [ ] The `basedata_` tables do not have an update hierarchy (i.e. updating material table does not impact the pump tables)

* [ ] The API endpoint of the `view` feature is extended by the following features
  * [ ] There is a logic implemented that handles the update of a cell of a constructed `basedata_` tables
    * [ ] Updates are restricted to `basedata_` tables.
    * [ ] A set of query parameter is implemented to pass an update request
      * [ ] ...
    * [ ] The expected record state of the request is validated. (The request is stopped with an error if the record has changed in the meantime)
    * [ ] The change of the cell value is handled atomically in the respective database table
    * [ ] Updating the database is locked as long as an update procedure is ongoing.
    * [ ] The previous state of the cell is not deleted. It only gets a time stamp that marks it to be outdated
    * [ ] The integrity of the update is checked
    * [ ] Only if the integrity of the update is validated the change is done
    * [ ] The updated constructed table is returned
    * [ ] (Optionally, some hints are returned making it easier for the user to understand the) implications. E.g. if a change of a cell value is expected to effect multiple cells the respective cells could be marked)
    * [ ] There is a wrapper with timeout feature if the update takes to long to stop the update process and unlock the database for other changes
    * [ ] A successful update triggers the update of `catalog_` tables that are based on the updated table
  
  * [ ] (Optionally, the above logic is extended to the update of multiple cells in one call)  
  
  * [ ] The update procedures of `catalog_` tables is optimized:
    * [ ] Updating `catalog_` tables is not deleting existing data but creating new records and deactivating respective existing ones using an activation and deactivation timestamp.
    * [ ] Until an update process is finished the datetime of the deactivation is used to keep the database state consistent until all tables are updated.
    * [ ] There is a cue for triggered update jobs saved to a database tables
      * [ ] The database table holds the change information (user, datetime, table, pk, previous value, new value)
    * [ ] A celery job handles the update procedure and timeouts and returns state and errors to celery flower for monitoring purpose.
    * [ ] There is some kind of admin warning system if an update procedure creates an error
    * [ ] Performance tests are done to ensure update procedure has no effect on site performance

  * [ ] (Optionally, the creation of new records is implemented)
  * [ ] (Optionally, the creation of drafts is implemented)
  * [ ] (Optionally, `super_key` fields are returned and creation of existing super key is aborted while returning a meaningful response) 

Optionally, if a record is added all super_key fields are highlighted until the super_key is unique (and complete).
  * [ ] (Optionally, the deletion/deactivation of existing records is implemented) 

* [ ] A test is implemented for each feature 




# Notes

## Models with `basedata_` prefix

The database group with the prefix `basedata_` holds the raw data at a most basic level. It is the target when database changes are conducted. It has to types of models:
- `DataModel`: Holds business data, no relations, no meta data
- `RelationModel`: Holds meta data (`(de)activation-datetime`, `user`, `state`) and relations to DataModels. Mostly called `...Master`.

A meaningful table is constructed when all relations of the `RelationModel` are replaced be the actual data of the respective `DataModels`. This process is called `denormalization`. The denormalized table has the form (as an example pump base data):

|  `Meta data` (RelationModel `...Master`") | `Identity` (DataModel 0) | `Configuration` (DataModel 1) | `Reference` (DataModel 2) | (DataModel 3) | ... | (DataModel M) |
|---|---|---|---|---|---|---|
| created_by, ... , deactivated_by | id ... pump_code | stage_count ... impeller_code | speed_reference  | ...  | ... |... |
| ... | ... | ... | ... | ... | ... |... |
| ... | ... | ... | ... | ... | ... |... |

The super key (determines the relevant columns for uniqueness) is defined by certain fields in `Identity`, `Configuration` and `Reference` (if existing) and the activation and deactivation datetime.

## Models with `catalog_` prefix

The models with prefix `catalog_` hold processed data to minimize computations. They are fetched by APIs when data are requested for selection and filter features. These models are all based on `DataModel` (but should be extended to `CatalogModel` to hold activation and deactivation datetime.)


## How to handle growing tables

The `basedata_` tables are well normalized to minimize redundancy. They are expected to grow slow. The `catalog_` tables might grow fast. The disc space consumed can be limited by deleting older data. The performance impact will be none since the tables are cached.


## View `basedata_` tables

The current implementation uses the class `QueryMaster` to retrieve a fully constructed (denormalized) table as a pandas.DataFrame. It could be embedded in a view that allows filtering of rows and columns (or whole models = multiple columns). 

## Ideas including more frontend interaction

Optionally if the user requests a change of a record where the current data of an effected `DataModel` are referenced by other records he gets a corresponding note. He optionally gets a selection option to select for which additional super_keys the change is to be executed. For an instance a user requests a change of QH coefficients of a certain pump configuration X. The current active QH coefficients are also used by pump configuration Y and Z. Therefore the user gets a corresponding note and the option to also select changing QH coefficients of configuration Y and Z. Which the user might want if it is a general update and not a specification of a previously unspecified difference.

Optionally if the user requests to delete all records of a certain `Identity` instance and this `Identity` instance is referenced by another `Master` the user is informed and forced to again confirm the requested update.


## Thoughts about the update procedure of `basedata_`

An update is a list of changes. A change can be one of the following:
- create a record
- delete a record (i.e. soft delete -> deactivate the record)
- updating a record is a combination of both changes.

Each update is executed automatically. Only if all changes can be executed, the update is executed. Otherwise the update returns an error. Effected database tables are locked (read only) until the update process is terminated. All changes are applied with `atomic=True`, i.e. each change is only applied if all changes are applied, without causing an error.

For every record that is to be deleted the corresponding Master record is set to <br> "deactivated_at=<datetime-of-update-request>".

For every record that is to be created some validations are executed:
- super_key is unique and complete 
- all required fields (of effected models) are given
- field values meet defined choices and field types
- ...

## Thoughts about `catalog_` updates

*  [ ] Implement a function that can create a list of database transactions based on two database states represented as pandas dataframes.
    *  [ ] Input: `model to be updated`, `Current model records as dataframe`, `Target model records as dataframe`, `index <-> super_key` mapping,
    *  [ ] Get records that are to be created, change or deleted.
    *  [ ] User `index <-> super_key` mapping, to modify create, change and delete.
   
*  [ ] Implement a function that applies collected list of database transactions atomically. 

*  [ ] Use these functions for all (manually triggered) catalog update processes

*  [ ] Create a model to collect update task parameter (or check out what celery offers to manage tasks)

*  [ ] Create a task template that can be used to apply the tasks defined in the task parameter
   *  [ ] Write informations about the task state to the above model
   *  [ ] Check out celery visualisation tools for tasks
   *  [ ] Lock the source database tables for any changes
   *  [ ] Implement a check, that the transactions lead to the desired result

*  [ ] Apply update tasks 
 

TODO Propose update tasks: What tasks are to be done for a `catalog` update
TODO Propose tracking solution: Where are `catalog` updates tracked? Model, methods, errorhandlings
