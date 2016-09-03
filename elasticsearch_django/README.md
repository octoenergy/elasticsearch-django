# Search, part three.

This is the third iteration of "search", and it does away with much of the complexity of the previous phase ("Yuno4Search"). There are good reasons why Y4S was so complex, but in the end it defeated us. When we switched from Haystack (phase 1), we carried over a lot of the patterns which it had developed in trying to provide a generic search client for a range of backends (ElasticSearch, Solr, Whoosh). This meant excessive abstractions, and in the end a sub-optimal implementation.

We don't have this restriction, as we are using a single search application, ElasticSearch, which also happens to expose a simple HTTP interface, meaning that it can be configured, managed and queried through JSON requests. Y4S also included a lot of magic - which made it very hard to debug / tune the search indexing process. This has also been removed. There's a lot less magic in this version, but it's also a lot smaller, and simpler.

---

# Search Index Lifecycle

The basic lifecycle for a search index is simple:

1. Create an index
2. Post documents to the index
3. Query the index

Relating this to our use of search within a Django project it looks like this:

1. Create mapping file for a named index
2. Add index configuration to Django settings
3. Map models to document types in the index
4. Post document representation of objects to the index
5. Update the index when an object is updated
6. Remove the document when an object is deleted
7. Query the index
8. Convert search results into a QuerySet (preserving relevance)

---

# Django Implementation

This section shows how to set up Django to recognise ES indexes, and the models that should appear in an index. From this setup you should be able to run the management commands that will create and populate each index, and keep the indexes in sync with the database.

## Create index mapping file

The prerequisite to configuring Django to work with an index is having the mapping for the index available. This is a bit chicken-and-egg, but the underlying assumption is the you are capable of creating the index mappings outside of Django itself, as raw JSON - e.g. using the Chrome extension [Sense](https://chrome.google.com/webstore/detail/sense-beta/lhjgkmllcaadmopgmanpapmpjgmfcfig?hl=en), or the API tool [Paw](https://paw.cloud/).
(The easiest way to spoof this is to POST a JSON document representing your document type at URL on your ES instance (`POST http://ELASTICSEARCH_URL/{{index_name}}`) and then retrieving the auto-magic mapping that ES created via `GET http://ELASTICSEARCH_URL/{{index_name}}/_mapping`.)

Once you have the JSON mapping, you should save it as `search/mappings/{{index_name}}.json`.

## Configure Django settings

The Django settings for search are contained in a dictionary called `SEARCH_SETTINGS`, which should be in the main `django.conf.settings` file. The dictionary has three root nodes, `connections`, `indexes` and `settings`. Below is an example:

```
SEARCH_SETTINGS = {
    'connections': {
        'default': getenv('ELASTICSEARCH_URL'),
    },
    'indexes': {
        'people': {
            'models': [
                'profiles.FreelancerProfile',
            ]
        }
    },
    'settings': {
        # batch size for ES bulk api operations
        'chunk_size': getenv('SEARCH_INDEX_CHUNK_SIZE', 500),
        # truncate source querysets - used for testing
        'truncate': getenv('SEARCH_INDEX_TRUNCATE', None),
        # default page size for search results
        'page_size': getenv('SEARCH_INDEX_PAGE_SIZE', 25),
        # set to True to connect post_save/delete signals
        'auto_sync': getenv('SEARCH_INDEX_AUTO_SYNC', True),
    }
}
```

The `connections` node is (hopefully) self-explanatory - we support multiple connections, but in practice you should only need the one - ‘default’ connection. This is the URL used to connect to your ES instance. The `setting` node contains site-wide search settings. The `indexes` nodes is where we configure how Django and ES play together, and is where most of the work happens.

### Index settings

Inside the index node we have a collection of named indexes - in this case just the single index called `people`. Inside each index we have a `models` key which contains a list of Django models that should appear in the index, denoted in `app.ModelName` format. You can have multiple models in an index, and a model can appear in multiple indexes. How models and indexes interact is described in the next section.

**Configuration Validation**

When the app boots up it validates the settings, which involves the following:

1. Do each of the indexes specified have a mapping file?
2. Do each of the models implement the required mixins

## Implement search document mixins

So far we have configure Django to know the names of the indexes we want, and the models that we want to index. What it doesn’t yet know is which objects to index, and how to convert an object to its search index document. This is done by implementing two separate mixins - `SearchDocumentMixin` and `SearchDocumentManagerMixin`. The configuration validation routine will tell you if these are not implemented.

**SearchDocumentMixin**

This mixin must be implemented by the model itself, and it requires a single method implementation - `as_search_document()`. This should return a dict that is the index representation of the object; the `index` kwarg can be used to provide different representations for different indexes. By default this is `_all` which means that all indexes receive the same document for a given object.

```
def as_search_document(self, index=‘_all’):
    return {name: “foo”} if index == ‘foo’ else {name = “bar”}
```

**SearchDocumentManagerMixin**

This mixin must be implemented by the model’s default manager (`objects`). It also requires a single method implementation - `get_search_queryset()` - which returns a queryset of objects that are to be indexed. This can also use the `index` kwarg to provide different sets of objects to different indexes.

```
def get_search_queryset(self, index):
    return self.get_queryset().filter(foo="bar")
```

We now have the bare bones of our search implementation. We can now use the included management commands to create and populate our search index:

```
# create the index ‘foo’ from the ‘foo.json’ mapping file
$ ./manage.py create_search_index foo

# populate foo with all the relevant objects
$ ./manage.py update_search_index foo
```

The next step is to ensure that our models stay in sync with the index.

## Add model signal handlers to update index

If the setting `auto_sync` is True, then on `AppConfig.ready` each model configured for use in an index has its `post_save` and `post_delete` signals connected. This means that they will be kept in sync across all indexes that they appear in whenever the relevant model method is called. (There is some very basic caching to prevent too many updates - the object document is cached for one minute, and if there is no change in the document the index update is ignored.)

There is a VERY IMPORTANT caveat to the signal handling. It will **only** pick on changes the the model itself, and not on related (`ForeignKey`, `ManyToManyField`) model changes. If the search document it affected by such a change then you will need to implement additional signal handling yourself.

We now have documents in our search index, kept up to date with their Django counterparts. We are ready to start querying ES.

---

# Search Queries (How to Search)

## Running search queries

Search3 now includes some additional search-related features. The search itself is done using `elasticsearch_dsl`, which provides a pythonic abstraction over the QueryDSL, but also allows you to use raw JSON if required:

```
from elasticsearch_django.settings import get_client
from elasticsearch_dsl import Search

# run a default match_all query
search = Search(using=get_client())
response = search.execute()

# change the query using the python interface
search = search.query("match", title="python")

# change the query from the raw JSON
search.update_from_dict({"query": {"match": {"title": "python"}}})
```

The response from `execute` is a `Response` object which wraps up the ES JSON response, but is still basically JSON.

**SearchQuery**

The `elasticsearch_django.models.SearchQuery` model wraps this functionality up and provides helper properties, as well as logging the query:

```
from elasticsearch_django.settings import get_client
from elasticsearch_django.models import SearchQuery
from elasticsearch_dsl import Search

# run a default match_all query
search = Search(using=get_client(), index='profiles')
sq = SearchQuery.execute(search)
```

Calling the `SearchQuery.execute` class method will execute the underlying search, log the query JSON, the number of hits, and the list of hit meta information for future analysis. The `execute` method also includes these additional kwargs:

* `user` - the user who is making the query, useful for logging
* `reference` - a free text reference field - used for grouping searches together - could be session id, or brief id.
*  `save` - by default the SearchQuery created will be saved, but passing in False will prevent this.

In conclusion - running a search against an index means getting to grips with the `elasticsearch_dsl` library, and when playing with search in the shell there is no need to use anything else. However, in production, searches should always be executed using the `SearchQuery.execute` method.

## Converting search hits into Django objects

Running a search against an index will return a page of results, each containing the `_source` attribute which is the search document itself (as created by the `SearchDocumentMixin.as_search_document` method), together with meta info about the result - most significantly the relevance **score**, which is the magic value used for ranking (ordering) results. However, the search document probably doesn’t contain all the of the information that you need to display the result, so what you really need is a standard Django QuerySet, containing the objects in the search results, but maintaining the order. This means injecting the ES score into the queryset, and then using it for ordering. There is a method on the `SearchDocumentManagerMixin` called `from_search_query` which will do this for you. It uses raw SQL to add the score as an annotation to each object in the queryset. (It also adds the ‘rank’ - so that even if the score is identical for all hits, the ordering is preserved.)

```
from profiles.models import FreelancerProfile

# run a default match_all query
search = Search(using=get_client(), index='profiles')
sq = SearchQuery.execute(search)
for obj in FreelancerProfile.objects.from_search_query(sq):
    print obj.search_score, obj.search_rank
```


---

## ElasticSearch Primer

For the full picture, read [Elasticsearch: The Definitive Guide](https://www.elastic.co/guide/en/elasticsearch/guide/current/index.html).

### ElasticSearch vs. RDBMS full-text search

ElasticSearch is a (distributed) document-centric datastore that is optimised for query and analysis, and is markedly different from a traditional RDBMS (PostgreSQL, MySQL) which is a datastore optimised for efficient storage and retrieval of data through structure queries. ElasticSearch parses and tokenises text data on input and stores it in something called an [Inverted Index](https://www.elastic.co/guide/en/elasticsearch/guide/2.x/inverted-index.html) which is then used to run full-text search queries against. It is the analysis of text on input, and how ES matches text queries to this that sets it apart.

Related reading: [You Know, for Search…](https://www.elastic.co/guide/en/elasticsearch/guide/current/intro.html)

### Indexes

In ES an **index** is an addressable endpoint that contains related documents, and is analogous to a database. There are nuances to indexes, but these are outside the scope of this document. The only thing you should be aware of is that, because of the way that they are implemented, it is recommended that a given index should contain similar document types - e.g. an index called 'profiles' might contain different profile documents, but probably shouldn't contain conversation messages.

Related reading: [Indexing Documents](https://www.elastic.co/guide/en/elasticsearch/guide/current/_indexing_employee_documents.html)

### Document Types

Although ES can be used in a schemaless way (you can just POST a JSON document to an HTTP index endpoint and ES will deal with it), internally it has the concept of **document types**, which dictate how the document is analysed. These are analagous to table schemas, but much less strict. If you don't explicitly set up the mapping, ES will make a best effort to analyse the document fields according to their data type - _exact values_ (dates, numbers, bools) are just stored as-is, unanalysed, whilst _full text_ fields are analysed using the _standard analyzer_. This is great for testing ES out, but is almost certainly not what you want in production. Hence the need the apply custom mappings to your index.

Related reading: [What is a Document?](https://www.elastic.co/guide/en/elasticsearch/guide/current/document.html)

### Mappings

In order to have more control over how documents are analysed, you can set up an index with an explicit mapping that determines how each field in each document type is analysed. There is an important caveat to this, which is that because the data is analysed on input, it's not possible to change the mapping once you have started indexing documents. i.e. **change = new**. (This is not strictly true, but it's easiest to think of this way.) This means that changing the mapping involves deleting the entire index, recreating it with the new mapping and then re-indexing all of the documents you just deleted. This isn't a problem for us as our index is small, but you should be aware of this.

Related reading: [Mapping and Analysis](https://www.elastic.co/guide/en/elasticsearch/guide/current/mapping-analysis.html)

### Relevance

The key concept that ES (other search technologies are available) implements is **relevance** - that is, how relevant a document is to the query that you have supplied. This is a completely different concept to SQL querying, which performs a binary in/out query against a data set, and leaves ordering up to the user. The simplest example is Google - you can't order search results, they just appear in order of 'relevance'.

Relevance is the secret sauce of search, and the reason we are invested in ES. It removes the concept of explicit ordering (order by most recent activity, most reference, etc.) and replaces it with a complex calculation of what is the most relevant result.

Related reading: [What is Relevance?](https://www.elastic.co/guide/en/elasticsearch/guide/current/relevance-intro.html)

### Filtering and Querying

ES supports two key types of query - _structured_ and _full-text_. A structured query is like a SQL query - it can be used to **filter** the complete set of documents using logical operations. Examples of structure queries include filtering by fixed ids - e.g. City, or Discipline. A full text query is where relevance is applied. Within a full text query documents are neither included nor excluded, they are instead _scored_ based on their relevance to query. This is the most important difference to grasp when coming to search for the first time. If you query an index for "banana", a document in which the word "banana" never appears **will appear in the search results**, albeit with a relevancy score of 0.

The other key aspect of querying documents is which fields you are querying - by default all text fields are added to a single global field called '**\_all**' which is then used as the basis for full text queries. This is probably not what you want in production, and managing which text fields appear in the '_all' is an important part of the mapping structure.

Related reading: [Search in Depth](https://www.elastic.co/guide/en/elasticsearch/guide/2.x/search-in-depth.html)

### Query DSL

Whilst ES supports searches via the querystring ([search lite](https://www.elastic.co/guide/en/elasticsearch/guide/2.x/search-lite.html)), the real power of ES comes from the custom _Query DSL_. This is incredibly powerful... and horrendously complex. It's a nightmarish soup of queries, filters, filtered queries and query filters which is almost impossible to understand without examples. This is the one area where an abstraction above the DSL adds value. There is a separate python library [elasticsearch-dsl](http://elasticsearch-dsl.readthedocs.io/en/latest/) designed for this purpose.

Related reading: [Query DSL](https://www.elastic.co/guide/en/elasticsearch/guide/2.x/query-dsl-intro.html)