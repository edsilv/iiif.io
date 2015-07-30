---
title: "IIIF Content Search API 0.9"
title_override: "IIIF Content Search API 0.9"
id: content-search-api
layout: spec
tags: [specifications, content-search-api]
major: 0
minor: 9
patch: 0
pre: draft
cssversion: 2
---

## Status of this Document
{:.no_toc}
__This Version:__ {{ page.major }}.{{ page.minor }}.{{ page.patch }}{% if page.pre != 'final' %}-{{ page.pre }}{% endif %}

__Beta Specification for Trial Use__
This is a work in progress. We are actively seeking implementations and feedback.  No section should be considered final, and the absence of any content does not imply that such content is out of scope, or may not appear in the future.  Please send any feedback to [iiif-discuss@googlegroups.com][iiif-discuss].
{: .alert}

**Editors**

  * Michael Appleby, _Yale University_
  * Tom Crane, _Digirati_
  * Robert Sanderson, _Stanford University_
  * Jon Stroop, _Princeton University_
  * Simeon Warner, _Cornell University_
  {: .names}


_Copyright © 2015 Editors and contributors. Published by the IIIF under the [CC-BY][cc-by] license._


## Table of Contents
{:.no_toc}

* Table of Discontent (will be replaced by macro)
{:toc}

## 1. Introduction

In the IIIF [Presentation API][prezi-api], content is brought together from distributed systems via annotations.  That content might be images, often with a IIIF [Image API][image-api] service to access them, audio, video, rich or plain text, or anything else.  In a vibrant and dynamic system, that content can come from many sources and be rich, varied and abundant.  Of that list, textual content lends itself to being searched, either as the transcription, translation or edition of the intellectual content, or commentary, description, tagging or other annotations about the resource.  

This specification lays out the interoperability mechanism for performing these searches within the IIIF context.  The scope of the specification is searching textual annotation content within a single IIIF resource, such as a Manifest or Range.  Every effort is made to keep the interaction as consistent with existing IIIF patterns as possible.

In order to make searches easier against unknown content, a related service for the auto-completion of search terms is also specified. The auto-complete service is specific to a search service to ensure that the retrieved terms can simply be copied to the query of the search.

Please send feedback to [iiif-discuss@googlegroups.com][iiif-discuss]

### 1.1. Use Cases

Use cases for being able to search the annotations within the Presentation API include:

 * Searching OCR generated text to find words or phrases within a book, newspaper or other primarily textual content
 * Searching transcribed content, provided by crowd-sourcing or transformation of scholarly output
 * Searching multiple streams of content, such as the translation or edition, rather than the raw transcription of the content, to jump to the appropriate part of an object
 * Searching on sections of text, such as defined chapters or articles.
 * Searching for user provided commentary about the resource, either as a discovery mechanism for the resource or for the discussion.
 * Discovering similar sections of text to compare either the content or the object

User interfaces that could be built using the search response include highlighting matching words in the display, providing a heatmap of where the matches occur within the object, and providing a mechanism to jump between points within the object.  The auto-complete service assists users in identifying terms that exist within the selected scope.

### 1.2. Terminology

The key words _MUST_, _MUST NOT_, _REQUIRED_, _SHALL_, _SHALL NOT_, _SHOULD_, _SHOULD NOT_, _RECOMMENDED_, _MAY_, and _OPTIONAL_ in this document are to be interpreted as described in [RFC 2119][rfc-2119].

## 2. Overview

The IIIF Presentation API provides just enough information to a viewer so that it can present the images and other content to the user in a rich and understandable way.  Those content resources may have textual annotations associated with them.  Annotations may also be associated with the structural components of the Presentation API, such as the manifest itself, sequences, ranges, and layers.  Further, annotations can be replied to by annotating them to form a threaded discussion about the commentary, transcription, edition or translation.

Annotations are typically made available to viewing applications in an Annotation List, where all of the annotations in the list target the same resource, or part of it.  Where known, these lists can be directly referenced from the manifest document to allow clients to simply follow the link to retrieve them.  For fixed, curated content, this is an appropriate method to discover them, as the annotations do not frequently change, nor are they potentially distributed amongst multiple servers.

However this is less useful for comment-style annotations, crowd-sourced or distributed transcriptions, corrections to automated OCR transcription, and similar, as the annotations may be in constant flux.  Further, being able to quickly discover individual annotations without stepping through all of the views of an object is essential for a reasonable user experience.  

Beyond the ability to search for words or phrases, users find it helpful to have suggestions for what terms they should be searching for.  This facility is often called auto-complete or type-ahead, and within the context of a single object can provide insight into the language and content.  The auto-complete service is associated with a search service into which the terms can be fed as part of a query.

This specification details how, within the context of IIIF, search and auto-complete services should be implemented such that they are easy and consistent to interact with from viewing applications.

## 3. Search

The search service takes a query, including typically a search term or uri, and potentially filtering further by the date the annotation was created or last modified, the motivation for the annotation as to whether it is painting a resource on to a canvas or not, or the user that created the annotation.

### 3.1. Service Description

Any resource in the Presentation API may have a search service associated with it.  The resource determines the scope of the content that will be searched.  A service associated with a manifest will search all of the annotations on canvases or other objects below the manifest, a service associated with a particular range will only search the canvases within the range, or a service on a canvas will search only annotations on that particular canvas.  

The description of the service follows the pattern established in the [Linking to Services][service-annex] pattern.  It _MUST_ have the `@context` property with the value "http://iiif.io/api/search/{{ page.major }}/context.json", the  `profile` property with the value "http://iiif.io/api/search/{{ page.major }}/search", and the `@id` property that contains the URI where the search can be performed.  

An example service description block:

{% highlight json %}
{
  // ... the resource that the search service is associated with ...
  "service": {
    "@context": "http://iiif.io/api/search/{{ page.major }}/context.json",
    "@id": "http://example.org/services/identifier/search",
    "profile": "http://iiif.io/api/search/{{ page.major }}/search"
  }
}
{% endhighlight %}

### 3.2. Request

The search request is made to a service that is related to a particular Presentation API resource.  The URIs for services associated with different resources must be different to allow the client to use the correct one for the desired scope of the search.  To perform a search, the client _MUST_ use HTTP GET (rather than POST) to make the request to the service, with query parameters to specify the search terms.

#### 3.2.1. Query Parameters

All parameters are _OPTIONAL_ in the request.  The default, if a parameter is empty or not supplied, is to not restrict the matching annotations by that parameter.

Servers _MUST_ implement the `q` parameter, and _SHOULD_ implement the `motivation` parameter, the others are _OPTIONAL_ to support. Parameters that are received in a request but not implemented _MUST_ be ignored, and _SHOULD_ be included in the `ignored` property of the Layer in the response, described in the next section.

| Parameter  | Definition | 
| ---------  | ---------- |
| `q`          | A space separated list of search terms. The search terms _MAY_ be either words (to search in textual bodies) or URIs (to search identities of annotation body resources).  The semantics of multiple, space separated terms is server implementation dependent.|
| `motivation` | A space separated list of motivation terms. If multiple motivations are supplied, an annotation matches the search if any of the motivations are present. Expected values are given below. |
| `date`       | A space separated list of date ranges.  An annotation matches if the date on which it was created falls within any of the supplied date ranges. The dates _MUST_ be supplied in the ISO8601 format: `YYYY-MM-DDThh:mm:ssZ/YYYY-MM-DDThh:mm:ssZ`. The dates _MUST_ be expressed in UTC and _MUST_ be given in the `Z` based format. |
| `user`       | A space separated list of URIs that are the identities of users. If multiple users are supplied, an annotation matches the search if any of the users created the annotation. |
| `box`        | A space separated list of boxes specified as `x,y,w,h` that limits the area of a canvas that matching annotation must overlap with.  Annotations that target the entire canvas are considered to overlap with all possible bounding boxes. |
{: .api-table}

Common values for the motivation parameter are:

| Motivation | Definition |
| ---------- | ---------- |
| `painting`     | Only annotations with the `sc:painting` motivation |
| `non-painting` | Annotations with any motivation other than `sc:painting` |
| `commenting`   | Annotations with the `oa:commenting` motivation |
| `describing`   | Annotations with the `oa:describing` motivation |
| `tagging`      | Annotations with the `oa:tagging` motivation |
{: .api-table}

Other motivations are possible, and the full list from the [Open Annotation][openanno] specification _SHOULD_ be available by dropping the "oa:" prefix.  Other, community specific motivations _SHOULD_ include a prefix or use their full URI.

#### 3.2.2. Example Request

This example request:

```
http://example.org/services/manifest/search?q=bird&motivation=painting
```
{: .urltemplate}

Would search for annotations with the word "bird" in their textual content, and have the motivation of `painting`.  It would search within the resource it was associated with.

### 3.3. Response

The response from the server is an [AnnotationList][prezi-annolist], following the format from the Presentation API with a few additional features.  This allows clients that already implement the AnnotationList format to avoid further implementation work to support search results.

The search results are returned as Annotations in the regular IIIF syntax. Note that the annotations can come from multiple canvases, rather than the default situation from the Presentation API where all of the annotations target a single canvas.

#### 3.3.1. Simple Lists

The simplest response looks exactly like a regular AnnotationList, other than the `@context` which is from this API. The value of `@id` will be the same as the URI used in the query.  If the server does not support all of the request parameters it _MUST_ instead use the response structure discussed in the following section in order to report which parameters could not be processed.

Clients wishing to know the total number of annotations that match may simply count the number of annotations in the `resources` property, as all have been returned.

{% highlight json %}
{
  "@context":"http://iiif.io/api/search/{{ page.major }}/context.json",
  "@id":"http://example.org/service/manifest/search?q=bird&motivation=painting",
  "@type":"sc:AnnotationList",

  "resources": [
    {
      "@id": "http://example.org/identifier/annotation/anno-line",
      "@type": "oa:Annotation",
      "motivation": "sc:painting",
      "resource": {
        "@type": "cnt:ContentAsText",
        "chars": "There are two birds in the bush"
      },
      "on": "http://example.org/identifier/canvas1#xywh=100,100,250,20"
    }
    // Further annotations here ...
  ]
}
{% endhighlight %}

#### 3.3.2. Paging Results and Ignored Parameters

For very long lists of annnotations, the server may choose to divide the response into multiple sections, often called pages.  Each page is an AnnotationList and can refer to other pages to allow the client to traverse the entire set.

The next page of results _MUST_ be referenced in a `next` property of the AnnotationList, and the previous page _MUST_ be referenced in a `prev` property.  Pages _SHOULD_ refer to the first and last pages with `first` and `last` properties, respectively.  The URI of the first AnnotationList reported in the `@id` property _MAY_ be different from the one used by the client to request the search.  Each page _SHOULD_ also have a `startIndex` property with an integer value that reports the position of the first result within the entire resultset, where the first annotation has an index of 0.  For example, if the client has requested the first page which has 10 hits, then the `startIndex` will be 0, and the `startIndex` of second page will be 10, being the 11th hit.

All of the pages are within a [Layer][prezi-layer] that represents the entire resultset of matched annotations.  The layer is the value of a `within` property on each of the page AnnotationLists.  The layer _MAY_ have a URI given as the value of the `@id` property.  The layer _SHOULD_ have a `total` property which is the total number of hits generated by the query.

If the server has ignored any of the parameters in the request, then the layer _MUST_ be present and _MUST_ have an `ignored` property where the value is a list of the ignored parameters.

An example request:

```
http://example.org/service/manifest/search?q=bird&box=0,0,100,100
```
{: .urltemplate}

And the response for the first page of annotations from a total of 125 matches, where the service did not process the `box` parameter:

{% highlight json %}
{
  "@context":"http://iiif.io/api/search/{{ page.major }}/context.json",
  "@id":"http://example.org/service/manifest/search?q=bird&page=1",
  "@type":"sc:AnnotationList",

  "within": {
    "@type": "sc:Layer",
    "total": 125,
    "ignored": ["box"]
  },
  "next": "http://example.org/service/identifier/search?q=bird&page=2",
  "last": "http://example.org/service/identifier/search?q=bird&page=13",
  "startIndex": 0,

  "resources": [
    {
      "@id": "http://example.org/identifier/annotation/anno-line",
      "@type": "oa:Annotation",
      "motivation": "sc:painting",
      "resource": {
        "@type": "cnt:ContentAsText",
        "chars": "There are two birds in the bush"
      },
      "on": "http://example.org/identifier/canvas1#xywh=100,100,250,20"
    }
    // Further annotations from the first page here ...
  ]
}
{% endhighlight %}

#### 3.3.3. Target Resource Structure

The annotations may also include references to the structure or structures that the target (the resource in the `on` property) is found within.  The URI and type of the including resource _MUST_ be given, and a `label` _SHOULD_ be included.

This structure is called out explicitly as although it uses only properties from the Presentation API, it is not a common or documented pattern, and thus clients may not be expecting it.

{% highlight json %}
{
  "@context":"http://iiif.io/api/search/{{ page.major }}/context.json",
  "@id":"http://example.org/service/manifest/search?q=bird&motivation=painting",
  "@type":"sc:AnnotationList",

  "resources": [
    {
      "@id": "http://example.org/identifier/annotation/anno-line",
      "@type": "oa:Annotation",
      "motivation": "sc:painting",
      "resource": {
        "@type": "cnt:ContentAsText",
        "chars": "There are two birds in the bush"
      },
      "on": {
        "@id": "http://example.org/identifier/canvas1#xywh=100,100,250,20",
        "within": {
          "@id": "http://example.org/identifier/manifest",
          "type": "sc:Manifest",
          "label": "Example Manifest"
        }
      }
    }
    // Further annotations here ...
  ]
}
{% endhighlight %}

#### 3.3.4. Search Specific Features

There may be properties that are specific to the search result, and not features of the annotation in general, that are valuable to return to the client.  Examples include the text before and after the matched content to allow a result snippet to be presented, the matched text in case stemming or wildcards have been applied, or to link together annotations that together fulfil the search query for a phrase, such as when the phrase spans across lines of text.

To incrementally build upon existing solutions and provide graceful degradation for clients that do not support these features, the information is included in a second list within the AnnotationList called `hits`.  AnnotationLists _MAY_ have this property, and servers _MAY_ support these features through their use.

If supported, each entry in the list is a `search:Hit` object.  This type must be included as the value of the `@type` property. Hit objects reference one or more annotations that they provide additional information for, in a list as the value of the hit's `annotations` property.  The reference is made to the value of the `@id` property of the annotation, and thus annotations _MUST_ have a URI to enable this further information.

The basic structure is:

{% highlight json %}
{
  "@context":"http://iiif.io/api/search/{{ page.major }}/context.json",
  "@id":"http://example.org/service/manifest/search?q=bird&page=1",
  "@type":"sc:AnnotationList",

  "resources": [
    {
      "@id": "http://example.org/identifier/annotation/anno-line",
      "@type": "oa:Annotation"
      // More regular annotation information here
    }
    // Further annotations from the first page here ...
  ],

  "hits": [
    {
      "@type": "search:Hit",
      "annotations": [
        "http://example.org/identifier/annotation/anno-line"
      ]
      // More search specific information here
    }
    // Further hits for the first page here ...
  ]
}
{% endhighlight %}

__3.3.4.1 Search Term Snippets__

The simplest addition to the Hit object is to add text that appears before and after the matching text in the annotation.  This allows the client to construct a snippet where the matching text is provided in the context of surrounding content, rather than simply by itself.  This is most useful when the service has word-level boundaries of the text on the Canvas, such as available when OCR has been used to generate the text positions.

The service _MAY_ add a `before` property to the Hit with some amount of text that appears before the content of the annotation (given in `chars`), and _MAY_ also add an `after` property with some amount of text that appears after the content of the annotation.

For example, in a search for the query term "bird" in our example sentence, when the server has full word level coordinates:

```
http://example.org/service/manifest/search?q=bird
```
{: .urltemplate}

That the server matches against the plural "birds":

{% highlight json %}
{
  "@context":"http://iiif.io/api/search/{{ page.major }}/context.json",
  "@id":"http://example.org/service/manifest/search?q=bird&motivation=painting",
  "@type":"sc:AnnotationList",

  "resources": [
    {
      "@id": "http://example.org/identifier/annotation/anno-bird",
      "@type": "oa:Annotation",
      "motivation": "sc:painting",
      "resource": {
        "@type": "cnt:ContentAsText",
        "chars": "birds"
      },
      "on": "http://example.org/identifier/canvas1#xywh=200,100,40,20"
    }
    // Further annotations here ...
  ],

  "hits": [
    {
      "@type": "search:Hit",
      "annotations": [
        "http://example.org/identifier/annotation/anno-bird"
      ],
      "before": "There are two ",
      "after": " in the bush"
    }
    // Further hits for the first page here ...
  ]
}
{% endhighlight %}

__3.3.4.2. Highlight Search Term in the Annotation Content__

Many systems do not have full word-level coordinate information, and are restricted to line or paragraph level boundaries.  In this case the most useful thing that the client can do is to display the entire annotation and highlight the hits within it.  This is similar, but different, to the previous use case.  Here the word will appear somewhere within the `chars` property of the annotation, and the client needs to make it more prominent.  In the previous situation, the word was the entire content of the annotation, and the information was convenient for presenting it in a list.

The client in this case needs to know the text that caused the service to create the hit, and enough information about where it occurs in the content to reliably highlight it and not highlight non-matches.  To do this, the service can supply text before and after the matching term within the content of the annotation, via an Open Annotation TextQuoteSelector.  TextQuoteSelectors have three properties: `exact` to record the exact text to look for, `prefix` with some text before the match, and `suffix` with some text after the match.

This would look like:

{% highlight json %}
{
  "@type": "oa:TextQuoteSelector",
  "exact": "birds",
  "prefix": "There are two ",
  "suffix": " in the bush"
}
{% endhighlight %}

As multiple words might match the query within the same annotation, multiple selectors may be given in the Hit as objects within a `selectors` property.  For example, if the search used a wildcard to search for all words starting with "b" it would match the same annotation twice:

```
http://example.org/service/manifest/search?q=b*
```
{: .urltemplate}

The result might be:

{% highlight json %}
{
  "@context":"http://iiif.io/api/search/{{ page.major }}/context.json",
  "@id":"http://example.org/service/manifest/search?q=b*&page=1",
  "@type":"sc:AnnotationList",

  "resources": [
    {
      "@id": "http://example.org/identifier/annotation/anno-line",
      "@type": "oa:Annotation",
      "motivation": "sc:painting",
      "resource": {
        "@type": "cnt:ContentAsText",
        "chars": "There are two birds in the bush."
      },
      "on": "http://example.org/identifier/canvas1#xywh=200,100,40,20"
    }
    // Further annotations here ...
  ],

  "hits": [
    {
      "@type": "search:Hit",
      "annotations": [
        "http://example.org/identifier/annotation/anno-line"
      ],
      "selectors": [
        {
          "@type": "oa:TextQuoteSelector",
          "exact": "birds",
          "prefix": "There are two ",
          "suffix": " in the bush"
        },
        {
          "@type": "oa:TextQuoteSelector",
          "exact": "bush",
          "prefix": "two birds in the ",
          "suffix": "."
        }        
      ]
    }
    // Further hits for the first page here ...
  ]
}
{% endhighlight %}

__3.3.4.3. Multiple Matching Annotations in a Single Hit__

There may be multiple annotations that match a single phrase search.  If the following line of the text was a continuation of the sentence "and they are worth one in the hand", and the user searched for the phrase "bush and", the match would span multiple annotations regardless of whether the annotation was at word or line level.

In this case there needs to be more annotations than actual hits, as two annotations are needed to make up this one hit.   When the phrase spans multiple lines, only the lines can be matched as the highlighting selectors aren't associated directly with each annotation, and just the annotations would be listed together.

For the word level annotation case, the response might look like:

{% highlight json %}
{
  "@context":"http://iiif.io/api/search/{{ page.major }}/context.json",
  "@id":"http://example.org/service/manifest/search?q=bird&motivation=painting",
  "@type":"sc:AnnotationList",

  "resources": [
    {
      "@id": "http://example.org/identifier/annotation/anno-bush",
      "@type": "oa:Annotation",
      "motivation": "sc:painting",
      "resource": {
        "@type": "cnt:ContentAsText",
        "chars": "bush"
      },
      "on": "http://example.org/identifier/canvas1#xywh=200,100,40,20"
    },
    {
      "@id": "http://example.org/identifier/annotation/anno-and",
      "@type": "oa:Annotation",
      "motivation": "sc:painting",
      "resource": {
        "@type": "cnt:ContentAsText",
        "chars": "and"
      },
      "on": "http://example.org/identifier/canvas1#xywh=200,100,40,20"
    },
    // Further annotations here ...
  ],

  "hits": [
    {
      "@type": "search:Hit",
      "annotations": [
        "http://example.org/identifier/annotation/anno-bush",
        "http://example.org/identifier/annotation/anno-and",
      ],
      "match": "bush and",
      "before": "There are two birds in the ",
      "after": " they are worth two in the hand."
    }
    // Further hits for the first page here ...
  ]
}
{% endhighlight %}

__Issue: Cannot use Selectors for Multiple Lines__
We can't duplicate this with selectors, as the selector isn't tied to the annotation at all, that happens through a SpecificResource and is for the ContentAsText ... which we don't want to have to mint a URI for.
{: .warning}

__Properties of Layers__

| Field   | Definition |
| ------- | ---------- |
| total   | |
| ignored | |
{: .api-table}

__Properties of AnnotationLists__

| Field | Definition |
| ----- | ---------- |
| startIndex | The index within the full result set, starting at 0, of the first annotation returned in the list |
| next | The page that follows the current page |
| prev | The page the precedes the current page |
| first | The first page in the sequence, if not the current one |
| last| The last page in the sequence, if not the current one |
{: .api-table}

__Properties of Hits__

| Field   | Definition |
| ------- | ---------- |
| annotations | | 
| match   | |
| before  | |
| after   | |
| selectors | |


## 4. Autocomplete

The autocomplete service returns terms that can be added into the `q` parameter of the related search service, given the first characters of the term.

### 4.1. Service Description

The autocomplete service is nested within the search service that it provides term completion for.  This is to allow multiple search services, each with their own autocomplete service.

The autocomplete service _MUST_ have an `@id` property with the value of the URI where the service can be interacted with, and _MUST_ have a `profile` property with the value "http://iiif.io/api/search/{{ page.major }}/autocomplete" to distinguish it from other types of service.

{% highlight json %}
{
  // Resource that the services are associated with ...
  "service": {
    "@context": "http://iiif.io/api/search/{{ page.major}}/context.json",
    "@id": "http://example.org/services/identifier/search",
    "profile": "http://iiif.io/api/search/{{ page.major }}/search",
    "service": {
      "@id": "http://example.org/services/identifier/autocomplete",
      "profile": "http://iiif.io/api/search/{{ page.major }}/autocomplete"
    }
  }
}
{% endhighlight %}

### 4.2. Request

The request is very similar to the search request, with one additional parameter to allow the number of occurrences of the term within the object to be constrained.  The value of the `q` parameter is the beginning characters from the term to be completed by the service.  For example, the query term of 'bir' might complete to 'bird', 'biro', 'birth', and 'birthday'.

The other parameters (`motivation`, `date`,`user` and `box`), if supported, would refine the set of terms in the response to only ones from the annotations that match those filters.  For example, if the motivation is given as "painting", then only text from painting transcriptions will contribute to the list of terms in the response.

#### 4.2.1. Query Parameters

| Parameter | Definition |
| --------- | ---------- |
| `min`       | The minimum number of occurrences for a term in the index in order for it to appear within the response ; default is 1 if not present.  Support for this parameter is not required |
{: .api-table}

#### 4.2.2. Example Request

An example request

```
http://example.org/service/identifier/autocomplete?q=bir&motivation=painting&box=0,0,10,10
```
{: .urltemplate}

### 4.2. Response

The response is a list of very simple objects that include the term and its number of occurrences.  The number of terms provided is determined by the server.

Parameters that were not processed by the service _MUST_ be returned in the response in the `ignored` property.

The terms _SHOULD_ be provided in ascending alphabetically sorted order.

__Question: Return Full Search URL?__
Instead of making the client construct the URL to the search service, it would be more REST/HATEOAS to return it so the client just follows their nose.  It bulks out the response a bit, but would compress very well over the wire.
{: .warning}

The example request above might generate the following response:

{% highlight json %}
{
  "@context": "http://iiif.io/api/search/{{ page.major }}/context.json",
  "@id": "http://example.org/service/identifier/autocomplete?q=bir&motivation=painting",
  "@type": "search:TermList",
  "ignored": ["box"],
  "terms": [
    {
      "match": "bird", 
      "total": 15,
      "search": "http://example.org/service/identifier/search?q=bird&motivation=painting"
    },
    {
      "match": "biro", 
      "total": 3,
      "search": "http://example.org/service/identifier/search?q=biro&motivation=painting"
    },
    // ... or just ...
    {"match": "birth", "total": 9},
    {"match": "birthday", "total": 21}
  ]
}
{% endhighlight %}

Semantic Tags or other URIs might also have labels to display to the user, instead of the full URI:

{% highlight json %}
{
  "@context": "http://iiif.io/api/search/{{ page.major }}/context.json",
  "@id": "http://example.org/service/identifier/autocomplete?q=http://semtag.example.org/tag/b&motivation=tagging",
  "ignored": ["box"],
  "terms": [
    {
      "match": "http://semtag.example.org/tag/bird", 
      "total": 15, 
      "label": "bird"},
    {
      "match": "http://semtag.example.org/tag/biro", 
      "total": 3,
      "label": "biro"
    }
  ]
}
{% endhighlight %}


__Properties__

| Field | Definition |
| ----- | ---------- |
| terms | An alphabetically ordered list of terms|
| Term  | A term that has a value, and may have other attributes such as count |
| value | The string form of the term. Language may be associated with the value |
| count | The number of times that the term appears in the dataset |
{: .api-table}

## 5. Service Descriptions

In the Presentation API, objects such as Manifests, Ranges, Layers or even Canvases may have search services associated with them.



The auto-complete service is related specifically to a search service, and is thus referenced from within the search service's description.



## Appendices

### A. Parameter Requirements

| Parameter | Required in Request | Required in Search | Required in Autocomplete |
| --------- | ------------------- | ------------------ | ------------------------ |
| `q`       | mandatory | mandatory | mandatory |
| `motivation` | optional | recommended | mandatory |
| `date`    | optional | optional | optional |
| `uri`     | optional | optional | optional |
| `box`    | optional | optional | optional |
| `min`     | optional | n/a | optional |
{: .api-table}

### B. Versioning

This specification follows [Semantic Versioning][semver]. See the note [Versioning of APIs][versioning] for details regarding how this is implemented.

### C. Acknowledgements

The production of this document was generously supported by a grant from the [Andrew W. Mellon Foundation][mellon].

Many thanks to the members of the [IIIF][iiif-community] for their continuous engagement, innovative ideas and feedback.

### D. Change Log

| Date       | Description                                        |
| ---------- | -------------------------------------------------- |
| 2015-07-20 | Version 0.9 (Trip Life) draft                        |
{: .api-table}


[cc-by]: http://creativecommons.org/licenses/by/4.0/ "Creative Commons &mdash; Attribution 4.0 International"
[iiif-discuss]: mailto:iiif-discuss@googlegroups.com "Email Discussion List"
[versioning]: /api/annex/notes/semver.html "Versioning of APIs"
[mellon]: http://www.mellon.org/ "The Andrew W. Mellon Foundation"
[semver]: http://semver.org/spec/v2.0.0.html "Semantic Versioning 2.0.0"
[iiif-community]: /community.html "IIIF Community"
[stable-version]: http://iiif.io/api/image/{{ site.search_api.latest.major }}.{{ site.search_api.latest.minor }}/ "Stable Version"

[image-api]: /api/image/{{ site.image_api.latest.major }}.{{ site.image_api.latest.minor }}/ "Image API"
[openanno]: http://www.openannotation.org/spec/core/ "Open Annotation"
[prezi-api]: /api/presentation/{{ site.presentation_api.latest.major }}.{{ site.presentation_api.latest.minor }}/ "Presentation API"
[rfc-2119]: http://tools.ietf.org/html/rfc2119
[service-annex]: /api/annex/services/
[prezi-annolist]: /api/presentation/{{ site.presentation_api.latest.major }}.{{ site.presentation_api.latest.minor }}/index.html#other-content-resources 
[prezi-layer]: /api/presentation/{{ site.presentation_api.latest.major }}.{{ site.presentation_api.latest.minor }}/index.html#Layers

[icon-req]: /img/metadata-api/required.png "Required"
[icon-recc]: /img/metadata-api/recommended.png "Recommended"
[icon-opt]: /img/metadata-api/optional.png "Optional"
[icon-na]: /img/metadata-api/not_allowed.png "Not allowed"

{% include acronyms.md %}
