---
layout: post
title: Keywords from thesaurus as Queryable
author: Paul van Genuchten
author_url: https://github.com/pvangenuchten
publish_date: 2024-09-18 14:59:00-0400
---

## Keywords from thesaurus as Queryable

A common convention in catalogues is the use of keywords from a dedicated thesaurus. The assignment of these keywords can then later be used to filter or query the catalogue by these terms. To achieve this use case in pycsw, some configuration needs to be tailored. This blog post indicates the changes needed for this scenario.

For this example we'll use a keyword from the [INSPIRE Themes](http://inspire.ec.europa.eu/theme) thesaurus. We will define a new queryable `inspiretheme`, which will be populated with the relevant keyword (if present).

You can repeat these steps for any other thesaurus.

## Extra database column

Extend the records table in the database with an extra field for the selected thesaurus. This is usually a manual operation on the database.

```sql
ALTER TABLE records
ADD inspiretheme VARCHAR(255);
```

## Add parameter to pycsw

In `pycsw/core/config.py` the newly created database column can be registered to pycsw. 

```python 
'pycsw:InspireTheme': 'inspiretheme',
```

## Add column to mapping

`etc/mappings.py` links the pycsw parameter to the columnname in the table.

```python 
'pycsw:InspireTheme': 'inspiretheme',
```

## Define parameter as queryable

Which of the parameters are queryable is defined in `pycsw/core/repository.py`.

 'inspiretheme': self.dataset.inspiretheme,

## Add parameter to record results?

Keywords are already published in records, so there is generally no need to extend the record with the new parameter. If needed you can do so in `pycsw/ogc/api/records.py#Line=1150`.

## Populate the parameter from record imports

We have 2 options here, either manage the population of the column within the database as part of an insert trigger on the `record.themes` field. Alternatively update `pcsw/core/metadata.py` so the column is populated when records are imported.

For the second option consider the following code. For each of the keyword blocks, it tries to match the thesaurus title or uri and, if matched, adds the keywords to the new parameter.

```python
_set(context, recobj, 'pycsw:InspireTheme', ", ".join(
    [t for t in md_identification.keywords if ( 
        t.thesaurus not in [None,''] and ((
            'name' in t.thesaurus and t.thesaurus['name'] not in [None,''] and
            t.thesaurus['name'] in ['GEMET - INSPIRE themes, version 1.0','GEMET Themes, version 2.3']
        ) or (
            'uri' in t.thesaurus and t.thesaurus['uri'] not in [None,''] and
            t.thesaurus['uri'] == 'http://inspire.ec.europa.eu/theme')))]))
```

## Add parameter to OGCAPI - Records facets

Facets enable to further limit search results. Keywords from thesauri are very usefull to add as facet. Add the paremeter to `default.yml`.

```yaml
facets:
    - type
    - inspiretheme
```
