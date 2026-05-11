# Differences

## FHIR

FHIR references are explicitly modeled in resources.

They can point to an identifier or a string, or both. They also carry a `type` and a `display`.

![Reference](https://raw.githubusercontent.com/SevKohler/openehr-reference/main/Reference.png)

`CodeableReference` can hold either a code or a `Reference`.

![CodeableReference](https://raw.githubusercontent.com/SevKohler/openehr-reference/main/CodeableReference.png)

## openEHR

openEHR has two ways of linking things.

`LINK` is available on every `LOCATABLE`. It carries a meaning, a type, and a target (an internal EHR URI). It is generic and can be used almost anywhere.

![LINKS](https://raw.githubusercontent.com/SevKohler/openehr-reference/main/LINKS.png)

External references are modeled separately in the RM, with several flavours.

![PARTY](https://raw.githubusercontent.com/SevKohler/openehr-reference/main/PARTY.png)

## Problems

`LINK` can be attached to any `LOCATABLE`, which kills interoperability. The `type` is a free string, so its use is arbitrary.

External references are explicitly modeled in the RM, and `LINK` cannot stand in for them.

You also cannot proxy parts of the referenced data the way `CodeableReference` does. A `Procedure.reason` should link to `problem_diagnosis` and proxy the diagnosis code, not duplicate it.

## Use cases

1. **Encounter linkage on a `COMPOSITION`**. Standard RM-level field, present on every composition.
2. **Procedure archetype with `reason`** linking a `problem_diagnosis` field. Archetype-level reference. If nothing exists to link, the same slot falls back to a static coded value.
3. **`CLUSTER` archetype that hosts a link**, e.g. a device cluster pointing to a device entry. References work at sub-composition granularity, not only at archetype root.
4. **Archetype that surfaces multiple fields from the same source**, e.g. linking a `problem_diagnosis` and showing both diagnosis and severity from it. The single-field case is easy: one ELEMENT, one link, done. Two or more fields raise a real modeling question. A modeler's instinct is to write `ELEMENT[reason]` and `ELEMENT[severity]` as separate fields, each with its own description, value set, and terminology bindings. Open questions: do they share one link, do they carry one each, or is this an engine-level concern that resolves the values at runtime from a single shared link. Meaning also shifts when linking: the source archetype calls the field `diagnosis`, but the using archetype frames it as `reason`. Could the description be copied over from the source? Probably not, sadly. The using side's semantics win, so the source description rarely fits.
5. **Reference to an external entity** outside openEHR, e.g. a FHIR `Organization` from the demographic side. External pointer typed as `fhir_organization`.
6. The CodeabelReference where we have geniuly a choice a simple code maybe enough, or you want to link smt or maybe they need to slot that in. Like a Device name. Oh wait we want to point to the actual structure. 
7. Sombedy has an archetype we need to know the device name, just that. Someone else says occassionaly we need more details about the device, right now you would add a cluster slot for device and but now we have a problem we havce a decvice name in the top level, but we also have a device name in the cluster so which one do we use. Can we point the top one to the one in the cluster. Could we not one place where we expand the coded to a complex strucutre. 
8. Now more complicated now this device is not even stored in the EHR maybe we got a registry and we point to that them and e.g. stored in a fhir store. Connect to a Cluster archetype we need to point out to an external locations. 
External Internal Slot and CodedText.

Replace DEVICE link you have with the DV_REFERENCE 

## Requirements

Derived from the use cases:

- Usable as a typed RM-level attribute (case 1).
- Usable inside `ELEMENT.value` so archetypes control where it appears (cases 2, 3, 4).
- Hostable on `ENTRY` and `CLUSTER` archetypes alike (cases 2, 3).
- Target may be any internal `LOCATABLE`, including a sub-tree like a `CLUSTER` (cases 2, 3).
- Optional coded fallback when no internal target exists (case 2).
- Proxy multiple typed values from the target via paths (case 4).
- Reference an internal `LOCATABLE` or an external entity in one mechanism (cases 2-5).
- `type` value set extensible to cover openEHR target classes and external targets like `fhir_organization` (case 5).
- Not attachable to arbitrary `LOCATABLE`s. Replaces the ad-hoc use of `LINK` for clinical references.
- Each linked or hardcoded value must keep full ELEMENT-level modeling: description, constraints, value-set bindings, terminology, translations. Values embedded as a sub-attribute of `DV_REFERENCE` (e.g. inside a `PROXY_VALUE`) lose that surface, so the design must let modelers express each value as an `ELEMENT` rather than only as nested entries (cases 2, 4).

## Suggestion

Add a new RM class `DV_REFERENCE` that combines the logic of `PARTY`, `LINK`, and FHIR `Reference` / `CodeableReference`.

### Where it should appear

A reference lives in one of two places. Either explicitly modeled in the archetype as a `DV_REFERENCE` inside `ELEMENT.value`, or as a typed attribute in the RM itself. Nothing in between.

For a `COMPOSITION`, the only RM-level slot is the encounter (`EVENT_CONTEXT`). Everything clinical is archetyped, so clinical references fall to the archetype case.

Example: an `ACTION` for a procedure carries `reason` as a `DV_REFERENCE` defined in the archetype, targeting the `EVALUATION` for `problem_diagnosis` and proxying the diagnosis code. Today this lives in free text or is bolted on via a `LINK`, so tooling cannot rely on it.


Slot as it should be

SLOT REFERENCE

SLOT || REFERENCE THIS IS THE SITUATION

Add to the CLUSTER SLOT, or add an ITEM_STRUCTURE into REFERENCE. 

ELEMENT converge and CLUSTER into CLASS and then allowed SMART LINKS. 
### `DV_REFERENCE` class

Inherits from `DATA_VALUE`.

<table>
  <tr style="background-color:#87CEEB; color:#000;">
    <td><b>CLASS</b></td>
    <td colspan="2"><b><code>DV_REFERENCE</code></b></td>
  </tr>
  <tr style="background-color:#ffffff; color:#000;">
    <td><b>Description</b></td>
    <td colspan="2">A reference from a clinical data value to another resource. Points internally to an EHR <code>LOCATABLE</code> (<code>internal_ref</code>) or externally to a non-openEHR entity (<code>external_ref</code>). When a target is linked, <code>proxy</code> surfaces specific values from it. When nothing is linked, <code>proxy</code> may instead carry hardcoded values populated directly by the modeler. Used inside <code>ELEMENT.value</code> wherever an archetype models a typed link, replacing the ad-hoc use of <code>LINK</code> for clinical references.</td>
  </tr>
  <tr style="background-color:#87CEEB; color:#000;">
    <td><b>Attributes</b></td>
    <td><b>Signature</b></td>
    <td><b>Meaning</b></td>
  </tr>
  <tr style="background-color:#ffffff; color:#000;">
    <td>0..1</td>
    <td><code>internal_ref</code>: <code>DV_EHR_URI</code></td>
    <td>Pointer to a <code>LOCATABLE</code> inside the EHR.</td>
  </tr>
  <tr style="background-color:#f5f5f5; color:#000;">
    <td>0..1</td>
    <td><code>external_ref</code>: <code>OBJECT_REF</code></td>
    <td>Pointer to an external entity.</td>
  </tr>
  <tr style="background-color:#ffffff; color:#000;">
    <td>0..1</td>
    <td><code>display</code>: <code>DV_TEXT</code></td>
    <td>Human-readable label for the target.</td>
  </tr>
  <tr style="background-color:#ffffff; color:#000;">
    <td>0..*</td>
    <td><code>proxy</code>: <code>List&lt;PROXY_BASE&gt;</code></td>
    <td>Values surfaced from the target, or hardcoded when no link is set.</td>
  </tr>
  <tr style="background-color:#87CEEB; color:#000;">
    <td><b>Invariants</b></td>
    <td colspan="2"></td>
  </tr>
  <tr style="background-color:#ffffff; color:#000;">
    <td colspan="3">
      <ul>
        <li>At least one of <code>internal_ref</code>, <code>external_ref</code>, or <code>proxy</code> is present.</li>
        <li><code>internal_ref</code> and <code>external_ref</code> cannot both be set.</li>
      </ul>
    </td>
  </tr>
</table>

#### `PROXY_BASE` class

<table>
  <tr style="background-color:#87CEEB; color:#000;">
    <td><b>CLASS</b></td>
    <td colspan="2"><b><code>PROXY_BASE</code></b> (abstract)</td>
  </tr>
  <tr style="background-color:#ffffff; color:#000;">
    <td><b>Description</b></td>
    <td colspan="2">Abstract base for proxy entries. Carries the resolved or hardcoded <code>value</code>. Path handling is left to subclasses: <code>PROXY_VALUE</code> carries a single <code>path</code> into the linked target; <code>PROXY_EXPRESSION</code> manages paths per component axis inside its <code>EXPRESSION_DEFINITION</code> entries.</td>
  </tr>
  <tr style="background-color:#87CEEB; color:#000;">
    <td><b>Attributes</b></td>
    <td><b>Signature</b></td>
    <td><b>Meaning</b></td>
  </tr>
  <tr style="background-color:#ffffff; color:#000;">
    <td>0..1</td>
    <td><code>value</code>: <code>DATA_VALUE</code></td>
    <td>Resolved or hardcoded value.</td>
  </tr>
</table>

#### `PROXY_VALUE` class

Inherits from `PROXY_BASE`. Narrows `path` to `0..1`.

<table>
  <tr style="background-color:#87CEEB; color:#000;">
    <td><b>CLASS</b></td>
    <td colspan="2"><b><code>PROXY_VALUE</code></b></td>
  </tr>
  <tr style="background-color:#ffffff; color:#000;">
    <td><b>Description</b></td>
    <td colspan="2">Carries a single value alongside a reference. With <code>path</code> set, it surfaces a value resolved from <code>internal_ref</code>. Without <code>path</code>, it carries a hardcoded value the modeler populated directly.</td>
  </tr>
  <tr style="background-color:#87CEEB; color:#000;">
    <td><b>Attributes</b></td>
    <td><b>Signature</b></td>
    <td><b>Meaning</b></td>
  </tr>
  <tr style="background-color:#ffffff; color:#000;">
    <td>0..1</td>
    <td><code>path</code>: <code>EHR_PATH</code></td>
    <td>Archetype path relative to <code>internal_ref</code>. Present when proxying from a linked target.</td>
  </tr>
  <tr style="background-color:#87CEEB; color:#000;">
    <td><b>Invariants</b></td>
    <td colspan="2"></td>
  </tr>
  <tr style="background-color:#ffffff; color:#000;">
    <td colspan="3">
      <ul>
        <li>At least one of <code>path</code> or <code>value</code> is set.</li>
      </ul>
    </td>
  </tr>
</table>

#### `PROXY_EXPRESSION` class

Inherits from `PROXY_BASE`. Intended for post-coordinated expressions where a concept is refined by a set of attribute-value axes, as defined by clinical expression languages such as SNOMED CT Compositional Grammar (SCG) or ICD-11 post-coordination. Each axis is an `EXPRESSION_DEFINITION` with an `attribute` holding the expression language operator (e.g. `:` = AS OF, `,` = AND in SCG), a `code` holding the terminology attribute, and a `path` resolving the attribute value from the linked archetype.

<table>
  <tr style="background-color:#87CEEB; color:#000;">
    <td><b>CLASS</b></td>
    <td colspan="2"><b><code>PROXY_EXPRESSION</code></b></td>
  </tr>
  <tr style="background-color:#ffffff; color:#000;">
    <td><b>Description</b></td>
    <td colspan="2">Carries a post-coordinated definition as an ordered list of <code>EXPRESSION_DEFINITION</code> entries, one per refinement axis. Each entry has an <code>attribute</code> holding the expression language operator (e.g. <code>":"</code> = AS OF, <code>","</code> = AND in SNOMED CT SCG), a <code>code</code> (<code>CODE_PHRASE</code> for the terminology attribute, e.g. <code>272741003|Laterality|</code>), and a <code>path</code> resolving the attribute value from <code>internal_ref</code>. The inherited <code>value</code> holds the single assembled result.</td>
  </tr>
  <tr style="background-color:#87CEEB; color:#000;">
    <td><b>Attributes</b></td>
    <td><b>Signature</b></td>
    <td><b>Meaning</b></td>
  </tr>
  <tr style="background-color:#ffffff; color:#000;">
    <td>1..*</td>
    <td><code>components</code>: <code>List&lt;EXPRESSION_DEFINITION&gt;</code></td>
    <td>Ordered list of component axes that compose the definition.</td>
  </tr>
  <tr style="background-color:#87CEEB; color:#000;">
    <td><b>Invariants</b></td>
    <td colspan="2"></td>
  </tr>
  <tr style="background-color:#ffffff; color:#000;">
    <td colspan="3">
      <ul>
        <li>At least one <code>components</code> entry is set.</li>
      </ul>
    </td>
  </tr>
</table>

#### `EXPRESSION_DEFINITION` class

<table>
  <tr style="background-color:#87CEEB; color:#000;">
    <td><b>CLASS</b></td>
    <td colspan="2"><b><code>EXPRESSION_DEFINITION</code></b></td>
  </tr>
  <tr style="background-color:#ffffff; color:#000;">
    <td><b>Description</b></td>
    <td colspan="2">A single refinement axis of a post-coordinated expression. <code>attribute</code> holds the operator defined by the expression language (e.g. <code>:</code> = AS OF and <code>,</code> = AND in SNOMED CT SCG). <code>code</code> holds the terminology attribute being applied (e.g. <code>272741003|Laterality|</code>). <code>path</code> resolves the attribute value from the linked archetype instance.</td>
  </tr>
  <tr style="background-color:#87CEEB; color:#000;">
    <td><b>Attributes</b></td>
    <td><b>Signature</b></td>
    <td><b>Meaning</b></td>
  </tr>
  <tr style="background-color:#ffffff; color:#000;">
    <td>1..1</td>
    <td><code>attribute</code>: <code>CODE_PHRASE</code></td>
    <td>Operator defined by the expression language for this axis (e.g. <code>"focus concept"</code> for the base concept, <code>":"</code> = AS OF for the first refinement, <code>","</code> = AND for subsequent refinements in SNOMED CT SCG). Other expression languages may define different operators.</td>
  </tr>
  <tr style="background-color:#f5f5f5; color:#000;">
    <td>1..1</td>
    <td><code>code</code>: <code>CODE_PHRASE</code></td>
    <td>The terminology attribute applied in this axis (e.g. <code>363698007|Finding site|</code> or <code>272741003|Laterality|</code> in SNOMED CT SCG).</td>
  </tr>
  <tr style="background-color:#ffffff; color:#000;">
    <td>0..1</td>
    <td><code>path</code>: <code>EHR_PATH</code></td>
    <td>Path to the attribute value, relative to <code>internal_ref</code>. Resolves dynamically from the linked archetype instance.</td>
  </tr>
  <tr style="background-color:#87CEEB; color:#000;">
    <td><b>Invariants</b></td>
    <td colspan="2"></td>
  </tr>
  <tr style="background-color:#ffffff; color:#000;">
    <td colspan="3">
      <ul>
        <li><code>path</code> is set.</li>
      </ul>
    </td>
  </tr>
</table>

`EHR_PATH` is a new RM primitive. Constrained string validated against the archetype path grammar (`/segment[at-code]/...`).

#### `DV_EHR_URI` class (revised)

<table>
  <tr style="background-color:#87CEEB; color:#000;">
    <td><b>CLASS</b></td>
    <td colspan="2"><b><code>DV_EHR_URI</code></b></td>
  </tr>
  <tr style="background-color:#ffffff; color:#000;">
    <td><b>Description</b></td>
    <td colspan="2">A URI pointing to a <code>LOCATABLE</code> inside an EHR, using the <code>ehr://</code> scheme.</td>
  </tr>
  <tr style="background-color:#87CEEB; color:#000;">
    <td><b>Attributes</b></td>
    <td><b>Signature</b></td>
    <td><b>Meaning</b></td>
  </tr>
  <tr style="background-color:#ffffff; color:#000;">
    <td>1..1</td>
    <td><code>value</code>: <code>String</code></td>
    <td>The EHR URI. Must use the <code>ehr://</code> scheme.</td>
  </tr>
  <tr style="background-color:#f5f5f5; color:#000;">
    <td>1..1</td>
    <td><code>namespace</code>: <code>String</code></td>
    <td>Namespace to which this identifier belongs in the local system context (and possibly in any other openEHR compliant environment) e.g. terminology, demographic. Legal values are: <code>"local"</code>, <code>"unknown"</code>, or a string matching <code>[a-zA-Z][a-zA-Z0-9_.:\/&?=+-]*</code>.</td>
  </tr>
  <tr style="background-color:#ffffff; color:#000;">
    <td>1..1</td>
    <td><code>type</code>: <code>String</code></td>
    <td>Name of the class (concrete or abstract) of object to which this identifier type refers, e.g. <code>PARTY</code>, <code>PERSON</code>, <code>EVALUATION</code>. Use <code>ANY</code> to indicate that any type is accepted.</td>
  </tr>
  <tr style="background-color:#f5f5f5; color:#000;">
    <td>0..1</td>
    <td><code>archetype_id</code>: <code>String</code></td>
    <td>Constrains the archetype of the target <code>LOCATABLE</code>. Validated as an archetype id regex at constraint time.</td>
  </tr>
</table>

## Open thought: linked CLUSTER for multi-field references

For multi-field references, a CLUSTER with a single `shared_link` is the more natural modeling approach. The modeler defines the fields they need as ELEMENTs inside the CLUSTER — each with its own description, constraints, and terminology bindings — and the link lives once on the CLUSTER, not per field. This maps directly to how modelers already think about CLUSTERs and is far less awkward than multiple parallel `DV_REFERENCE`s.

`shared_link` could also be placed on `ELEMENT` directly, but that is intentionally avoided. Putting a link on every ELEMENT would reproduce the same problem as `LINK` on `LOCATABLE` — modelers can attach references anywhere, constraints become unenforceable, and interoperability breaks down. Keeping `shared_link` on `CLUSTER` only means the archetype structure itself makes the scope of the reference explicit and auditable.

### Sketch: CLUSTER with `shared_link`

<table>
  <tr style="background-color:#87CEEB; color:#000;">
    <td><b>CLASS</b></td>
    <td colspan="2"><b><code>CLUSTER</code></b> (extension)</td>
  </tr>
  <tr style="background-color:#ffffff; color:#000;">
    <td><b>Description</b></td>
    <td colspan="2">One new optional attribute added. The CLUSTER otherwise behaves as today.</td>
  </tr>
  <tr style="background-color:#87CEEB; color:#000;">
    <td><b>Attributes</b></td>
    <td><b>Signature</b></td>
    <td><b>Meaning</b></td>
  </tr>
  <tr style="background-color:#ffffff; color:#000;">
    <td>0..1</td>
    <td><code>shared_link</code>: <code>DV_REFERENCE</code></td>
    <td>Link to a source archetype shared by all child ELEMENTs that resolve from it.</td>
  </tr>
</table>

## FHIR alignment

`DV_REFERENCE` maps closely to two FHIR constructs.

### FHIR `Reference`

FHIR `Reference` carries four optional fields: `reference` (a literal URL or relative reference), `type`, `identifier` (a logical identifier independent of the URL), and `display`. The key design point is that `reference` and `identifier` can coexist in the same instance — a FHIR resource can say "this is the URL *and* this is the logical business identifier for the same target".

`DV_REFERENCE` covers the same ground but splits the two pointer kinds explicitly:

- `internal_ref` (`DV_EHR_URI`) maps to FHIR's `reference` field — a resolvable pointer to the target.
- `external_ref` (`OBJECT_REF`) maps to FHIR's `identifier` — a logical identifier for an external entity that may not be directly dereferenceable (e.g. a FHIR `Organization` identifier in a remote registry).
- `display` maps directly to FHIR's `display`.

The split also resolves FHIR's ambiguity: `internal_ref` and `external_ref` are mutually exclusive by invariant, whereas FHIR allows `reference` and `identifier` to coexist. The mutual exclusion was a deliberate choice — a single `DV_REFERENCE` targets either an internal openEHR `LOCATABLE` or an external entity, never both. An open question is whether `external_ref` should be allowed to carry a logical identifier alongside a resolvable URL (as FHIR does), to support external targets that are both addressable and identifiable independently.

### FHIR `CodeableReference`

`CodeableReference` holds either a `CodeableConcept` or a `Reference`. The design intent is that a field may be satisfied by a code alone (when no structured resource is available) or by a pointer to a resource (when one exists), and the two are interchangeable representations of the same thing.

`DV_REFERENCE` covers this pattern through its invariant that allows `proxy` to be present without any `internal_ref` or `external_ref`. In that case:

- **Reference case**: `internal_ref` or `external_ref` is set, and `proxy` surfaces values derived from the target — analogous to `CodeableReference.reference`.
- **Coded case**: only `proxy` is populated with a hardcoded `DV_CODED_TEXT` value and no link is set — analogous to `CodeableReference.concept`.

The ADL `Procedure.reason` example makes this explicit: the `ELEMENT.value` constraint accepts either a `DV_REFERENCE` with `internal_ref` (linked case) or a plain `DV_CODED_TEXT` (coded fallback), which is the same either/or that `CodeableReference` expresses natively in FHIR.

## Open points

- **`EHR_PATH` positional indexing**: the standard archetype path grammar has no notation for selecting a specific occurrence of a `0..*` node. AQL uses name predicates (`[at0002, 'name']`), not positional ones. If a path must unambiguously target one occurrence among many, a positional predicate such as `[n]` (e.g. `items[at0002][1]`) would need to be added to the `EHR_PATH` grammar. Without it, paths that pass through `0..*` nodes are ambiguous and cannot be resolved to a single value.

- **Cross-CDR openEHR references**: `internal_ref` scopes to the local EHR and `external_ref` was designed for non-openEHR entities. There is no mechanism to point to a `LOCATABLE` in another openEHR CDR — the location of the remote CDR cannot be expressed. Needs a dedicated use case and requirement.

- **Legacy data**: deprecate `LINK`; how to handle existing `OBJECT_REF` / `PARTY_REF` data is unresolved.

- **`PROXY_EXPRESSION` component ordering**: the `components` list is ordered (AS OF first, AND following), but JSON does not enforce array ordering in all implementations. An explicit `index` field on `EXPRESSION_DEFINITION` may be needed to guarantee the correct expression language serialization order regardless of the serialization format.

## Examples

### `DV_REFERENCE` — Procedure reason as JSON instance

Two cases for `Procedure.reason`: linked (reference to a recorded `problem_diagnosis`) and unlinked (static coded fallback).

**Linked case** — `internal_ref` points to a `problem_diagnosis` EVALUATION; `proxy` surfaces the diagnosis code:

```json
{
  "_type": "DV_REFERENCE",
  "internal_ref": {
    "_type": "DV_EHR_URI",
    "namespace": "openEHR",
    "type": "EVALUATION",
    "value": "ehr://ehr-id/compositions/comp-id/content[at0001]/data[at0002]/items[at0003]"
  },
  "display": {
    "_type": "DV_TEXT",
    "value": "Malignant neoplasm of right kidney"
  },
  "proxy": [
    {
      "_type": "PROXY_VALUE",
      "path": "/data[at0001]/items[at0002]/value",
      "value": {
        "_type": "DV_CODED_TEXT",
        "value": "Malignant neoplasm of right kidney",
        "defining_code": { "terminology_id": "snomed", "code_string": "363346000" }
      }
    }
  ]
}
```

**Unlinked case** — no `problem_diagnosis` recorded; `proxy` carries a hardcoded coded value from the reason value set:

```json
{
  "_type": "DV_REFERENCE",
  "proxy": [
    {
      "_type": "PROXY_VALUE",
      "value": {
        "_type": "DV_CODED_TEXT",
        "value": "Cancer",
        "defining_code": { "terminology_id": "local", "code_string": "at0010" }
      }
    }
  ]
}
```

### `DV_REFERENCE` — External reference to a FHIR Organization

A `Procedure.performer` pointing to an `Organization` held in a FHIR registry outside the EHR:

```json
{
  "_type": "DV_REFERENCE",
  "external_ref": {
    "_type": "OBJECT_REF",
    "namespace": "fhir",
    "type": "Organization",
    "id": {
      "_type": "GENERIC_ID",
      "value": "Organization/123",
      "scheme": "https://fhir.example.org"
    }
  },
  "display": {
    "_type": "DV_TEXT",
    "value": "City Oncology Centre"
  }
}
```

### `PROXY_EXPRESSION` — post-coordinated diagnosis on a procedure

This example records "Malignant neoplasm of right kidney" as a SNOMED CT post-coordinated expression, assembled from a linked `problem_diagnosis` archetype instance. The focus concept is hardcoded; the finding site is resolved dynamically from the linked archetype; laterality is a static hardcoded value.

The SNOMED CT Expression Language representation is:

```
363346000|Malignant neoplastic disease|:
  363698007|Finding site|=64033007|Kidney structure|,
  272741003|Laterality|=24028007|Right|
```

As a JSON instance:

```json
{
  "_type": "DV_REFERENCE",
  "internal_ref": {
    "_type": "DV_EHR_URI",
    "namespace": "openEHR",
    "type": "EVALUATION",
    "value": "ehr://ehr-id/compositions/comp-id/content[at0001]/data[at0002]/items[at0003]"
  },
  "proxy": [
    {
      "_type": "PROXY_EXPRESSION",
      "value": {
        "_type": "DV_CODED_TEXT",
        "value": "363346000|Malignant neoplastic disease|:363698007|Finding site|=64033007|Kidney structure|,272741003|Laterality|=24028007|Right|",
        "defining_code": { "terminology_id": "snomed", "code_string": "363346000" }
      },
      "components": [
        {
          "_type": "EXPRESSION_DEFINITION",
          "attribute": { "terminology_id": "snomed", "code_string": "focus concept" },
          "code": { "terminology_id": "snomed", "code_string": "363346000" },
          "path": "/data[at0001]/items[at0001]/value"
        },
        {
          "_type": "EXPRESSION_DEFINITION",
          "attribute": { "terminology_id": "snomed", "code_string": ":" },
          "code": { "terminology_id": "snomed", "code_string": "363698007" },
          "path": "/data[at0001]/items[at0002]/value"
        },
        {
          "_type": "EXPRESSION_DEFINITION",
          "attribute": { "terminology_id": "snomed", "code_string": "," },
          "code": { "terminology_id": "snomed", "code_string": "272741003" },
          "path": "/data[at0001]/items[at0005]/value"
        },
        {
          "_type": "EXPRESSION_DEFINITION",
          "attribute": { "terminology_id": "snomed", "code_string": "," },
          "code": { "terminology_id": "snomed", "code_string": "116676008" },
          "path": "/data[at0001]/items[at0073]/value"
        }
      ]
    }
  ]
}
```

For a static axis (e.g. laterality fixed at modeling time rather than resolved from the archetype), `code` replaces `path`:

```json
{
  "_type": "EXPRESSION_DEFINITION",
  "attribute": { "terminology_id": "snomed", "code_string": "," },
  "code": { "terminology_id": "snomed", "code_string": "272741003" },
  "path": "/data[at0001]/items[at0005]/value"
}
```

The focus concept (`363346000|Malignant neoplastic disease|`) is the archetype-level concept of the `problem_diagnosis` EVALUATION targeted by `internal_ref`. The `:` operator refines it with the two `EXPRESSION_DEFINITION` axes: the finding site is resolved at runtime from the archetype path; laterality is fixed at modeling time via a hardcoded `code`.
