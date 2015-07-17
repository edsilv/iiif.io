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

_Copyright © 2015 Editors and contributors. Published by the IIIF under the [CC-BY][cc-by] license._

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

## Table of Contents
{:.no_toc}

* Table of Discontent (will be replaced by macro)
{:toc}

## 1. Introduction

In the IIIF [Presentation API][prezi-api], content is brought together from distributed systems via annotations.  That content might be images, often with a IIIF [Image API][image-api] service to access them, audio, video, rich or plain text, or anything else.  In a vibrant and dynamic system, that content can come from many sources and be rich, varied and abundant.  Of that list, textual content lends itself to being searched, either as the transcription, translation or edition of the intellectual content, or commentary, description, tagging or other annotations about the resource.  

This specification lays out the interoperability mechanism for performing these searches within the IIIF context.  The scope of the specification is searching only textual annotation content within a single IIIF resource, such as a Manifest or Range.  Every effort is made to keep the interaction as consistent with existing IIIF patterns as possible.

Please send feedback to [iiif-discuss@googlegroups.com][iiif-discuss]

### 1.1. Use Cases

Use cases for being able to search the annotations within the Presentation API include:

 * Searching OCR generated text to find words or phrases within a book, newspaper or other primarily textual content
 * Searching transcribed content, provided by crowd-sourcing or transformation of scholarly output
 * Searching multiple streams of content, such as the translation or edition, rather than the raw transcription of the content, to jump to the appropriate part of an object
 * Searching on sections of text, such as defined chapters or articles.
 * Searching for user provided commentary about the resource, either as a discovery mechanism for the resource or for the discussion.
 * Discovering similar sections of text to compare either the content or the object

User interface considerations include highlight matching words in the display, providing a heatmap of where the matches occur within the object, and providing a mechanism to jump between points within the object.

### 1.2. Terminology

The key words _MUST_, _MUST NOT_, _REQUIRED_, _SHALL_, _SHALL NOT_, _SHOULD_, _SHOULD NOT_, _RECOMMENDED_, _MAY_, and _OPTIONAL_ in this document are to be interpreted as described in [RFC 2119][rfc-2119].



## 2. Overview

The IIIF Presentation API provides just enough information to a viewer so that it can present the images and other content to the user in a rich and understandable way.  Those content resources may have textual annotations associated with them.  Annotations may also be associated with the structural components of the Presentation API, such as the manifest itself, sequences, ranges, and layers.  Further, annotations can be replied to by annotating them to form a threaded discussion about the commentary, transcription, edition or translation.

Annotations are typically made available to viewing applications in an Annotation List, where all of the annotations in the list target the same resource, or part of it.  Where known, these lists can be directly referenced from the manifest document to allow clients to simply follow the link to retrieve them.  For fixed, curated content, this is an appropriate method to discover them, as the annotations do not frequently change, nor are they potentially distributed amongst multiple servers.

However this is less useful for comment-style annotations, crowd-sourced or distributed transcriptions, corrections to automated OCR transcription, and similar, as the annotations may be in constant flux.  Further, being able to quickly discover individual annotations without stepping through all of the views of an object is essential for a reasonable user experience.  

Beyond the ability to search for words or phrases, users find it helpful to have suggestions for what terms they should be searching for.  This facility is often called auto-complete or type-ahead, and within the context of a single object can provide insight into the language and content.  The auto-complete service is associated with a search service into which the terms can be fed as part of a query.

This specification details how, within the context of IIIF, search and auto-complete services should be implemented such that they are easy and consistent to interact with from viewing applications.

## 3. Search

The search service takes a query, including at least one search term or uri, and optionally filtering further by date the annotation was last created or modified, or the motivation for the annotation as to whether it is painting a resource on to a canvas, or other types of annotation such as comments about a target resource. 

### 3.1. Request

The search request is made to a service endpoint that is related to a particular IIIF Presentation API resource.  The URIs for the endpoints must be different to allow the client to search within a specific scope.  The URI is given in a service description related to the resource.

To perform a search, the client issues an HTTP GET request to the endpoint, with query parameters to specify the search terms.

__Search Query Parameters__

| Parameter  | Definition | 
| ---------  | ---------- |
| q          | The search terms to search in textual bodies.  The semantics of multiple, space separated terms is server implementation dependent; they may be treated as a phrase, that all are required, or that at least one is required. |
| motivation | The motivation of the annotation. Acceptable values are given below, and the default if not present is all motivations.  Support for this field is not required. |
| date       | A date range in ISO8601 format: `YYYY-MM-DDThh:mm:ssZ/YYYY-MM-DDThh:mm:ssZ` Dates must be expressed in UTC (and must be given in the `Z` based format) The default if not supplied is for all dates. Support for this field is not required. |
| uri        | A space separated list of URIs to filter the annotations by.  The URIs may be present in any of the fields of the annotation other than target, including the identity of users, clients, body as semantic tag or content resource. Support for this field is not required. |
| box        | A box specified as `x,y,w,h` that limits the area of a Canvas that the annotation must overlap with.  Annotations that target the entire Canvas are considered to overlap with all possible bounding boxes.  The default if not supplied is the entire canvas. |
{: .api-table}

The accepted values for the motivation parameter are:

| Motivation | Definition |
| ---------- | ---------- |
| `all`          | All annotations (default, if not supplied) |
| `painting`     | Only annotations with the `sc:painting` motivation |
| `non-painting` | Annotations with any motivation other than `sc:painting` |
| `commenting`   | Annotations with the `oa:commenting` motivation |
| `describing`   | Annotations with the `oa:describing` motivation |
| `tagging`      | Annotations with the `oa:tagging` motivation |
{: .api-table}

Other motivations are possible, and the full list from the [Open Annotation][openanno] specification should be available by dropping the "oa:" prefix.

Example request:

```
http://example.org/service/identifier/search?q=bird&motivation=painting
```
{: .urltemplate}

Would search for annotations with the word "bird" in their textual content, and have the motivation of `painting`.

### 3.2. Response

The response from the server is an Annotation List, following the format from the Presentation API with a few additional features. 

The search results are returned as Annotations, in the regular IIIF syntax. Note that the annotations can come from multiple canvases, rather than the default situation from the Presentation API where all of the annotations target a single canvas.

For very long lists of annnotations, the server may choose to divide the response into multiple sections, often called pages.  Each page of the response can refer to the previous and next page, to allow the client to traverse the entire set.  The next page of results is referenced in a `next` field of the AnnotationList, and the previous in a `prev` field.

Each paged AnnotationList references a Layer.  The Layer does not need to have a URI, and must be included in every AnnotationList in the set of pages.  The Layer has a "total" property which is the total number of hits generated by the query.

For textual searches, it is also useful to include the text before the matching word or phrase, and the text after it, but still within the annotation's content. This can be achieved with the `prefix` (text before), `suffix` (text after) and `exact` (matched text) fields on the resource of the annotation.  The `exact` field is useful when stemming or other transformations of the query term have been applied before matching against the index.

The annotations may also include references to the structure or structures that the target is found within.  At least the URI and the type of the including resource should be given, and a label is also especially recommended if there are multiple resources of the same type.

Example response with a the first annotation from a total of 125:

{% highlight json %}
{
  "@context":"http://iiif.io/api/search/1/context.json",
  "@id":"http://example.org/service/identifier/search?q=bird&motivation=painting",
  "@type":"sc:AnnotationList",
  "within": {
    "@type": "sc:Layer", 
    "total":125
  },
  "next": "http://example.org/service/identifier/search?q=bird&motivation=painting&page=2",
  "last": "http://example.org/service/identifier/search?q=bird&motivation=painting&page=125",
  "startIndex": 0,

  "resources": [
    {
      "@type":"oa:Annotation",
      "motivation":"sc:painting",
      "resource":{
        "@type":"dctypes:Text",
        "chars": "There are two birds in the bush",
        "prefix": "There are two",
        "exact": "birds",
        "suffix": "in the bush"
      },
      "on": {
        "@id": "http://www.example.org/iiif/identifier/canvas/p1#xywh=10,10,400,80",
        "within": { 
          "@id": "http://www.example.org/iiif/identifier/manifest",
          "@type": "sc:Manifest"
        }
	  }
    }
  ]
}
{% endhighlight %}
{: .urltemplate}

The additional fields added by the search API to the regular set of fields from the Presentation API are:

__Properties of Layers__

| Field | Definition |
| ----- | ---------- |
| total | The total number of matched annotations |
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

__Properties of Textual Annotation Bodies__

| Field | Definition |
| ----- | ---------- |
| prefix | Server determined amount of text immediately before the matched text |
| exact | The matched text |
| suffix | Server determined amount of text immediately after the matched text |
{: .api-table}

## 4. Autocomplete

The autocomplete service returns terms that can be added into the `q` parameter of the related search service, given the first characters of the term.

### 4.1. Request

The request is very similar to the search request, with one additional parameter to allow the number of occurrences of the term within the object to be constrained.  The value of the `q` parameter is the beginning characters from the term to be completed by the service.  For example, the query term of 'bir' might complete to 'bird', 'biro', 'birth', and 'birthday'.

The other parameters (`motivation`, `date` and `uri`), if supported, would refine the set of terms in the response to only ones from the annotations that match those filters.  For example, if the motivation is given as "painting", then only text from painting transcriptions will contribute to the list of terms in the response.

__Additional Query Parameters__

| Parameter | Definition |
| --------- | ---------- |
| min       | The minimum number of occurrences for a term in the index in order for it to appear within the response ; default is 1 if not present.  Support for this parameter is not required |
{: .api-table}

### 4.2. Response

The response is a list of very simple objects that include the term and its number of occurrences.  The number of terms provided is determined by the server, but may be split into pages if necessary with `next` and `prev` links. 

{% highlight json %}
{
  "@context": "http://iiif.io/api/search/1/context.json",
  "@id": "http://example.org/service/identifier/autocomplete?q=bir&motivation=painting",
  "terms": [
  	{"value": "bird", "count": 15},
  	{"value": "biro", "count": 3},
  	{"value": "birth", "count": 9},
  	{"value": "birthday", "count": 21}
  ]
}
{% endhighlight %}
{: .urltemplate}

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

{% highlight json %}
{
  "service": {
    "@id": "http://example.org/services/identifier/search",
    "profile": "http://iiif.io/api/search/1/search"
  }
}
{% endhighlight %}
{: .urltemplate}

The auto-complete service is related specifically to a search service, and is thus referenced from within the search service's description.

{% highlight json %}
{
  "service": {
    "@id": "http://example.org/services/identifier/search",
    "profile": "http://iiif.io/api/search/1/search",
    "service": {
      "@id": "http://example.org/services/identifier/autocomplete",
      "profile": "http://iiif.io/api/search/1/autocomplete"
    }
  }
}
{% endhighlight %}
{: .urltemplate}

## Appendices

### A. Requirements

Parameter Requirements

| Parameter | Required in Request | Required in Search | Required in Autocomplete |
| --------- | ------------------- | ------------------ | ------------------------ |
| `q`       | mandatory | mandatory | mandatory |
| `motivation` | optional | recommended | mandatory |
| `date`    | optional | optional | optional |
| `uri`     | optional | optional | optional |
| `box`    | optional | optional | optional |
| `min`     | optional | n/a | optional |


### B. Versioning

Starting with version 2.0, this specification follows [Semantic Versioning][semver]. See the note [Versioning of APIs][versioning] for details regarding how this is implemented.

### C. Acknowledgements

The production of this document was generously supported by a grant from the [Andrew W. Mellon Foundation][mellon].

Many thanks to the members of the [IIIF][iiif-community] for their continuous engagement, innovative ideas and feedback.

### D. Change Log

| Date       | Description                                        |
| ---------- | -------------------------------------------------- |
| 2015-07-20 | Version 0.9 (Trip Life) draft                        |


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

[icon-req]: /img/metadata-api/required.png "Required"
[icon-recc]: /img/metadata-api/recommended.png "Recommended"
[icon-opt]: /img/metadata-api/optional.png "Optional"
[icon-na]: /img/metadata-api/not_allowed.png "Not allowed"

{% include acronyms.md %}
