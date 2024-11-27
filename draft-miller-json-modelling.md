---
###
# Internet-Draft Markdown Template
#
# Rename this file from draft-todo-yourname-protocol.md to get started.
# Draft name format is "draft-<yourname>-<workgroup>-<name>.md".
#
# For initial setup, you only need to edit the first block of fields.
# Only "title" needs to be changed; delete "abbrev" if your title is short.
# Any other content can be edited, but be careful not to introduce errors.
# Some fields will be set automatically during setup if they are unchanged.
#
# Don't include "-00" or "-latest" in the filename.
# Labels in the form draft-<yourname>-<workgroup>-<name>-latest are used by
# the tools to refer to the current version; see "docname" for example.
#
# This template uses kramdown-rfc: https://github.com/cabo/kramdown-rfc
# You can replace the entire file if you prefer a different format.
# Change the file extension to match the format (.xml for XML, etc...)
#
###
title: "JSON Modelling"
category: info

docname: draft-miller-json-modelling-latest
submissiontype: IETF  # also: "independent", "editorial", "IAB", or "IRTF"
number:
date:
consensus: true
v: 3
area: "Applications"
workgroup: "JavaScript Object Notation"
keyword:
 - JSON Schema
 - OpenAPI
venue:
  group: "JavaScript Object Notation"
  type: "Working Group"
  mail: "json@ietf.org"
  arch: "https://mailarchive.ietf.org/arch/browse/json"
  github: "darrelmiller/json-modelling"
  latest: "https://darrelmiller.github.io/json-modelling/draft-miller-json-modelling.html"

author:
 -

    fullname: Darrel Miller
    organization: Microsoft
    email: darrel@tavis.ca

normative:

informative:

--- abstract

This document defines a normalization process and semantic profile, simplifying the use of JSON Schemas for JSON modelling.

--- middle

# Introduction

JSON Schema is widely used as a JSON modelling language for describing JSON instances and language types, despite it having a primary goal of being a JSON validation language. There is not a direct mapping between a JSON Schema and the shape of the JSON it validates. The same JSON structure can be validated using different JSON Schemas. This characteristic makes it complex to infer a JSON instance, or a corresponding language type from a JSON Schema.

JSON Schema Intent: JSON Instance + JSON Schema => true/false

Common Usage Scenario: JSON Schema => JSON Instance or Type.

JSON Schema is commonly used in HTTP API documentation to describe the JSON instances that are included in HTTP request/response content.  The JSON Schemas are also used by client library generators to project language types that JSON instances are (de)serialized to/from.

JSON Schema is often included in prompts for Large Language Models to enable the LLM to produce JSON instances that contain values inferred from natural language content in the prompt.

JSON Schema is used by JSON editing tools to provide autocomplete experiences to assist users in constructing a JSON instance.

JSON Schemas can be created that only validate some aspects of a JSON instance. They can also include keywords that are only applicable if a JSON value is a specific type. This ambiguity causes implementors that are trying to infer an instance or type from a schema to make certain assumptions. Different implementations have made different assumptions and have created confusion for authors of JSON Schemas as to what are the best practices.

This document describes a process that normalizes JSON Schema documents into a form that more directly maps to corresponding language types. It also assigns different semantics for a set of JSON Schema keywords that must be processed differently to produce design time artifacts.

# Conventions and Definitions

{::boilerplate bcp14-tagged}

# Terminology

Root schema: The top level JSON schema that is being considered.

subschema: A JSON schema that is contained in a root schema.

Anonymous schema: A JSON schema that has no user provided identifier. i.e. it is not referenced with a $ref and it doesn't have an $id.

# Normalization

JSON Schema is effectively a graph of boolean expressions that are evaluated against JSON instances. The nature of boolean expressions is that they can be expressed in many different ways that are logically equivalent. Through boolean algebra operations, the JSON Schema document can be refactored to produce a canonical form that is more suitable for modelling types.

As an example, the following two schemas are logically equivalent but are expressed with different structures:

```json
{
  "type": "object",
  "properties": {
    "id": { "type": "string" },
    "name": { "type": "string" },
    "age": { "type": "integer" }
  }
}
```

```json
{
    "allOf": [
      "type": "object",
      "properties": {
        "id": { "type": "string" }
      }
  ],
  "type": "object",
  "properties": {
    "name": { "type": "string" },
    "age": { "type": "integer" }
  }
}
```

### Eliminating anonymous allOfs

The following demonstrates the boolean expression refactoring that can be used to eliminate anonymous schemas in allOfs.

`x = (a & b & c) & (d & e & f) & (g & h & i)`
simplifes to
`x = a & b & c & d & e & f & g & h & i`
assuming there are no contradicting keywords.

It is only necessary to eliminate anonymous schemas as schemas with $id or using $ref can be projected into a type that is used as an inheritance base type or interface on the root schema.

This technique can also help when there are keywords in the root schema.

`x = a & b & (c & d & e)`
simplifies to
`x = a & b & c & d & e`

### Simplifying oneOf/AnyOfs

Schemas that contain a `oneOf` or `anyOf` keyword and other keywords can have the top level keywords pushed down into the `oneOf/allOf` subschemas. This enables the use of union types to represent `oneOfs` and a type with multiple interfaces to support `anyOfs`.

`x = a & b & (c | d | e)`
is equivalent to
`x = a & b & c | a & b & d | a & b & e`


## Type Inference

All schemas need a type keyword that only has a single type value, other than null, to be able to map the schema to a language type. If the type keyword is missing, it is inferred based on the presence of properties/items.  This only happens after processing of allOf/anyOf/oneOf keywords.
In the scenario where the keywords in schema apply to different types, the schema can be refactored to a `oneOf` where each subschema contains only the keywords that apply to that type.

e.g.

```json
{
  "type": ["object", "array"],
  "properties": {
    "name": { "type": "string" },
    "age": { "type": "integer" }
  },
  "items": {
    "type": "string"
  }
}
```

becomes

```json
{
  "oneOf": [
    {
      "type": "object",
      "properties": {
        "name": { "type": "string" },
        "age": { "type": "integer" }
      }
    },
    {
      "type": "array",
      "items": {
        "type": "string"
      }
    }
  ]
}
```

## Inline vs references

If you want to model a schema as a type, either reference it or give it a title.

## allOf


* allOfs that contain inlined schemas without titles are flattened.
* additionalProperties of inlined schemas are ignored

## oneOf

Schemas that only contain oneOfs with titles or

## additionalProperties as an object

## patternProperties

# Constraints

Keywords that do not match the inferred type are ignored for modelling.

## Keywords alongside $refs

## Keywords with unclear modelling semantics

### anyOf/oneOf

anyOf/oneOf keywords are treated identically for modelling purposes.

anyOf/oneOf without titles or $refs are considered not considered for modelling purposes.

<aside>The challenge comes when populating instances of the model.  oneOf is fine, but anyOf could match to multiple types. So, how can you deserialize the data into object instances?</aside>

## Keywords that depend on the instance

The following keywords depend on the instance document and therefore SHOULD NOT be used to define the design time model:

* if/then/else
* unevaluatedProperties
* dependentSchemas

<aside> Note that these keywords could be processed to augment the model to accomodate any of the possible outcomes. This is problematic when the outcomes are contradictory. TBD </aside>

# Security Considerations

TODO Security

# IANA Considerations

This document has no IANA actions.

--- back

# Acknowledgments

{:numbered="false"}

TODO acknowledge.

## Appendix - Processing aglorithm

- Recursively normalize anonymous sub schemas contained in allOfs by moving the keywords to the parent schema.
- Where schemas contain oneOf or anyOf keywords, push keywords and remaining allOfs in the root schema into each of the sub schemas.
- Schemas with multiple oneOf keywords should merge the array of oneOfs (this is lossy)
- [At this point all schemas should only contain a single oneOf or allOf(s) of named schemas]
- Infer Type keyword where present
- If schema supports multiple types (not including null) then create a oneOf for each type.


## Appendix - Evaluation methodology

JSON Schema in prompt -> LLM -> JSON Instance -> Validate against JSON Schema -> Deserialize  -> Validate properties
Repeat using normalized JSON Schema
Translate to CDDL and repeat with CDDL in prompt

