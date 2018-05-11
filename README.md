# Django Elastic Migrations

Django Elastic Migrations provides a way to control the deployment of
multiple Elasticsearch schemas over time. Despite the name, it doesn't
have anything to do with Django Migrations (yet?), but like that system,
it is a tool created to help update a live database with changes from 
a codebase.

## Overview

Elastic has given us basic tools needed to configure search indexes:

* **[`elasticsearch-py`](https://github.com/elastic/elasticsearch-py)**
  is a python interface to elasticsearch's REST API
* **[`elasticsearch-dsl-py`](https://github.com/elastic/elasticsearch-dsl-py)**
  is a Django-esque way of declaring complex Elasticsearch index schemas
  (which itself uses `elasticsearch-py`).

Technically you can accomplish everything you need with these, but any
application a.) using more than one index or b.) deploying changes to 
schemas will want a *consistent* way to `create`, `update`, 
`activate` and `drop` their indexes over time. In addition, if you use 
AWS Elasticsearch, you will find that you cannot stop and apply a new
mapping to your index, so you must create a new index with a new schema
and then reindex into that schema, which requires extra care.

*Django Elastic Migrations provides Django management commands for
performing these actions, records a history of the actions performed,
and offers a conceptual layer for thinking about schemas that change
over time. It aims to be compatible with AWS Elasticsearch 6.0 and
greater.

### Models
Django Elastic Migrations provides comes with three models:
**Index**, **IndexVersion**, and **IndexAction**:

- **Index** - a base name, e.g. `course_search` that's the parent of 
  several *IndexVersions*. Not actually an Elasticsearch index.
  Each *Index* has at most one **active** *IndexVersion*.

- **IndexVersion** - an Elasticsearch index, configured with a schema
    at the time of creation. The Elasticsearch index name is
    the name of the *Index* plus the id of the *IndexVersion*
    model: `course_search-1`. When the schema is changed, a new
    *IndexVersion* is added with name `course_search-2`, etc.

- **IndexAction** - an action that impacts an *Index* or its
  children, such as updating the index or changing which *IndexVersion*
  is active in an *Index*.

### Management Commands

#### Read Only Commands

- `./manage.py es_list`
    - help: For each *Index*, list activation status and doc
      count for each of its *IndexVersions*
    - usage: `./manage.py es_list`

#### Action Commands

These management commands add an Action record in the database,
so that the history of each *Index* is recorded.

- `./manage.py es_create`
    - help: If configuration has changed, create a new *IndexVersion*
      record as well as a new index in Elasticsearch
    - usage: `./manage.py es_create [req Index name]`
    - example: ./manage.py es_create course_search

- `./manage.py es_activate`
    - help: Activate the specified *IndexVersion* or activate all latest *IndexVersions*
    - usage: `./manage.py es_activate [req Index name] [opt IndexVersion number]`
        - if no *IndexVersion* is specified, the latest is activated
    - example: `./manage.py es_activate course_search-1`
    - flag: `--all-latest` - ensure all *Indexes* are activated at the
      latest available *IndexVersion* (called after deploy, before
      making a new release public)

- `./manage.py es_update [req Index name] [opt IndexVersion number] `
    - help: Update the documents in the specified *IndexVersion*.
      If *IndexVersion* is not supplied, updates the *active* index.
      By default, only indexes those documents that have changed
      since the last reindexing.
    - alternate usage: `./manage.py es_update --all-active`
        - update all active IndexVersions
    - flag: `--full` - do a full update of all documents in the
      index, rather than just the ones that have changed since
      the last update.
    - flag: `[req Index name] --last` - update the index
      of the *last* index in the given *Index*

- `./manage.py es_drop`
    - help: Drop the documents from the specified IndexVersion index
    - usage `./manage.py es_drop [req Index name] [req IndexVersion number]`

- `./manage.py predeploy`
    - help: find all schemas that have changed, and for each, create
      a new *IndexVersion* and begin reindexing. Does not activate 
      those indexes, though, assuming you will do this on your own.


### Deployment Flow

#### Development Time
- Developer (in the past) has subclassed
  `django_elastic_migrations.DEMIndex`, associating it with a
  `elasticsearch_dsl.document.DocType` schema as well as a base name
  for the index, e.g. `course_search`. See *installation* section below
  for more information.

- Developer (now) changes the schema of a `elasticsearch_dsl.document.DocType`
  associated with an `Index`, say, the `course_search` index.

- Developer runs `./manage.py es_makemigrations course_search`, which
  will, when it is run, is responsible for the *IndexVersion* preparation.
  (see below). Developer commits it into the pull request that contains
  the index schema changes.

#### Pre-deployment (optional performance optimization)
- `course_search-1` is the ES index currently being used in prod.

- Login to the template server, check out the new codebase,
  and run `./manage.py es_predeploy`. Behind the scenes, this is the same as
  calling:
    - `./manage.py es_create course_search`, which creates
      `course_search-2` with new settings (with no change to
      `course_search-1`, which is still in use)
    - `./manage.py es_update course_search --latest`. This updates
      the latest course_search index, `course_search-2` (also no change
      to `course_search-1`).

#### Deployment
A django migration runs the following:
- `./manage.py es_create course_search 2`, *which does not in this case
  make a change since index 2 was manually created*

- `./manage.py es_update course_search 2`, *which updates those docs
  that have changed since earlier in the day*. Potentially quite fast,
  if most of the docs have been indexed in pre-deployment.

#### Post-deployment, before the flip
- `./manage.py es_activate --all-latest` activates the latest indexes.
  All further reindexing events are sent to the latest *IndexVersions*.


## Installation
1. Put a reference to this package in your `requirements.txt`
2. Add `django_elastic_migrations` to `INSTALLED_APPS` in your Django
   settings file
3. Add the following information to your Django settings file:
   ```
   DJANGO_ELASTIC_MIGRATIONS_ES_CLIENT = "path.to.your.singleton.ES_CLIENT"
    ```
4. Create the `django_elastic_migrations` tables by running `./manage.py migrate`
5. Create an `DEMIndex`:
   ```
   from django_elastic_migrations import ESSearchIndex

   class CourseSearch(ESSearchIndex):

       name = 'course_search'

       # must be subtype of elasticsearch_dsl.document.DocType
       doc_type = CourseSearchDoc

       @classmethod
       def get_updated_docs_since(cls, date_time):
           """Returns DocTypes that have been modified after date_time"""
           changed_teis = TeachingElementInstance.objects.filter(
           ...
   ```
6. Run `./manage.py es_list` to see the index as available


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



# Ideas, Considerations, etc.

## Ideas

1. `./manage.py es_activate --update` flag could update and then activate
   when the last docs have been activated. This could be

## Considerations

1. If the `./manage.py predeploy` indexes without running prior,
   unapplied migrations, it could lead to unexpected behavior
