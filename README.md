# The assignment

We want to you build a single-page web application that would enable
a user to understand twitter data. Which framework
(angular, react, ...) to use is entirely up to you, so we will need
instructions on how to start the application, e.g. `npm start` or
"simply open index.html".

## Checking out and submitting

After we've added you as a collaborator to the repository just clone 
it and work on it locally. Once done, zip the working directory and 
email the bundle to owl@meltwater.com


## Features

Given a query for some text, we want to:

* see the top 10 results, which includes the title and a way to navigate to the URL of the
original article
* visually see the language distribution
* be able to filter by language

What other visualisations, drill-downs or similar can you think of? If you have time, implement some of them. Otherwise write them down in the readme. Be creative!


## What we expect
We will grade the code quality and the design. If you know your solution 
has limitations or bugs, please let us know.

We expect you to spend approximately 4 hours on this assignment.


## Wireframe

```
+-----------------------------------------------------------------------+
|                                                                       |
|     Query:                                                            |
|   +--------------------------------------+                            |
|   |  search query input field            |                            |
|   +--------------------------------------+                            |
|                                                                       |
|     Result list:                              Language distribution   |
|   +--------------------------------------+    +-------------------+   |
|   |                                      |    |                   |   |
|   |                                      |    |                   |   |
|   |                                      |    |                   |   |
|   | -------------------------------------|    |                   |   |
|   |                                      |    |                   |   |
|   |                                      |    |                   |   |
|   |                                      |    |                   |   |
|   | -------------------------------------|    |                   |   |
|   |                                      |    |                   |   |
|   |                                      |    |                   |   |
|   |                                      |    |                   |   |
|   | -------------------------------------|    |                   |   |
|   |                                      |    |                   |   |
|   |                                      |    +-------------------+   |
|   |                                      |                            |
|   +--------------------------------------+                            |
|                                                                       |
|                                                                       |
+-----------------------------------------------------------------------+
```


# Getting started


## Starting elasticsearch

```sh
$ docker-compose up -d
```

## Populating the database

```sh
$ curl -XPUT localhost:9200/twitter -H 'Content-Type: application/json' --data-binary @setup/mappings.json
$ curl -XPUT localhost:9200/twitter/tweet/_bulk -H 'Content-Type: application/json;charset=UTF-8' --data-binary @setup/bulk_data_20171128.json 
```

### Make sure the documents are in elasticsearch

```sh
$ curl localhost:9200/twitter/_count
{"count":23822,"_shards":{"total":5,"successful":5,"skipped":0,"failed":0}}
```

## Data structure
We have populated the elasticsearch cluster with ~200k documents from November 28 2017. All documents are in the index 'twitter' with doctype 'tweet' and have the form:

```
{
  "text": "@Project_Veritas 🚨Undercover video inside Washington Post shows their Natl Security Correspondent & Director of Pro… https://t.co/Uvjc7Qm1SQ",
  "url": "http://twitter.com/r_little_finger/statuses/935333742998163457",
  "publishDate": 1511835916000,
  "languageCode": "en",
  "coordinates": {
    "lat": 39.76838,
    "lon": -86.15804
  },
  "author": {
    "displayName": "J_Patriot_Train",
    "userName": "r_little_finger",
    "followers": 19234,
    "imageUrl": "https://pbs.twimg.com/profile_images/907570364955467776/KULvJNYr_normal.jpg",
    "languageCode": "en"
  }
}

```

## Performing a query
Elasticsearch documentation: https://www.elastic.co/guide/en/elasticsearch/reference/current/index.html

To make things easier, we have compiled a list of commands that might be useful for this task.


### Queries

To search in all documents, in the field 'text', for the term 'apple' you'd send this query (on the same machine where you had run docker-compose up):

```sh

curl -XPOST 'localhost:9200/twitter/_search?pretty' -H 'Content-Type: application/json' -d'
{
    "query": {
        "term" : {
            "text" : "apple"
        }
    }
}
'
```

The results are in json format.

### Aggregations
For these examples, the size is 0, note that you might want to change this in your code. The examples below has two parts - first it specifies a search query and then it specifies the aggregations. You can have many aggregations in one request.


To get an aggregation over the languages, for a given query, execute this command:

```sh

curl -XPOST 'localhost:9200/twitter/_search?size=0&pretty' -H 'Content-Type: application/json' -d'
{

    "query": {
        "term" : {
            "text" : "apple"
        }
    },
    "aggs" : {
        "language_aggregatation" : {
            "terms" : { "field" : "languageCode" }
        }
    }
}
'

```


To get an aggregation of how many tweets that contain the word 'apple' that are close to Washington DC, run this search:
(distance units are meters)

```sh

curl -XPOST 'localhost:9200/twitter/_search?size=0&pretty' -H 'Content-Type: application/json' -d'
{

    "query": {
        "term" : {
            "text" : "apple"
        }
    },
    "aggs" : {
        "distance_from_washington_dc" : {
            "geo_distance" : {
                "field" : "coordinates",
                "origin" : "38.8935754,-77.0847872",
                "ranges" : [
                    { "to" : 300000 },
                    { "from" : 300000, "to" : 500000 },
                    { "from" : 500000 }
                ]
            }
        }
    }
}
'

```


To get multiple aggregations in one request:

```sh

curl -XPOST 'localhost:9200/twitter/_search?size=0&pretty' -H 'Content-Type: application/json' -d'
{

    "query": {
        "term" : {
            "text" : "apple"
        }
    },
    "aggs" : {
        "distance_from_washington_dc" : {
            "geo_distance" : {
                "field" : "coordinates",
                "origin" : "38.8935754,-77.0847872",
                "ranges" : [
                    { "to" : 300000 },
                    { "from" : 300000, "to" : 500000 },
                    { "from" : 500000 }
                ]
            }
        },
        "language_aggregatation" : {
            "terms" : { "field" : "languageCode.keyword" }
        }
    }
}
'

```


### Filter

Here is an example of how to get the first 10 results in japanese:


```sh

curl -XGET 'localhost:9200/twitter/_search?size=10&pretty' -H 'Content-Type: application/json' -d'
{
  "query": { 
    "bool": { 
      "must": [
        { "term" : {"text" : "apple" }}
      ],
      "filter": [ 
        { "term":  { "languageCode": "ja" }}
      ]
    }
  }
}
'

```




# Questions?

If you have questions regarding the assingment, please contact
owl@meltwater.com. We will get back to you as soon as possible,
but expect us to answer during CET work hours.