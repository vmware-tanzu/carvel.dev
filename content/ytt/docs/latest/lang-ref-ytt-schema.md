---
title: Schema module
---

## Overview

`ytt`'s Schema feature provides a way to verify data value YAML structures and values with the help of annotations.
A schema allows you to provide default data values, and verify that each data value file provided is type checked and validated against the schema and guaranteed to be proper.

Using a schema guarantees to templates that all data values exist and are of the correct type. The alleviates templates from doing existence and type checks, themselves.

---
## Defining Schema
Configuration Authors establish a Schema by capturing the structure in a YAML document with `#@schema/match` at the top:
```yaml
#@schema/match data_values=True
---
#! Schema contents
```
- `data_values` (`bool`) — whether this schema is applicable to data values. If this
  is not set to `True`, this results in an error.

Notes:
- Schema must be defined _before_ any other files containing YAML can be processed. Therefore, a file containing a Schema document must not contain other kinds of documents.
- files containing Schema documents will be detected among input files (i.e. those specified by `-f`).
- the first file containing a Schema document establishes the "base". Subsequent files containing Schema documents are overlayed onto the "base" in the order they appear (identically to how Data Values files are processed).

---
## Supported Types

While YAML provides for an extendable range of types, the `ytt` Schema supports a specific set.

### Scalar and Explicit types

- `bool` — `true`, `false` (and when not strict, `yes`, `no`, `Y`, `N`, etc.)
- `float` — e.g. `0.4`
- `int` — e.g. `42`
- `null` — `null`, `~`, and when value is omitted.
- `string` — e.g. `""`, `"ConfigMap"`, `"0xbeadcafe"`
- `any` — any valid YAML value is permitted. Must be set [explicitly](#@schema/type).

### Array type

A YAML sequence, each item must be of the specified `type`.

_Example: Array of strings_
```yaml
- ""
``` 
Each array item must contain only a string

_Example: Array of maps_
```yaml
- key1: value1
  key2: value2
```
An array must contain only map instances of the same shape.

### Map type

A YAML mapping that must contain exactly one item of each given key, whose value must be of the type specified.

_Example: Map with string value_
```yaml
key1: ""
```
A map with `key1` must have a string value.

---
## Inferring Types

Structure in Schema is largely expressed by example rather than by description. Types are — by default — inferred based on the values given. The Configuration Author can override these defaults using the [`@schema/type`](#schematype) annotation.

```yaml
#@schema/match data_values=True
---
system_domain: ""

load_balancer:
  enable: true
  static_ip: ""

app_domains:
- ""

databases:
- name: ""
  adapter: postgresql
  host: ""
  port: 5432
  user: admin
  secretRef:
    name: ""
```

where:
- Data Values must consist of four top-level map items:
    - `system_domain` — whose value is a string,
    - `load_balancer` — whose value is a map containing two items:
        - `enable` — whose value is a boolean
        - `static_ip` — whose value is a string
    - `app_domains` — whose value is an array of strings (and only strings)
    - `databases` — whose values is an array of maps of a specific shape: six (6) items:
        - `name` — containing a string
        - `adapter` — containing a string
        - `host` — also holds a string
        - `port` — holds an integer
        - `user` — a string
        - `secretRef` — containing a map with one key:
            - `name` — containing a string

## Inferring Default Values

The exact values that the Configuration Author specifies in Schema are the defaults (with one notable exception — arrays — which are [detailed below](#inferringdefaultsforarrays)). This ensures that all nodes are initialized with values, with reasonable defaults.

Configuration Consumers should specify only those values which vary from the default, reducing risk of errors and generally making configuration easier to maintain.

### Inferring Defaults for Scalars

When a scalar value is specified, it is simply the default.  From the example, above:

- `system_domain` is an empty string, by default.
- `load_balancer.enable` is true by default.
- `databases[].adapter` is `"postgres"`, by default.

### Inferring Defaults for Mappings

The set of items of a map are its default: any item missing will be automatically inserted with _its_ defaults.

- if `load_balancer` is omitted from supplied Data Values, it is inserted:
    - `load_balancer.enabled` is `true`
    - `load_balancer.static_ip` is `""`
- if `load_balancer` is specified with only `static_ip`, `enabled` is `true`.

When the Configuration Consumer supplies an empty map (i.e. `{}`), all items are defaulted:

For example,

`load_balancer: {}`

is defaulted to:

```yaml
load_balancer:
  enable: true
  static_ip: ""
```

### Inferring Defaults for Arrays

The default value for all arrays is an empty array. Arrays are the only type for which defaults are **not** inferred.

From the example, above,

- `app_domains`, if not otherwise specified is `[]`
- `databases`, if not otherwise specified is `[]`

When values for an array _are_ provided by the Configuration Consumer, each item is defaulted based on the type specified in the schema.

Focusing on just `databases`, if the Configuration Consumer supplies this Data Value:

```yaml=
#@data/values
---
databases:
- name: uaa
- name: capi
  host: capi-db.svc.cluster.local
  secretRef:
    name: capi-db-credentials
- {}
```

The resulting Data Values (i.e. that are supplied to templates), would be:

```yaml=
databases:
- name: uaa
  adapter: postgresql
  host: ""
  port: 5432
  user: admin
  secretRef:
    name: ""
- name: capi
  adapter: postgresql
  host: capi-db.svc.cluster.local
  port: 5432
  user: admin
  secretRef:
    name: capi-db-credentials
- name: ""
  adapter: postgresql
  host: ""
  port: 5432
  user: admin
  secretRef:
    name: ""
```

## Defining Schema Explicitly

Configuration Authors may override inferred typing and defaults through annotations.

### @schema/type

Explicitly sets the type of the annotated node.

```yaml
@schema/type any=True
```

- `any` (`bool`) — whether or not any and all types are permitted on this node (and its children)
    - to avoid confusion, this keyword argument is mutually exclusive to all other arguments; if this argument is
      provided along with others, an error results.
__

### @schema/nullable

Extends the type of the node to include "null" _and_ to be "null" by default.

```yaml
@schema/nullable
```


where `name` can hold a string, but also be null. `name` is `null` by default.

_Example: Nullable map and string_ 

```yaml
#@schema/nullable
aws:
  username: ""
  password: ""

#@schema/nullable
name: ""
```

where `aws` and `name` are null by default. When set in a data value file `aws` will be a map with the two items, and `name` will be a string.
