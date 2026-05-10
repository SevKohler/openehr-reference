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
    <td><code>internal_ref</code> (0..1)</td>
    <td><code>DV_EHR_URI</code></td>
    <td>Pointer to a <code>LOCATABLE</code> inside the EHR.</td>
  </tr>
  <tr style="background-color:#f5f5f5; color:#000;">
    <td><code>external_ref</code> (0..1)</td>
    <td><code>OBJECT_REF</code></td>
    <td>Pointer to an external entity.</td>
  </tr>
  <tr style="background-color:#ffffff; color:#000;">
    <td><code>display</code> (0..1)</td>
    <td><code>DV_TEXT</code></td>
    <td>Human-readable label for the target.</td>
  </tr>
  <tr style="background-color:#ffffff; color:#000;">
    <td><code>proxy</code> (0..*)</td>
    <td><code>List&lt;PROXY_VALUE&gt;</code></td>
    <td>Values surfaced from the target, or hardcoded when no link is set.</td>
  </tr>
  <tr style="background-color:#87CEEB; color:#000;">
    <td><b>Invariants</b></td>
    <td colspan="2"></td>
  </tr>
  <tr style="background-color:#ffffff; color:#000;">
    <td colspan="3">
      <ul>
        <li>At most one of <code>internal_ref</code> or <code>external_ref</code> is set.</li>
        <li>At least one of <code>internal_ref</code>, <code>external_ref</code>, or <code>proxy</code> is present.</li>
        <li>A <code>PROXY_VALUE.path</code> may only be set when <code>internal_ref</code> is set.</li>
      </ul>
    </td>
  </tr>
</table>

#### `PROXY_VALUE` class

<table>
  <tr style="background-color:#87CEEB; color:#000;">
    <td><b>CLASS</b></td>
    <td colspan="2"><b><code>PROXY_VALUE</code></b></td>
  </tr>
  <tr style="background-color:#ffffff; color:#000;">
    <td><b>Description</b></td>
    <td colspan="2">Carries a value alongside a reference. With <code>path</code> set, it surfaces a value resolved from <code>internal_ref</code>. Without <code>path</code>, it carries a hardcoded value the modeler populated directly.</td>
  </tr>
  <tr style="background-color:#87CEEB; color:#000;">
    <td><b>Attributes</b></td>
    <td><b>Signature</b></td>
    <td><b>Meaning</b></td>
  </tr>
  <tr style="background-color:#ffffff; color:#000;">
    <td><code>path</code> (0..1)</td>
    <td><code>EHR_PATH</code></td>
    <td>Archetype path relative to <code>internal_ref</code>. Present when proxying from a linked target.</td>
  </tr>
  <tr style="background-color:#f5f5f5; color:#000;">
    <td><code>value</code> (0..1)</td>
    <td><code>DATA_VALUE</code></td>
    <td>Resolved or hardcoded value.</td>
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

`EHR_PATH` is a new RM primitive. Constrained string validated against the archetype path grammar (`/segment[at-code]/...`).

#### `OBJECT_REF` class (revised)

<table>
  <tr style="background-color:#87CEEB; color:#000;">
    <td><b>CLASS</b></td>
    <td colspan="2"><b><code>OBJECT_REF</code></b></td>
  </tr>
  <tr style="background-color:#ffffff; color:#000;">
    <td><b>Description</b></td>
    <td colspan="2">Class describing a reference to another object, which may exist locally or be maintained outside the current namespace.</td>
  </tr>
  <tr style="background-color:#87CEEB; color:#000;">
    <td><b>Attributes</b></td>
    <td><b>Signature</b></td>
    <td><b>Meaning</b></td>
  </tr>
  <tr style="background-color:#ffffff; color:#000;">
    <td><code>id</code> (1..1)</td>
    <td><code>OBJECT_ID</code></td>
    <td>Globally unique id of an object, regardless of where it is stored.</td>
  </tr>
</table>

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
    <td><code>value</code> (1..1)</td>
    <td><code>String</code></td>
    <td>The EHR URI. Must use the <code>ehr://</code> scheme.</td>
  </tr>
  <tr style="background-color:#f5f5f5; color:#000;">
    <td><code>namespace</code> (1..1)</td>
    <td><code>String</code></td>
    <td>Namespace to which this identifier belongs in the local system context (and possibly in any other openEHR compliant environment) e.g. terminology, demographic. Legal values are: <code>"local"</code>, <code>"unknown"</code>, or a string matching <code>[a-zA-Z][a-zA-Z0-9_.:\/&?=+-]*</code>.</td>
  </tr>
  <tr style="background-color:#ffffff; color:#000;">
    <td><code>type</code> (1..1)</td>
    <td><code>String</code></td>
    <td>Name of the class (concrete or abstract) of object to which this identifier type refers, e.g. <code>PARTY</code>, <code>PERSON</code>, <code>EVALUATION</code>. Use <code>ANY</code> to indicate that any type is accepted.</td>
  </tr>
  <tr style="background-color:#f5f5f5; color:#000;">
    <td><code>archetype_id</code> (0..1)</td>
    <td><code>String</code></td>
    <td>Constrains the archetype of the target <code>LOCATABLE</code>. Validated as an archetype id regex at constraint time.</td>
  </tr>
</table>

### Archetype constraints

The constraint logic is standard ADL. Reuse the regex-on-`archetype_id` mechanism that `ARCHETYPE_SLOT` already uses, but apply it to the archetype segment inside `internal_ref`. Add path constraints for each `proxy` entry. Constrain `type` like any other coded value.

Example for the Procedure `reason` slot, which either links a `problem_diagnosis` (proxying diagnosis and severity) or carries a static coded value when nothing exists to link.

ADL 2:

```adl
ELEMENT[id3] occurrences matches {0..1} matches {    -- Reason
  value matches {
    DV_REFERENCE[id4] matches {
      type matches {[at0010]}                        -- "problem_diagnosis_ref"
      internal_ref matches {
        DV_EHR_URI[id5] matches {
          archetype_id matches {/openEHR-EHR-EVALUATION\.problem_diagnosis\.v\d+/}
        }
      }
      proxy cardinality matches {2..2; ordered} matches {
        PROXY_VALUE[id6] matches {
          path matches {"/data[at0001]/items[at0002]/value"}    -- diagnosis
        }
        PROXY_VALUE[id7] matches {
          path matches {"/data[at0001]/items[at0073]/value"}    -- severity
        }
      }
    }
    DV_CODED_TEXT[id8] matches {                      -- static fallback
      defining_code matches {[ac0001]}                -- reason value set
    }
  }
}
```

Reading: `value` accepts either a `DV_REFERENCE` constrained to a `problem_diagnosis` URI with two ordered proxies, or a plain coded value from the reason value set.

## Open thought: linked CLUSTER for multi-field references

`PROXY_VALUE` carries only `path` and `value`, so it has none of the modeling surface an ELEMENT has (description, constraints, value sets, terminology). For a single proxied value that is fine. For two or more fields from the same source archetype it falls short, and modeling them as separate `DV_REFERENCE`s disconnects fields that belong together.

**Direction**

Model the linked group as a CLUSTER. The CLUSTER carries a `shared_link` to one source archetype (e.g. `problem_diagnosis`). Inside, the modeler picks the ELEMENTs they need (`reason`, `severity`) as specialized clones of the source ELEMENTs, keeping or respec'ing the at-code, description, and constraints locally. At recording time each picked ELEMENT is either resolved from the link or carries a hardcoded value.

Mental model: slot a CLUSTER from another archetype, pick and respec the fields you want. The shared origin lives once, on the CLUSTER, not duplicated per ELEMENT.

**Trade-offs**

- Validation must check that the source path resolves to an ELEMENT, types match, and the proxy is a valid specialization.
- Conceptual proximity to archetype specialization. Needs clear scoping so the two mechanisms do not bleed into each other.
- Terminology overlap (source vs proxy codes/text) needs an explicit rule for AQL and UI.
- Nested proxies need either a ban or specified semantics.
- Modeling will be messy because multiple valid approaches exist. Not a blocker. Down to the modeler's judgment.

**Net**

`PROXY_VALUE` handles the single-value case now. The linked CLUSTER is the likely direction for multi-field references.

### Sketch: CLUSTER with `shared_link`

`CLUSTER` gets one new optional attribute. Everything else stays as it is today.

<table>
  <tr style="background-color:#87CEEB; color:#000;">
    <td><b>CLASS</b></td>
    <td colspan="2"><b><code>CLUSTER</code></b> (extension)</td>
  </tr>
  <tr style="background-color:#ffffff; color:#000;">
    <td><b>Description</b></td>
    <td colspan="2">Adds a single shared link to a source archetype. When set, child ELEMENTs may resolve their values from paths inside that link. The CLUSTER otherwise behaves as a normal `CLUSTER`.</td>
  </tr>
  <tr style="background-color:#87CEEB; color:#000;">
    <td><b>Attributes</b></td>
    <td><b>Signature</b></td>
    <td><b>Meaning</b></td>
  </tr>
  <tr style="background-color:#ffffff; color:#000;">
    <td><code>shared_link</code> (0..1)</td>
    <td><code>DV_REFERENCE</code></td>
    <td>Link to a source archetype shared by all child ELEMENTs that resolve from it.</td>
  </tr>
</table>

`ELEMENT` gets one new optional attribute, `source_path`. When set, the value was resolved from the parent CLUSTER's `shared_link` at that path. When unset, the value was hardcoded.

<table>
  <tr style="background-color:#87CEEB; color:#000;">
    <td><b>CLASS</b></td>
    <td colspan="2"><b><code>ELEMENT</code></b> (extension)</td>
  </tr>
  <tr style="background-color:#87CEEB; color:#000;">
    <td><b>Attributes</b></td>
    <td><b>Signature</b></td>
    <td><b>Meaning</b></td>
  </tr>
  <tr style="background-color:#ffffff; color:#000;">
    <td><code>source_path</code> (0..1)</td>
    <td><code>EHR_PATH</code></td>
    <td>If set, the value was resolved from the parent CLUSTER's <code>shared_link</code> at this path. If unset, the value was hardcoded.</td>
  </tr>
</table>

`source_path` doubles as the ADL constraint hook. Modelers declare it in the archetype to say "this ELEMENT is linked to path X of the source", and the same field carries provenance into recorded data so AQL and validators can tell resolved values apart from hardcoded ones.

#### How it works in ADL

A `procedure_diagnosis_link` CLUSTER targets `problem_diagnosis` and respec'd `reason` and `severity` ELEMENTs:

```adl
CLUSTER[id1] matches {                                          -- procedure_diagnosis_link
  shared_link matches {
    DV_REFERENCE[id2] matches {
      internal_ref matches {
        DV_EHR_URI[id3] matches {
          archetype_id matches {/openEHR-EHR-EVALUATION\.problem_diagnosis\.v\d+/}
        }
      }
    }
  }
  items cardinality matches {1..*; ordered} matches {
    ELEMENT[id4] occurrences matches {0..1} matches {           -- reason (respec of source diagnosis)
      source_path matches {"/data[at0001]/items[at0002]/value"}
      value matches {
        DV_CODED_TEXT matches {
          defining_code matches {[ac0010]}                      -- value set narrowed locally
        }
      }
    }
    ELEMENT[id5] occurrences matches {0..1} matches {           -- severity
      source_path matches {"/data[at0001]/items[at0073]/value"}
      value matches {
        DV_ORDINAL matches { ... }
      }
    }
  }
}
```

Reading: the CLUSTER's `shared_link` constrains the link to a `problem_diagnosis`. Each child ELEMENT carries a `source_path` constraint pointing into the linked target. The local `value` constraint defines what the resolved value must look like and what the modeler accepts when hardcoding.

In ADL the constraint pins the path. At recording time the engine fills `ELEMENT.source_path` and the resolved `value`. If the modeler chose to hardcode, `source_path` stays unset and the modeler enters the value directly.

#### Modes at recording time

- `shared_link` set, ELEMENT value resolved from `source_path`. Engine fills the value, validates it against the local constraint, and stores it.
- `shared_link` unset, ELEMENT carries a hardcoded value. Validated against the local constraint only.
- Mixed within one CLUSTER is fine. Some ELEMENTs may resolve, others may be hardcoded, depending on what the source provides.

#### Open questions

- Whether `shared_link` should be `DV_REFERENCE` or a stricter type that forbids `proxy` (the CLUSTER replaces that role).
- Resolution behaviour when the linked target's value at `source_path` no longer satisfies the local constraint.
- Whether `source_path` on ELEMENT should be enforced as immutable once set, since changing it would silently rebind provenance.
