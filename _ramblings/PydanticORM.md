---
layout: page
title: 'A Minial ORM for Python, Leaning on Pydantic' 
order: 3
---

### A Minimal ORM for Python Leaning on Pydantic
We reached a point in an urgent project recently where we needed to make an 11th hour change in our backend. We would no longer be connecting to an MSSQL DB, but an intermediary data store that only accepts generic ODBC/JDBC connections.

We were depending on SQLAlchemy as a significant ORM library to manage non-trivial transactions (bulk upserts, bulk expire-create updates, etc...). Unfortunately, I couldn't find a SQLAlchemy Dialect that would work with our data store (I tried [sqlalchemy-jdbcapi](https://pypi.org/project/sqlalchemy-jdbcapi/) without luck).

It might have made sense at this point to dig deeper into SQLAlchemy's internals and implement our own Dialect but it was a scary path to take given our tight deadline (in hindsight it may have been the right call).

Regardless, in place of SQLAlchemy I developed a minimal ORM library that is (rather tightly) coupled to Pydantic, using `BaseModel` instances as the object representation of database records (currently 1 object = 1 row).

#### Our goals/requirements of this library were as follows:
- A declarative API for describing how database tables and columns map to python object representations
- Basic CRUD (create/retrieve/update/delete) operations for database tables
- Validation of all data using Pydantic prior to writing to the database via update/insert operations
- Validation of all data retrieved from the database to ensure we're type-safe
- Some mechanism for easily wrapping more sophisticated operations in a transaction to ensure atomicity
- An class structure that can be easily extended/inherited to add further functionality

#### Where we're currently
If we have a database table:
```SQL
CREATE TABLE my_table(
  some_identifier string,
  some_value boolean,
  some_other_value integer,
)
```
We can map to it as follows:
```python
from PydanticORM import Table
from pydantic import BaseModel
from typing import Optional, List

class MyTableT(BaseModel):
  id:str
  value:bool
  other_value:int

# TableMapper class handles the actual representation mapping. With knowledge of
# the SQL structure, TableMapper implements generation of SQL, and query 
# parameterization
table_mapping = TableMapper(
  table_name="my_table",
  column_mapping = {
    "some_identifier": "id",
    "some_value": "value",
    "some_other_value": "other_value",
  },
  primary_keys={"some_identifier"}
)

MyTable = Table(model=MyTableT, mapping=table_mapping)

with TransactionManager(con) as con:
  row: Optional[MyTableT] = MyTable.get_one_safe(con, id="somethign")
  rows: List[MyTableT] = MyTable.get(con, value=True)

  MyTable.create(con, id="1234", value=True, other_value=1234)
  MyTable.update(con, value=False) # Error => primary key not provided
  MyTable.update_many(con, value=False) # permitted
  MyTable.upsert(con, value=False) # Error -> PK required for upsert
  MyTable.upsert(con, id="1234", other_value=4321) # performs update if row w/ pk exists, otherwise inserts

# Delete apis are on their way, just haven't needed to implement them yet...
```
We also created an `Expire` mixin class to enable expiring rows instead of deleting:
```python
from PydanticORM.mixins import Expire
class ExpireTable(Table, Expire):
  expire_col = "expired_ts"

MyTable = ExpireTable(model=..., table_mapping=...)

MyTable.expire(con, ...)
MyTable.update_by_expire(con, ...) # expires a row, creates a new one with the provided updates
MyTable.update_many_by_expire(con, ...)
```
(I'm not quite sure a mixin was the right pattern here, `Expire` could probably be its own subclass of `Table`)
