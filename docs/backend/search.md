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


(backend-search-working-with-brains-label)=

### Working with catalog brains

Search results are iterable collections of brain objects:

```python
from plone import api

results = api.content.find(portal_type='Document')

for brain in results:
    # Access metadata directly as attributes
    print(f"Title: {brain.Title}")
    print(f"Description: {brain.Description}")
    print(f"URL: {brain.getURL()}")
    print(f"Path: {brain.getPath()}")
```

To get the actual content object from a brain:

```python
# Load the full object (performance cost)
obj = brain.getObject()
```

```{warning}
Calling `getObject()` on many brains can significantly impact performance.
Each call requires a separate database query.
Prefer using metadata columns when possible.
```

Common brain methods and attributes:

`brain.getObject()`
:   Returns the actual content object.
    Use sparingly due to performance cost.

`brain.getURL()`
:   Returns the absolute URL of the object.
    Equivalent to `obj.absolute_url()`.

`brain.getPath()`
:   Returns the physical path as a string.
    Equivalent to `'/'.join(obj.getPhysicalPath())`.

`brain.getRID()`
:   Returns the internal result ID used by the catalog.


(backend-search-available-indexes-label)=

### Available indexes

Plone provides many built-in indexes.
See {doc}`/backend/indexing` for details on index types.

Commonly used indexes include:

`Title`
:   The content object's title.

`Description`
:   The content object's description.

`SearchableText`
:   Full-text index used for site search.
    Supports operators like `AND` and `OR`.

`portal_type`
:   The content type name, such as `Document` or `News Item`.

`review_state`
:   The current workflow state, such as `published` or `private`.

`path`
:   The object's location in the site.
    Supports depth-limited searches.

`object_provides`
:   Interfaces implemented by the object.
    Useful for finding content by behavior or capability.

`Creator`
:   Username of the content creator.

`Subject`
:   Keywords/tags assigned to the content.
    This is a `KeywordIndex` that supports matching any value in a list.

`created`, `modified`
:   Creation and modification timestamps.

`effective`, `expires`
:   Publication date range for content.

You can view the complete list of indexes in the ZMI under `portal_catalog` > `Indexes`.


(backend-search-sorting-label)=

### Sorting and limiting results

Use `sort_on` and `sort_order` to sort results:

```python
from plone import api

# Sort by title alphabetically
results = api.content.find(
    portal_type='Document',
    sort_on='sortable_title',
    sort_order='ascending'
)

# Sort by modification date, newest first
results = api.content.find(
    portal_type='News Item',
    sort_on='modified',
    sort_order='descending'
)
```

The `sort_order` can be `ascending` (default), `descending`, or `reverse` (alias for descending).

To limit results, use `sort_limit` combined with Python slicing:

```python
# Get the 10 most recently modified documents
limit = 10
results = api.content.find(
    portal_type='Document',
    sort_on='modified',
    sort_order='descending',
    sort_limit=limit
)[:limit]
```

```{note}
The `sort_limit` parameter is a hint to the catalog for optimization.
It may return slightly more results, so always combine it with slicing to guarantee the exact count.
```

You can sort by multiple indexes:

```python
# Sort by portal_type, then by title within each type
catalog = api.portal.get_tool('portal_catalog')
results = catalog(
    review_state='published',
    sort_on=('portal_type', 'sortable_title'),
    sort_order='ascending'
)
```


(backend-search-query-patterns-label)=

### Common query patterns


(backend-search-query-by-path-label)=

#### Query by path

Search for content within a specific folder:

```python
from plone import api

# Find all content under /news
results = api.content.find(path='/Plone/news')

# Using a context object
folder = api.content.get(path='/news')
results = api.content.find(context=folder)
```

Limit the search depth:

```python
# Search only immediate children (depth=1)
results = api.content.find(
    context=folder,
    depth=1
)

# Using the catalog directly with path dict
catalog = api.portal.get_tool('portal_catalog')
folder_path = '/'.join(folder.getPhysicalPath())
results = catalog(
    path={'query': folder_path, 'depth': 1}
)
```

The `depth` parameter controls how deep to search:

-   `depth=0`: Returns only the folder itself
-   `depth=1`: Returns immediate children only
-   `depth=2`: Returns children and grandchildren
-   No depth specified: Returns all descendants


(backend-search-query-by-type-label)=

#### Query by content type

Search for specific content types:

```python
from plone import api

# Single type
results = api.content.find(portal_type='Document')

# Multiple types
results = api.content.find(portal_type=['Document', 'News Item', 'Event'])
```


(backend-search-query-by-interface-label)=

#### Query by interface

Search for content implementing a specific interface:

```python
from plone import api
from plone.dexterity.interfaces import IDexterityContent

# Find all Dexterity content
results = api.content.find(object_provides=IDexterityContent)

# Using the interface identifier string
results = api.content.find(
    object_provides='plone.dexterity.interfaces.IDexterityContent'
)
```

Searching by interface is more flexible than searching by `portal_type`.
Multiple content types can implement the same interface, and new types added later will automatically be included in the results.


(backend-search-query-by-date-label)=

#### Query by date

Date indexes support range queries:

```python
from plone import api
from DateTime import DateTime

# Items created today
results = api.content.find(created=DateTime())

# Items created in a date range
catalog = api.portal.get_tool('portal_catalog')
results = catalog(
    created={
        'query': (DateTime('2024-01-01'), DateTime('2024-12-31')),
        'range': 'min:max'
    }
)

# Items modified in the last 7 days
from datetime import datetime, timedelta
week_ago = DateTime(datetime.now() - timedelta(days=7))
results = catalog(
    modified={'query': week_ago, 'range': 'min'}
)
```

Range operators:

-   `min`: Values greater than or equal to the query value
-   `max`: Values less than or equal to the query value
-   `min:max`: Values between two query values (inclusive)


(backend-search-query-multiple-values-label)=

#### Query multiple values

For `KeywordIndex` fields like `Subject`, you can search for multiple values:

```python
from plone import api

# Items with ANY of the specified tags (default OR)
catalog = api.portal.get_tool('portal_catalog')
results = catalog(
    Subject=['python', 'plone']
)

# Items with ALL specified tags (AND)
results = catalog(
    Subject={
        'query': ['python', 'plone'],
        'operator': 'and'
    }
)
```


(backend-search-fulltext-label)=

#### Full-text search

Use the `SearchableText` index for full-text searches:

```python
from plone import api

# Simple text search
results = api.content.find(SearchableText='documentation')

# With boolean operators
results = api.content.find(SearchableText='plone AND documentation')
results = api.content.find(SearchableText='plone OR zope')
```

The `SearchableText` index aggregates text from multiple fields including title, description, and body text.
See {doc}`/backend/indexing` for information on adding custom fields to `SearchableText`.


## External search engines

For large sites or advanced search requirements, you can integrate external search engines:

-   [Solr](https://solr.apache.org/) - See [`collective.solr`](https://github.com/collective/collective.solr) and its [documentation](https://collectivesolr.readthedocs.io/en/latest/).
-   [`collective.elasticsearch`](https://github.com/collective/collective.elasticsearch)
-   [`collective.elastic.plone`](https://github.com/collective/collective.elastic.plone)

See [Awesome Plone - Searching and Categorizing](https://github.com/collective/awesome-plone?tab=readme-ov-file#searching-and-categorizing) for a comprehensive list of search options.


## Related content

-   {doc}`/backend/indexing`
-   {doc}`/plone.api/content`