---
layout: post
title: Google BigQuery on Google App Engine
---

{{ page.title }}
================

As of early 2018 support for accessing Google BigQuery in an
Google App Engine (Standard Environment) application is pathetically 
incomplete, misdocumented and an ever changing target. 
As of Version 0.28.0 the `google.cloud.bigquery` library saw a major 
interface overhaul but lost some core functionality.

The library uses the requests/urllib3 stack which has 
lot's of problems on App Engine. Even if you get the ominous
requests_toolbelt running.

So for example you read 
[here](https://developers.google.com/api-client-library/python/guide/thread_safety)
([as of may 2015](https://archive.li/I4Fod):

> The google-api-python-client library is built on top of the httplib2 library, 
which is not thread-safe. 

But this is a deprecated library. You will be using 
[google-cloud-python-bigquery](https://github.com/GoogleCloudPlatform/google-cloud-python/tree/master/bigquery)
which recently got a 
[major rewrite](https://cloud.google.com/bigquery/docs/python-client-migration) 
and lost some features.

Ugh.

So how to use it? Some tips for 0.31.0 follow.


Avoiding timeouts
-----------------

Understand that there are different levels of timeouts involved.
Timeouts for polling and HTTP-Request-Timeouts. My App was plagued by random
HTTP-Request-Timeouts occuring deep within `urllib3/contrib/appengine.py`.

`client.query(sql).result(timeout=50)` did not help at all, it ounly incerased
the polling timeout.

`urlfetch.set_default_fetch_deadline(50)` did also not help. I assume because
some layer inbetween overrides the default.

What finnaly helped was pulling `client = bigquery.Client(project='myproject')`
out of the function and have the client as a module global variable, instead
of reinstantiationg it on every request.
I'm very uncomfortable with this becvause I have no Idea, what is actually
happening in there.

But this gets the job done.


Getting Pagination to work
--------------------------

Ther is a lot of documentation on howe to use PageIterators, 
getting a `next_page_token` and all kind of clerverish stuff to do 
to pagination or to hide pagination via iterators from the calling code.

So what is the default page size? Not 50 rows or something like that. It
seems the default page size is *unlimited* - so you allways get all
results in a single page.

After some had scrating you find out that the parameter `max_results` 
controls the maximum number of rows per page. So this is not like
SQL `LIMIT`. The parameter would be named more apropiately `page_size`.

The evil twist is, that some of the high level functions used to access
BigQuery query results do not pass the `max_results` to the lower 
level functions. 

So currently (0.31.0) only monkey-patching allows you to get paginated
results from the google-cloud-python-bigquery library.

To make a long story short:

	def _monkeypatched_QueryJob_result(self, timeout=None, max_results=None,
	                                   page_token=None, start_index=None):
	    from google.cloud.bigquery._helpers import DEFAULT_RETRY
	    from google.cloud.bigquery.job import QueryJob
	    from google.cloud.bigquery.query import _QueryResults

	    super(QueryJob, self).result(timeout=timeout)
	    if not self._query_results:
	        extra_params = {'maxResults': 50, 'timeoutMs'} = 4000
	        path = '/projects/{}/queries/{}'.format(self.project, self.job_id)
	        resource = self._client._call_api(
	            DEFAULT_RETRY, method='GET', path=path, query_params=extra_params)
	        self._query_results = _QueryResults.from_api_repr(resource)
	    schema = self._query_results.schema
	    dest_table = self.destination
	    return self._client.list_rows(dest_table, selected_fields=schema,
	                                  max_results=max_results, page_token=page_token,
	                                  start_index=start_index)

	    
	client = bigquery.Client(project='pyproject')
    query_job = client.query(sql)
    query_job.result = types.MethodType(_monkeypatched_QueryJob_result, query_job)
    iterator = query_job.result(timeout=15, max_results=100)
    for page in iterator.pages:
    	....

