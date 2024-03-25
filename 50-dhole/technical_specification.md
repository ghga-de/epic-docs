# Rewriting existing metldata transformations
**Epic Type:** Implementation Epic

Epic planning and implementation follow the
[Epic Planning and Marathon SOP](https://docs.ghga-dev.de/main/sops/sop001_epic_planning.html).

**Attention: Please do not put any confidential content here.**

## Scope
### Outline:
The aim is to rewrite all existing metldata transformations to use schemapack.


### Included/Required:
- add possibility to use custom embedding profile to configure the denormalization
  in schemapack
- migrating the already schemapack-based transformations to spec version 0.2.0
  (see https://github.com/ghga-de/metldata/tree/poc/src/metldata/schemapack_/builtin_transformations):
  - delete_properties
  - infer_relations
- re-implement the following LinkML-based transformations to use schemapack
  (see https://github.com/ghga-de/metldata/tree/poc/src/metldata/builtin_transformations):
  - merge_slots
- re-evaluate and potentially refactor the following transformations:
  - aggregate


### Not included:
- Reimplementation of the custom_ebeddings transformation is not required since it will already
  covered by builtin functionality of schemapack
- Reimplementation of the add_accessions transformation is postponed since it might be
  moved handled by the submission store and not be implemented as transformation.
- A full reimplementation of the GHGA transformation workflow (this depends on other changes to
  be in place first and would make this epic dependent on other lines of work).
- The normalize_model transformation is not needed anymore

## Additional Details:

### Embedding Profile to Configure Denormalization:

This helps to control which relations (and relations of relations) will be embedded and
which won't. It can even be used to deal with circular dependencies that would otherwise
lead to an exception when denomalizing.

Here is an example:

Given the following rooted schemapack:
```yaml
schemapack: 0.2.0
description: A schema used to describe relationships between team members.
classes:
  Person:
    id:
      propertyName: name
    content: ../../content_schemas/AnyObject.schema.json` # Any object allowed
    relations:
      teammates:
        targetClass: Person # refering to itself
        mandatory:
          origin: false
          target: false
        multiple:
          origin: true
          target: true
      manager:
        targetClass: Person # refering to itself
        mandatory:
          origin: false
          target: false
        multiple:
          origin: true
          target: false

  rootClass: Person
```

There is the following datapack:
```yaml
datapack: 0.2.0
resources:
    Person:
        Alice:
            content: {}
            relations:
                teammates:
                    - Bob
                    - Charlie
                manager: Dave
        Bob:
            content: {}
            relations:
                teammates:
                    - Alice
                    - Charlie
                manager: Dave
        Charlie:
            content: {}
            relations:
                teammates:
                    - Alice
                    - Bob
                manager: Dave
        Dave:
            content: {}
            relations:
                teammates: []
                manager: Eve
        Eve: # the boss of Dave
            content: {}
            relations:
                teammates: []
                manager: null
rootResource: Alice
```

Running a denormalization without configuration would not work because there are
circular dependencies between Alice, Bob, and Charlie regarding the teammates relation.

To resolve that, the following embedding profile can be used to configure the
denormalization process:

```yaml
teammates:
  # do embed teammates, however, for each embedded teammate the following embedding
  # restrictions apply:
  teammates: false # do not embed teammates of teammates
  manager: false # do not embed managers of teammates
manager: true # do embed the managers and all the nested relations that the manager
              # might have
```

The outcome would be the following denormalized data:
```yaml
name: Alice
teammates:
  - name: Bob
    teammates:
      - Alice
      - Charlie
    manager:
      - Dave
  - name: Charlie
    teammates:
      - Alice
      - Bob
    manager:
      - Dave
manager:
  name: Dave
  teammates: []
  manager:
    name: Eve
    teammates: []
    manager: null
```

### Immutability of Datapack and Schemapack Objects:
Currently, the pydantic models for interacting with DataPack and SchemaPack
definitions are not fully frozen, yet, however in the future they will be.
Thus, while re-implementing the transformations, modifications to a
DataPack or SchemaPack object should never be done in place but
using methods that would also work on frozen pydantic models (e.g.
the `model_copy(update={...})` method). This is also true for the already
migrated transformation.

### Refactor merge_slots transformation:
The original merge_slots transformation should be replaced by two new transformations:

A. For merging content properties:
- nested properties (of nested JSON objects) must be supported
- If the source properties are not of the same type and the `assume_same_type` argument
  (see config example below) is set to `false`, a union type is created (AnyOf).
- The new property is always a list even if the source properties were not multivalued.
- a config example might look like this:
  ```yaml
  merge_content_properties:
    ClassA:
      # merged properties that shall be created in ClassA:
      new_property_x:
        source_properties:
          - old_property_a
          - old_property_b
        deduplicate: true # the default, all values are kept even if they occur
                          # multiple times accross the source properties
        assume_same_type: true # the default, raises an exception if the source
                               # properties do not have the same type
      old_property_c.new_property_y:
        # support for nesting:
        # A new property shall be created in the object contained in the
        # `old_property_c`. The properties `nested_property_d" and `nested_property_e`
        # inside of the object contained in `old_property_c` serve as sources.
        # Dots ('.') are used to denote nesting. If a property name contains a dot, it
        # can be escaped using '\.'. We assume that nobody would have `\.` in their
        # property names, so that cannot be escaped.
        source_properties:
          - old_property_c.nested_property_d
          - old_property_c.nested_property_e
        deduplicate: true
        assume_same_type: false # Will create a union type if the source properties
                                # differ in type.
    ClassB:
      # merged properties for ClassB:
      new_property_z:
        source_properties:
          - old_property_a
          - old_property_b
        # not specifying `deduplicate` and `assume_same_type` is equivalent to
        # their defaults
  ```

B. For merging relation properties:
- the source properties must have the same targetClass
- a config example might look like this:
```yaml
  merge_relation_properties:
    ClassA:
      # merged relation properties that shall be created in ClassA:
      new_property_x:
        source_properties:
          - old_property_a
          - old_property_b
        # There are no option `deduplicate` because relation properties may never
        # contain duplicate values.
        # Moreover, there is no option `assume_same_type` since currently schemapack
        # does not support unions of multiple target classes for a relation. Thus
        # the source relations must all point to the same target class.
        # Nesting is also not supported in relations, so not special treatment of
        # dots is required.
      new_property_y:
        # another merged property
        source_properties:
          - old_property_a
          - old_property_b
    ClassB:
      # merged properties for ClassB:
      new_property_z:
        source_properties:
          - old_property_a
          - old_property_b
  ```

### Refactor aggregate transformation:

The former `aggregate` transformation shall be replaced by a mixture of existing
and to be written transformations. The following new transformations shall be
implemented:

#### Transformation 1: Add Subschema

A transformation that enables the insertion of a subschema into a new property
of any object within the current content schema of a class, including an initial
value. The configuration may look as follows:

```yaml
# Generic Example
- class: MyClass
  target_path: [path, of, properties]
  schema:
    type: "object"
    additionalProperties: false
  value: {}

# Examples from current config
- class: Dataset
  target_path: [datasets]
  schema:
    type: object
    additionalProperties: false
    properties:
      stats:
        type: object
        additionalProperties: false
- class: Dataset
  target_path: [studies_summary]
  schema:
    type: object
    additionalProperties: false
    properties:
      stats:
        type: object
        additionalProperties: false
```

* The transformation shall check whether the specified `target_path` is valid, i.e. all but the last elements are objects and the property does not yet exist.

#### Transformation 2: Count References

* The transformation shall count how many target objects are referenced from each source object given the reference name.
* The transformation shall validate whether the target is defined with multiplicity and fail otherwise

Example config:

```yaml
- class: Dataset
  target_path: [samples_summary, count]
  source_path: [samples]
```

####  Transformation 3: Count content values

* The transformation shall count the values encountered at a specified property in the content of an object.
* The transformation shall validate that at least one of the traversed references is multi-valued by its cardinality or one of the traversed content elements is an array.
* The path to the content property is specified as the list of references to traverse (`source_path.reference`) and the list of content objects to traverse (`source_path.content`).
* The terminal element of the `source_path.content` configuration is the name of the property to be created to hold the resulting value. It is added to the schema by the transformation and has the following schema:
  ```yaml
  type: object
  additionalProperties: true
  ```
  where each observed value will be mapped to an integer value representing the
  number of times it was observed

Draft Config:

```yaml
- class: Dataset
  target_path: [samples_summary, stats, sex]
  source_path:
    reference: [individuals]
    content: [sex]
- class: Dataset
  target_path: [samples_summary, stats, tissues]
  source_path:
    reference: [biospecimens]
    content: [tissue]
```

#### Transformation 4: Arithmetic Operation

* Similarly to the previous transformation, this transformation does not count the element occurrences but enables arithmetic operations on the yielded elements.
* The transformation shall validate that the type of the configured source element allows arithmetic operations (number, integer, boolean).
* The type of the resulting subschema shall be that of the source subschema.
* The transformation shall offer a SUM operator. For future purposes other operators such as MEAN may be useful.

Draft Config:

```yaml
- class: Dataset
  target_path: [files_summary, stats, size]
  source_path:
    reference: [files]
    content: [size]
  operator: SUM
```

#### Transformation 5: Copy

* The transformation shall enable copying the values from one content location (of potentially a referenced class) to another content location.
* The type of the resulting subschema shall be that of the source subschema.

Draft Config:

```yaml
- class: Dataset
  target_path: [dac_email]
  source_path:
    references: [data_access_policy, data_access_committee]
    content: [email]
```

#### Transformation 6: Delete Reference

As the name suggests.

#### Transformation 7: Delete Content Subschema

As the name suggests, following the conventions used in the aforementioned transformations.


## Human Resource/Time Estimation:

Number of sprints required: 1

Number of developers required: multiple
