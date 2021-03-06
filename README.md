# Introduction
The schema.org vocabulary is a de facto industrial standard for creating semantically annotated data. The vocabulary with its actions subset that allows to describe not only entities on the web, but also actions that can be taken on them. This specification puts schema.org actions into a web services perspective by restricting and extending it with the help of the [domain specification](#domain-specification) process for the annotation of HTTP APIs. 


# Domain Specification

A domain specification is a process to create a domain specific pattern, which is an extended subset of schema.org. schema.org is a large vocabulary that covers several domains in a shallow way. To create a domain specific pattern, the domain specification process applies an operator on schema.org to remove the types and properties that are not relevant for the given domain, defines local properties on the remaining types and applies additional constraints on the ranges and values of remaining properties. The syntax of domain specification operator is a subset of [Shapes Constraint Language (SHACL)](https://www.w3.org/TR/shacl/). The semantics is _slightly_ [different](https://drive.google.com/file/d/1BmAikrlw8lRMZWrXT1sFfHZUEPruEbcy/view). 

![ds process](_media/ds-process.svg  ':class=figure' )

<center><span class="caption">The domain specification process.</span></center>


!> **_Relationship between SHACL and Domain Specifications_**: SHACL is a language that is built around the notion of *shape* in order to verify RDF graphs. A shape is either a node shape that applies constraints on nodes in an RDF graph, or a property shape that does the same to properties. In principle, we use SHACL as is, but we apply stricter syntax rules in terms of which constraint components can be applied on which type of shapes and how the shapes are interpreted (semantics). For instance, multiple target definitions are interpreted as disjunction in SHACL, but as a conjunction in domain specification approach.

?> _TODO_ add an example domain specification 


# Schema.org Actions

Schema.org contains an [Action](https://schema.org/Action) type to provide a mechanism to define actions that can be take on entities. In the context of schema.org, the actions are quite generic. For example, the [Action](https://schema.org/Action) type includes properties like startTime and endTime to describe the time span an action occurred or the location property to describe where the action took place. We restrict and extend the properties defined for the [Action](https://schema.org/Action) type and consequently its subtypes in order to make them more specific to a Web API annotation task.
The table below shows the properties of Action type we use for the Web API annotations.

|    Property    |            Range            | Description |
|:--------------:|:---------------------------:|:-----------:|
|  actionStatus  |       ActionStatusType      |             |
| authentication | AuthenticationSpecification |             |
|     object     |            Thing            |             |
|     result     |            Thing            |             |
|     target     |          EntryPoint         |             |
|      error     |            Error            |             |


# Conceptualization

A schema.org action can have one of four statuses depending on its state. We map different stages of the client-API interaction to schema.org action statuses.

**Resource Description**: The description of an operation on a specific resource. This action is in **PotentialActionStatus**. 

?> See [Appendix](#appendix) for the domain specification operator that defines the Action pattern for resource descriptions.

**Request**: An action instance with all required parameters (and possible optional) parameters are filled. This action is in **ActiveActionStatus**.

**Response**: A (lifted) response from the server to a request. This action instance is in **CompletedActionStatus**.

# Specification


## Resource Description

A resource description is an extended SHACL node shape. It consists of the following elements:
### ID
Required. The URI of the resource description.
### Target Class
Required. The type of the operation described. A subtype of schema:Action (e.g., BuyAction).

 

## Resource Linking

In many cases, an operation on a resource of an API requires the response of another operation on another resource in the same or a different API. To support such cases, we allow linking of property shapes as the value of a property whose value needs to be retrieved from the result of another action. There are two ways to do this linking. 

The simplest way is when the entire result object is needed as an input. This can be accomplished by just linking a node shape. See the example below for an action to search bus stops.

**BusStopSearch.jsonld**
```json

{
    "@context": {"@vocab": "http://schema.org/", "smtfy": "https://actions.semantify.it/vocab/"},
    "@type": "SearchAction",
    "@id": "http://actions.semantify.it/actions/30cc4a54-6391-4ec9-aaef-d11b72c752e2"
    "name": "Search for bus stops",
    "actionStatus": "PotentialAction",
    "target": {
      "@type": "EntryPoint",
      "urlTemplate": "http://actions.semantify.it/api/vao/busstops",
      "httpMethod": "GET",
      "encodingType": "application/ld+json",
      "contentType": "application/ld+json"
    },
    "smtfy:authentication": {
      "@type": "smtfy:TokenAuthentication",
      "value-input": "required"
    },
    "query-input": "required",
    "result": {
      "@type": "BusStop",
      "name-output": "required",
      "geo": {
        "@type": "GeoCoordinates",
        "latitude-output": "required",
        "longitude-output": "required"
      }
    }
  }

```
In this example, the operation takes a query string (query-input property), and returns a list of BusStop with their names and geo-coordinates. Note that, the Action itself is identified with a URI
> http://actions.semantify.it/actions/30cc4a54-6391-4ec9-aaef-d11b72c752e2

Below there is an example action for searching bus connections. We link the action above to the input properties **fromLocation** and **toLocation**. 

```json
{
    "@context": {
        "@vocab": "http://schema.org/",
        "sh": "https://raw.githubusercontent.com/w3c/shacl/master/.../shacl.context.ld.json",
        "smtfy": "https://actions.semantify.it/ns/",
        "smtfya": "https://actions.semantify.it/actions/"
    },
    "@type": "SearchAction",
    "object": {
        "@type": "Trip, TravelAction",
        "fromLocation-input": "required hasValueFrom={action: smtfy:30cc4a54-6391-4ec9-aaef-d11b72c752e2}",
        "toLocation-input": "required hasValueFrom={action: smtfy:30cc4a54-6391-4ec9-aaef-d11b72c752e2}",
        "departureTime-input": "required ",
        "arrivalTime-input": "optional"
    },
    "result": 
    {
        "@type": ["Trip","TravelAction"],
        "fromlocation": {
          "@type": "Place",
          "name-output": "required"
        },
        "toLocation": {
          "@type": "Place",
          "name-output": "required"
        }
        "departureTime-output": "required",
        "arrivalTime-output": "required",
        "subTrip": {
            "@type": [
                "Trip",
                "TravelAction"
            ],
            "sh:shape": [
                "smtfy:FromLocationShapeOut",
                "smtfy:FromLocationShapeOut"
            ],
            "departureTime-output": "required",
            "arrivalTime-output": "required"
        }
    },
    "target": {
        "@type": "EntryPoint",
        "urltemplate": "http://actions.semantify.it/zpwv/...",
        "httpMethod": "POST"
    }
}

```

An action processing client should first make the necessary request to the linked action (i.e. bus stop search), and then complete the main action (i.e. search connections). 

## Lifting and Grounding

### Grounding (Request Mapping)

?> _TODO_ will be done with an extended version of Xquery and Handlebars.

### Lifting (Response Mapping)

The example below shows a response mapping. An Offer is returned as a result of a SearchAction. A potential action is attached to the Offer instance,with the result of a smtfy:link function as its object. This function returns an action (a SHACL node shape with targetClass schema:Action and its subtypes) with the given parameters filled with the specified value (schema:identifier is filled with the ID of the returned Offer).

?> _TODO_ A short tip about potential actions.

?> _TODO_ Check the Function Ontology from imec Gent for describing functions.

!> **Thibault** please check the syntax. 

```yaml

prefixes:
  schema: "http://schema.org/"

mappings:
  action:
    sources:
      - ['input~xpath', '/Result']
    po:
      - [a, schema:SearchAction]
      - [schema:actionStatus, 'CompletedActionStatus']
      - [schema:result, {mapping: offer}]

  offer:
    sources:
      - ['input~xpath', '/Result/Offer']
    po:
      - [a, schema:Offer]
      - [schema:name, $(@name)]
      - [schema:price, $(@price)]
      - [schema:identifier, $(@id)]
      - [schema:potentialAction, {function: buyActionFn}]

functions:
  buyActionFn:
    - function: smtfy:Link
    parameters:
      - [smtfy:actionURI, smtfy:BuyAction2455]
      - [schema:identifier, $(@id)]
  

```


# Use Case: Dialog Generation from API Annotations

# Appendix 
## Abstract Syntax

TBD

## Domain Specification for Resource Description Annotations