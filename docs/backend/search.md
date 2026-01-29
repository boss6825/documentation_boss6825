---
myst:
  html_meta:
    "description": "How to query and search content in Plone using the catalog"
    "property=og:description": "How to query and search content in Plone using the catalog"
    "property=og:title": "Search"
    "keywords": "Plone, search, catalog, query, brains, ZCatalog, portal_catalog, FieldIndex, KeywordIndex, DateIndex, searchResults"
---

(backend-search-label)=

# Search

Searching is the action of retrieving data from Plone's catalog based on specific criteria.
In Plone, this typically means querying content items using either the `plone.api.content.find` function or directly using the `portal_catalog` tool.

This chapter focuses on **querying** the catalog to find content.
For information about **indexing** content and making it searchable, see {doc}`/backend/indexing`.


(backend-search-catalog-label)=

## Catalog

Plone uses the ZODB to store content in a flexible, hierarchical manner.
However, searching through this object graph directly would require loading each object into memory, which would be prohibitively slow on large sites.

The ZCatalog solves this problem by providing a table-like structure optimized for searching.
In Plone, the main ZCatalog instance is called `portal_catalog`.
Content is automatically indexed when created or modified, and unindexed when removed.


(backend-search-indexes-vs-metadata-label)=

### Indexes versus metadata

The catalog manages two types of data:

Indexes
:   Searchable fields that you can query against.
    Different index types support different query operations.
    For example, `FieldIndex` for exact matches, `KeywordIndex` for list values, and `ZCTextIndex` for full-text search.

Metadata (columns)
:   Copies of object attributes stored in the catalog.
    Metadata values are returned with search results, allowing you to access common attributes without loading the full object.

You can view the available indexes and metadata columns through the Zope Management Interface (ZMI) by navigating to `portal_catalog`.


(backend-search-brains-label)=

### Catalog brains

When you search the catalog, the results are not actual content objects.
Instead, you receive *catalog brains*, which are lightweight proxy objects.

Brains are lazy in two ways:

1.  They are created only when your code requests each result.
2.  They don't load the actual content objects from the database.

This lazy behavior provides significant performance benefits.
You can iterate through thousands of search results without loading any objects into memory.

Brains provide access to:

-   All metadata columns defined in the catalog
-   Methods to retrieve the actual object when needed
-   The path and URL of the indexed content

```{note}
Calling `brain.getObject()` loads the full object from the database.
This has performance implications when working with many results.
Use metadata whenever possible to avoid unnecessary database access.
```


(backend-search-other-catalogs-label)=

### Other catalogs

Besides `portal_catalog`, Plone maintains additional specialized catalogs:

`uid_catalog`
:   Maintains a lookup table for objects by their Unique Identifier (UID).
    UIDs remain constant even when objects are moved.

`reference_catalog`
:   Tracks inter-object references by UID.
    Used internally by relation fields.

Add-on products may install their own catalogs optimized for specific purposes.


(backend-search-querying-label)=

## Querying the catalog


(backend-search-accessing-catalog-label)=

### Accessing the catalog

The recommended way to search for content is using `plone.api`:

```python
from plone import api

# Search using api.content.find (recommended)
results = api.content.find(portal_type='Document')

# Get the catalog tool directly
catalog = api.portal.get_tool('portal_catalog')
```

You can also use the traditional `getToolByName` helper:

```python
from Products.CMFCore.utils import getToolByName

catalog = getToolByName(context, 'portal_catalog')
```


(backend-search-performing-queries-label)=

### Performing queries

There are several ways to query the catalog:

```python
from plone import api

# Using api.content.find (recommended for most cases)
results = api.content.find(portal_type='Document')

# Using the catalog directly
catalog = api.portal.get_tool('portal_catalog')
results = catalog(portal_type='Document')

# Using searchResults explicitly
results = catalog.searchResults(portal_type='Document')
```

Pass search criteria as keyword arguments, where the key is an index name and the value is the search term.

```python
# Find all published News Items
results = api.content.find(
    portal_type='News Item',
    review_state='published'
)
```

Multiple criteria are combined with logical AND.
The query above finds items that are both a News Item AND in the published state.

Calling the catalog without arguments returns all indexed content:

```python
# Get all content (use with caution on large sites)
all_brains = catalog()
```
