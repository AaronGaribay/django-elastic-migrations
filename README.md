# Django Elastic Migrations

Django Elastic Migrations provides a way to control the deployment of
multiple Elasticsearch schemas over time.

## Overview

Elastic has given us the basic tools needed to configure search indexes:

* **[`elasticsearch-py`](https://github.com/elastic/elasticsearch-py)**
  is a python interface to elasticsearch's REST API
* **[`elasticsearch-dsl-py`](https://github.com/elastic/elasticsearch-dsl-py)**
  is a Django-esque way of declaring complex Elasticsearch index schemas
  (which itself uses `elasticsearch-py`).

Technically you can do everything you need with these, but any
application using more than one index or deploying changes to schemas
will need a consistent way to create, rebuild, update, activate
and drop their indexes over time. In addition, if you use AWS
Elasticsearch, you will find that you cannot stop and apply a new mapping
to your schema, so you must create a new index with a new schema and
then reindex into that schema, which requires extra care.

Django Elastic Migrations provides Django management commands for
performing these actions, records a history of the actions performed,
and offers a conceptual layer for thinking about schemas that change
over time. It aims to be compatible with AWS Elasticsearch 6.0 and
greater.

### Models
Django Elastic Migrations provides comes with three models:
**IndexMaster**, **IndexInstance**, and **IndexAction**:

- **IndexMaster** - the parent of a series of *IndexInstances* with
  a single name, e.g. `course_search`. This isn't actually an Elasticsearch
  index itself, but every *IndexInstance* is, and points to to a single
  *IndexMaster*. In addition, each *IndexMaster* has at
  most one **active** *IndexInstance*.

- **IndexInstance** - an Elasticsearch index, configured with a schema
    at the time of creation. The Elasticsearch index name is
    the name of the *IndexMaster* plus the id of the *IndexInstance*
    model: `course_search-1`. When the schema is changed, a new
    *IndexInstance* is added with name `course_search-2`, etc.

- **IndexAction** - an action that impacts an *IndexMaster* or its
  children, such as updating the index or changing which *IndexInstance*
  is active.

### Management Commands

#### Read Only Commands

- `./manage.py es_list`
    - help: For each *IndexMaster*, list activation status and doc
      count for each of its *IndexInstances*
    - usage: `./manage.py es_list`

#### Action Commands

These management commands add an Action record in the database,
so that the history of each *IndexMaster* is recorded.

- `./manage.py es_create`
    - help: If configuration has changed, create a new IndexInstance
    - usage: `./manage.py es_create [req IndexMaster name]`
    - example: ./manage.py es_create course_search

- `./manage.py es_activate`
    - help: Activate the specified IndexInstance
    - usage: `./manage.py es_activate [req IndexMaster name] [opt IndexInstance number]`
        - if no IndexInstance is specified, the latest is activated
    - example: `./manage.py es_activate course_search-1`

- `./manage.py es_update [req IndexMaster name] [opt IndexInstance number] `
    - help: Update the documents in the specified *IndexInstance*.
      If *IndexInstance* is not supplied, updates the *active* index.
      By default, only indexes those documents that have changed
      since the last reindexing.
    - alternate usage: `./manage.py es_update --all-active`
        - update all active IndexInstances
    - flag: `--full` - do a full update of all documents in the
      index, rather than just the ones that have changed since
      the last update.
    - flag: `[req IndexMaster name] --last` - update the index
      of the *last* index in the given *IndexMaster*

- `./manage.py es_drop`
    - help: Drop the documents from the specified IndexInstance index
    - usage `./manage.py es_drop [req IndexMaster name] [req IndexInstance number]`


### Deployment Flow

#### Pre-deployment
- `course_search-1` is currently being used in prod
- login to template server, check out new codebase,
  run `./manage.py es_create course_search`
    - creates `course_search-2` with new settings and NO CHANGE to
      `course_search-1`

- `./manage.py es_update course_search --latest`
    - updates `course_search-2`, with NO CHANGE to `course_search-1`
    - takes potentially a longer time

#### Deployment
A django migration runs the following:
- `./manage.py es_create course_search 2`
    - no change to index, because schema is already in index

- `./manage.py es_update course_search 2`
    - updates those docs that have changed since earlier in the day
    - fast, since most of the docs have been indexed

- `./manage.py es_activate course_search`
    - activates latest index. All further changes get done to
      this new index


### Installation
1. Put a reference to this package in your `requirements.txt`
2. Add `django_elastic_migrations` to `INSTALLED_APPS` in your Django
   settings file
3. Add the following information to your Django settings file:
   ```
   DJANGO_ELASTIC_MIGRATIONS_ES_CLIENT = "path.to.your.singleton.ES_CLIENT"
    ```
4. Create an IndexMaster:
   ```
   from django_elastic_migrations.indexes import IndexMaster

   class CourseSearch(IndexMaster):
       name = 'course_search'
       # must be subtype of elasticsearch_dsl.document.DocType
       doc_type = CourseSearchDoc
   ```
3. Run `./manage.py migrate` to create the initial database tables


## Development

This project uses `make` to manage the build process. Type `make help`
to see the available `make` targets.

### Requirements

Then, `make requirements` runs the pip install. 

This project also uses [`pip-tools`](https://github.com/jazzband/pip-tools).
The `requirements.txt` files are generated and pinned to latest versions 
with `make upgrade`. 

### Updating Egg Info

To update the `egg-info` directory, run `python setup.py egg_info`