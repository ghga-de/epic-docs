# Metldata Configurable Workflows (Carpenter Bee)
**Epic Type:** Implementation Epic

Epic planning and implementation follow the
[Epic Planning and Marathon SOP](https://docs.ghga-dev.de/main/sops/sop001_epic_planning.html).

## Scope

### Outline

The goal of this epic is to implement workflows in metldata. In contrast to the previously used approach the workflows shall not be implemented in Python but be fully configurable. Just like individual transformations, workflows shall consume and produce a pair of schemapack and datapack.

In this epic, the following shall be implemented:

* A yaml-serializable WorkflowDefinition class according to the specification below
* Functionality to execute a workflow, given a schemapack, datapack and WorkFlowDefinition

### Workflow Language

The workflow shall be specified as indicated by the following examples:

```yaml
- input: input_schema_name # input schema
  steps:
  - operation: some_transformation
    description: some description
    args:
      {} # transformation config
  - operation: other_transformation
    args:
      {} # ...
```

Originally the data transformation was described in two layers: A _workflow_ comprising _workflow steps_ which correspond to a single transformation, hard coded in Python with a release cycle bound to that of metldata. And a corresponding configuration file configuring each workflow step with the config schema being defined by the transformation executed in the workflow step.

To retain more flexibility the transformations were previously implemented such that they could perform multiple operations of the same type at once, avoiding too frequent changes to the hard coded workflow. With a configurable workflow this can now be relaxed since additional workflow steps can now be added without any development cost. Any "for each" logic shall therefore be removed from the transformations. For example, the previous "delete relation" transformation used to have a config

```yaml
ClassOne: [slot_a, slot_b]
ClassTwo: [slot_k]
ClassThree: [slot_x, slot_y]
```

allowing for multiple slots in multiple classes to be deleted. Following the logic of this workflow language, this would become

```yaml
- input: input_schema_name # input schema
  steps:
  - operation: delete_relation
    args:
      class_name: ClassOne
      relation_name: slot_a
  - operation: delete_relation
    args:
      class_name: ClassOne
      relation_name: slot_b
# ... followed by three more invocations of the delete_relation transformation
```

To generically solve the frequent use case of executing the same operation on multiple targets, the specification shall allow for simple loops, similarly to the Ansible language as described [here](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_loops.html#using-loops):

```yaml
- input: input_schema_name # input schema
  steps:
  - operation: delete_relation
    description: "Delete relation {{ item.relation_name }} from class {{ item.class }}"
    args:
      class_name: "{{ item.class }}"
      relation_name:  "{{ item.relation_name }}"
    loop:
    - class_name: ClassOne
      relation_name: slot_a
    - class_name: ClassOne
      relation_name: slot_b
    - class_name: ClassTwo
      relation_name: slot_k
    - class_name: ClassThree
      relation_name: slot_x
    - class_name: ClassThree
      relation_name: slot_y
    - class_name: ClassThree
      relation_name: slot_z
```

### Implementation Detail

Templating the loop logic can be achieved using a minimal data model for the initial deserialization of the YAML file, followed by YAML or JSON serialization, templating and re-deserialization into the actual operation models. The initial pre-templating model may be specified as

```Python
class StepTemplate:
    name: str
    description: str
    args: dict
    loop: list

class WorkflowTemplate:
    input: str
    output: str
    steps: list[StepTemplate]
```

Templating the loop logic for each list item will then resolve every item to a list of items where the properties `name`, `description` and `args` were re-serialized, templated for each `loop` item and de-serialized.



### Included / Required

### Example

An example that aims to replace the current workflow specification ([ghga.py](https://github.com/ghga-de/metldata/blob/2.1.2/src/metldata/builtin_workflows/ghga_archive.py)) and configuration ([metadata_config.yaml](https://github.com/ghga-de/metadata-config/blob/2.0.0%2B6/configuration/metadata_config.yaml)) i provided [here](./example_workflow.yaml) as part of the epic spec.

## Human Resource/Time Estimation:

Number of sprints required: ?

Number of developers required: ?
