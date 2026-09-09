# VEP #457: Mutable Virtual Machine Display Name

## VEP Status Metadata

- Author: Shirly Radco
- Enhancement issue: [#457](https://github.com/kubevirt/enhancements/issues/457)

### Target releases

<!--
A PR must update this section during the planning phase of a given release in order to track it.
Please avoid targeting future releases in this section. Only capture the upcoming release.
-->

- This VEP targets alpha for version:
- This VEP targets beta for version:
- This VEP targets GA for version:

### Release Signoff Checklist

Items marked with (R) are required *prior to targeting to a milestone / release*.

- [ ] (R) Enhancement issue created, which links to VEP dir in [kubevirt/enhancements] (not the initial VEP PR)
- [ ] (R) Alpha target version is explicitly mentioned and approved
- [ ] (R) Beta target version is explicitly mentioned and approved
- [ ] (R) GA target version is explicitly mentioned and approved

## Overview

Propose `kubevirt.io/display-name` as a **provisional, optional annotation** on a VirtualMachine's top-level `metadata.annotations`. It holds one mutable, human-readable inventory name. A reader presents its nonempty value when present and otherwise presents `metadata.name`. There is no blanket defaulting or backfill. Users may set the annotation; the naming service writes it only when a new VM's requested effective name collides and needs a unique suffix, or when rolling back a raced edit. The key and its community ownership require agreement before implementation as a supported convention.

Changing this annotation changes presentation and inventory discovery, not Kubernetes identity or VM runtime configuration. The proposed delivery scope is **HCO-managed installations**, with a dedicated naming service deployed by HCO as an operand. A separate naming operator is not required. Avoiding changes to `kubevirt/kubevirt` and `kubevirt/api` is a feasibility target, not yet a demonstrated result. When enabled, the default policy **converges** effective names to namespace uniqueness, including names derived from `metadata.name`; customers can explicitly opt out. A successful direct create is initially provisional: a duplicate can be stored and visible until the service assigns a stable suffix. This is an eventual-uniqueness contract, not a pre-persistence uniqueness guarantee. Standalone KubeVirt has no managed display-name uniqueness from this proposal.

## Motivation

Administrators need to update inventory names when an organization changes its naming convention, retain or align a source platform's inventory name during migration, and remedy customer naming-policy violations without replacing a VM. Source names may contain spaces, punctuation, or Unicode that `metadata.name` and label values cannot represent. Supported console and CLI presentation, editing, and discovery address those inventory tasks while automation continues to use stable Kubernetes identity.

This is a partial answer to a full-rename RFE. It does **not** rename literal `metadata.name`, Pods, DataVolumes, PVCs, Secrets, or ConfigMaps. A customer policy that inspects those literal resource names remains unmet; the remaining RFE requirements need customer validation. No regulation is assumed to mandate a particular VM-name pattern.

KubeVirt previously removed object rename after a reported data-loss case involving `dataVolumeTemplates` ([kubevirt/kubevirt#5564](https://github.com/kubevirt/kubevirt/pull/5564)). The lesson for this proposal is to preserve object identity, not to assume every inventory integration already understands a display name.

## Goals

- Set, change, and clear an inventory name on running or stopped VMs without a naming-induced guest restart, VMI recreation, or migration.
- Give UI, CLI discovery, import, and integrations one effective-name rule while keeping namespace, `metadata.name`, and UID available for identification.
- Expose the effective name as a `display_name` label on `kubevirt_vm_info` through `kubevirt-observability-controller`, retaining canonical monitoring identity.
- Keep existing VMs readable without blanket defaulting, backfill, or mass writes; annotate only new VMs needing collision repair.
- Document lifecycle behavior for create, pool scale and update, clone, restore, import, migration, delete, backup, and policy transitions.
- Deliver namespace-scoped effective-name conflict resolution by default in HCO-managed installations, an explicit customer opt-out, and a safe activation path for existing deployments.

## Restrictions

- In an HCO-managed installation with the feature enabled, duplicate effective names within one namespace may persist briefly during reconciliation, remain during a visible activation-pending state, or remain under an explicit `Disabled` opt-out. If the naming service is unavailable, convergence is delayed and the product must show the unresolved state. Standalone KubeVirt has no managed rule. Every discovery result and destructive confirmation must expose canonical identity; a consumer must never silently choose the first match.
- The displayed name is not an API object identifier, authorization key, policy selector, or positional operand to existing `virtctl start`, `stop`, and similar commands. Discovery resolves a canonical namespace/name/UID before any action.
- The base annotation convention does not provide native annotation field or label selection, server-side partial matching, or a guaranteed cluster-wide search index. Client-side discovery needs authorized list access.
- The default policy repairs conflicts after VM persistence, including creation without an annotation. Known conflicts on explicit edits and clears are rejected; a race missed by validation must be detected and rolled back to the last settled value. A successful API write does not mean its requested effective name has settled.

## Non Goals

- Literal Kubernetes object rename or automatic renaming of related resources, guest hostname, MAC/IP configuration, Service selectors, or runtime configuration.
- Implicit command resolution by display string. Display-name lookup must use an explicit mode, separate from ordinary resource-name operands.
- Automatic duplicate-name health alerts, metrics, or Events as a substitute for a naming policy. The display-name label on `kubevirt_vm_info` is an inventory field, not proof of uniqueness or settlement. Events are not durable audit history.
- A new search service, persistent full browser VM cache, or regex search backend without measured need.

## Definition of Users

- VM administrators editing names and resolving ambiguous inventory entries.
- Migration administrators preserving source inventory text.
- CLI and GitOps users who need predictable discovery and field ownership.
- Console, importer, backup, monitoring, and other integration developers.
- Cluster administrators resolving existing conflicts, activating default reconciliation, or choosing an explicit opt-out.

## User Stories

- As an administrator, I can display `SAP Database Server` for `production/finance-vm-001`, then change or clear it without changing the running guest.
- As an importer, I can request a source inventory name and see the final assigned name if a collision requires a suffix.
- As a CLI user, I can search resource and effective display names, see every match with namespace/name/UID, explicitly request display-name lookup, and act on a selected canonical target.
- As a cluster administrator, I can identify and resolve legacy conflicts before default reconciliation activates, or explicitly opt out, without silently renaming existing VMs.

## Repos

| Repository or component | Responsibility and tracking |
| --- | --- |
| `kubevirt/enhancements` | This VEP and issue #457 define the proposed convention and consumer contract. |
| `kubevirt/kubevirt` and `kubevirt/api` | No code or API change is planned. Existing VM metadata, pool template propagation, clone filters, and restore patches are integration surfaces; supported-release testing must prove they are sufficient. Pool or lifecycle behavior may require revisiting this boundary. |
| [`kubevirt/kubevirt-observability-controller`](https://github.com/kubevirt/kubevirt-observability-controller) | Add and test `display_name` on `kubevirt_vm_info`; update metric documentation and verify recording-rule, alert, dashboard, and scrape-migration compatibility. This companion change does not require a `kubevirt/kubevirt` or VM API change. |
| `kubevirt-ui/kubevirt-plugin` | Companion console work for presentation, search, editing, creation, clone/restore dialogs, and identity-aware confirmations. |
| External CLI plugin | Deliver supported discovery, explicit display-name lookup, canonical-target actions, and coordinated name changes without changing `virtctl`. |
| MTV/Forklift and other importers | Companion import mapping, conflict choice, and assigned-name reporting. |
| Dedicated naming service (code location to be decided) | Observe VMs, settle effective-name ownership and repair new-VM collisions; validate known edit/clear conflicts; handle naming status and VM pool integration. A new repository or separately installed operator is not required. The earlier admission-claim prototype explores a different, strict contract; it is not an implementation tracker. |
| HCO | Own the customer policy and lifecycle gate in the HyperConverged CR; deploy, configure, health-check, and upgrade the naming service and any status API, validation webhook, RBAC, and certificate resources; define their safe disablement and uninstall sequence. |
| `kubevirt/user-guide` and product docs | Naming, search, lifecycle, GitOps, compatibility, and policy documentation. |

The **proposed implementation has no planned `kubevirt/kubevirt` or `kubevirt/api` changes**: VM metadata stores the annotation, an HCO-managed service validates edits, watches persisted VMs, repairs conflicts, and reports naming status outside the VM API, and `kubevirt-observability-controller` adds the display-name metric label. Delivery requires an observability-controller version carrying that label and a supported metric-ownership migration. This is a conditional scope, not a finding that KubeVirt changes are unnecessary. In particular, the current VM pool controller can reapply a template annotation over a repaired child name, and clone/restore behavior varies by path and supported release. Before claiming an HCO-only delivery, integration tests must show stable pool-child assignments and usable direct create, Clone, and Restore flows on every supported KubeVirt version. If an HCO-managed integration cannot meet those requirements, this VEP must add the necessary KubeVirt API or controller changes, ownership, feature-gate decision, and release compatibility work.

## Design

### Terminology and placement

These are proposed product terms, not universal Kubernetes definitions:

| Term | Intended meaning | Choice here |
| --- | --- | --- |
| Display name | One mutable, human-readable inventory name; presentation falls back to the resource name | Use for the proposed annotation contract. |
| Alias | An alternative address or selector, potentially one of several | Reserve for a separately specified command-resolution or selector feature. |

A display name may be searchable and unique within its namespace without becoming Kubernetes identity. The annotation belongs to the **VirtualMachine top-level metadata**, not `spec.template.metadata`. It is an entry in Kubernetes's existing [annotations map](https://kubernetes.io/docs/concepts/overview/working-with-objects/annotations/), not a dedicated CRD schema field. Its presentation role, arbitrary human text, and precedent of the console's description annotation favor this placement. Standardizing it still requires community review; using metadata does not bypass governance.

### Effective-name and validation contract

For a VM, `effectiveName = nonempty(metadata.annotations["kubevirt.io/display-name"]) ? annotation : metadata.name`. Absence is normal. A clear/reset operation removes the key; it does not store the resource name. Readers also treat an explicit empty value submitted by a direct client as unset. Supported writers remove the key for an empty edit. Neither the key nor its value is copied into VMI template metadata by this convention.

The proposed product limit is 1–256 Unicode code points for a **nonempty** value, subject to review with UI, CLI, and import owners. Internal spaces, punctuation, and Unicode are allowed. Supported editors reject leading/trailing whitespace, line breaks, and control characters; they do not silently trim or normalize the user's text. The exact Unicode whitespace/control set and final length limit remain decisions. A Kubernetes annotation's overall storage limit is not product validation. UI, CLI, and importer writers must validate their own input; all readers must safely render even unexpected values as text, never as markup. Server validation must apply the agreed value rules and reject known conflicts on explicit set/change/clear, including direct API writes. Validation alone cannot guarantee uniqueness against concurrent writes; the reconciliation contract below handles races and outages.

| Approach | Benefit | Cost |
| --- | --- | --- |
| Populate annotation from resource name | Annotation-only consumers always have a value. | Duplicate storage, defaulting/backfill, ambiguous clear semantics, source-name inheritance on clone. |
| Optional annotation with read-time fallback **(proposed)** | Existing VMs work without blanket writes; clear normally returns to the resource name. | Every consumer implements fallback; a conflicting new fallback requires an annotation repair, and a conflicting clear is rejected or rolled back. |

Fallback can collide with an explicit name: VM `sap-db` has no display annotation; VM `vm-002` has annotation `sap-db`; both would display as `sap-db`. The default policy observes both after creation, retains the settled holder, and adds a suffix to the newer VM's display annotation. Thus even a new VM created without the key can gain one **only** to repair a collision. The scope is **effective names**, not merely nonempty annotation values. Create without a key, clear, recreation, and activation all participate. A known conflicting clear is rejected and leaves the annotation in place; a concurrent clear that slips through validation is rolled back to its last settled annotation. Suffixing a clear would defeat the requested absence.

Resource names and effective names occupy distinct lookup modes. For example, VM `sap-db` may have display annotation `Production DB`, while VM `vm-002` has display annotation `sap-db`. Their effective names differ, so both remain allowed after convergence. Ordinary CLI operand `sap-db` identifies the first VM by `metadata.name`; explicit display-name lookup for `sap-db` identifies the second. A combined inventory search returns both with namespace/name/UID and the matched field. Populating annotations on every VM would **not** remove this cross-mode search ambiguity. Preventing it would require reserving display names against every other VM's resource name even when that other VM has a different effective name. That stronger restriction is not proposed.

### Existing behavior used as design evidence

At inspected `kubevirt-plugin` main revision [`e77daba`](https://github.com/kubevirt-ui/kubevirt-plugin/commit/e77daba9b08fd46f25091f3747113a19befe3899), the [description editor](https://github.com/kubevirt-ui/kubevirt-plugin/blob/e77daba9b08fd46f25091f3747113a19befe3899/src/utils/components/DescriptionItem/components/DescriptionItemDescription.tsx) writes nonempty input and deletes the key on empty input. The [VM description filter](https://github.com/kubevirt-ui/kubevirt-plugin/blob/e77daba9b08fd46f25091f3747113a19befe3899/src/views/virtualmachines/list/filters/getDescriptionFilter.ts) calls `fuzzyCaseInsensitive`, which lowercases both strings and calls [`fuzzysearch`](https://github.com/bevacqua/fuzzysearch/blob/master/index.js). That algorithm matches the search characters in order, including non-contiguous matches; it is neither regex nor substring-only matching. The [VM list](https://github.com/kubevirt-ui/kubevirt-plugin/blob/e77daba9b08fd46f25091f3747113a19befe3899/src/views/virtualmachines/list/VirtualMachinesList.tsx) filters watched resources before local pagination; its [data source](https://github.com/kubevirt-ui/kubevirt-plugin/blob/e77daba9b08fd46f25091f3747113a19befe3899/src/views/virtualmachines/list/hooks/useVMDataSource.ts) and [resource hook](https://github.com/kubevirt-ui/kubevirt-plugin/blob/e77daba9b08fd46f25091f3747113a19befe3899/src/views/virtualmachines/search/hooks/useAccessibleResources.ts) use cluster-wide or per-namespace watches and a 10,000-object fetch limit on some paths. That code does not prove a complete inventory search or server-side prefix autocomplete at all scales.

At inspected `kubevirt` revision [`c418f32`](https://github.com/kubevirt/kubevirt/commit/c418f32d6fefd864f19d3a7f436dd6bfa8827036), [clone patch generation](https://github.com/kubevirt/kubevirt/blob/c418f32d6fefd864f19d3a7f436dd6bfa8827036/pkg/virt-controller/watch/clone/vm-target.go) applies top-level annotation filters. [Restore](https://github.com/kubevirt/kubevirt/blob/c418f32d6fefd864f19d3a7f436dd6bfa8827036/pkg/storage/snapshot/restore.go) copies snapshot top-level annotations when creating a missing target; for an existing target it deep-copies that target, replaces its spec, and retains its top-level annotation. `spec.patches` are applied only on new-target VM creation in this implementation. Supported-release behavior must be checked by each integration; the new-target patches below are not an in-place restore override.

At the same `kubevirt` revision, the VM pool controller [copies template annotations into each new child VM](https://github.com/kubevirt/kubevirt/blob/c418f32d6fefd864f19d3a7f436dd6bfa8827036/pkg/virt-controller/watch/pool/pool.go#L949-L950) and [reapplies them on child updates](https://github.com/kubevirt/kubevirt/blob/c418f32d6fefd864f19d3a7f436dd6bfa8827036/pkg/virt-controller/watch/pool/pool.go#L1086-L1087). A static display-name annotation on a multi-replica pool template would give its children the same requested effective name. Later pool updates could overwrite a repaired child's annotation and create a reconciliation loop; known-conflict validation could instead reject the pool's child update and stall rollout. Pool create, scale, rollout, and activation therefore need a defined and tested naming path before default conflict resolution can ship. If HCO cannot keep per-child assignments stable without changing KubeVirt's pool controller, the no-KubeVirt-code target must be revised.

### Console and discovery contract

VM list rows, detail headers, topology, related-resource associations, and success/failure messages show the current effective name with canonical namespace/resource name accessible. Related-resource views can show their association with the renamed VM without copying its display string onto every resource. Creation, clone, restore, and rename dialogs distinguish requested text, provisional effective text, settled assigned text, and resource identity. Destructive confirmations include namespace/name and, where available, UID. Set/change/clear preserve unrelated annotations. A visible naming condition reports provisional, conflict, settled, and unavailable states; a successful VM API response alone does not establish that the displayed name is settled.

Combined inventory search checks both `metadata.name` and effective display text and labels the field that matched. The proposed normal partial mode follows the verified description filter: case-insensitive ordered-character (subsequence) matching. Exact matching is also available for CLI lookup. Matching for discovery is distinct from equivalence for uniqueness; the resolver needs an explicit case and Unicode-normalization rule. Zero matches report no result; one match returns its canonical identity; many return all matches and require explicit selection. Exact effective-name lookup can also return several VMs while a conflict is unresolved. Even after convergence, combined search can return both VMs in the cross-mode example above. Never choose the first match. Bound autocomplete suggestions, while a separately accessible filtered result path must cover the complete selected, authorized inventory. Reuse the existing watch/list path where feasible, measure its limits, and do not equate current-page filtering or a 10,000-object limit with complete inventory search.

### CLI, API, and GitOps

Ordinary direct VM create remains supported. The following direct edits are also supported when the HCO naming service is active: validation rejects a known conflict or an edit on a VM whose initial name has not settled, and a racing conflict is rolled back after persistence. The caller must read the naming condition to distinguish API acceptance from settlement:

```bash
oc annotate vm finance-vm-001 -n production \
  kubevirt.io/display-name='SAP Database Server' --overwrite
oc annotate vm finance-vm-001 -n production kubevirt.io/display-name-
```

A supported external CLI plugin is required delivery work; this proposal does not add a `virtctl` command. The plugin can list authorized VMs and filter client-side. Its exact syntax is open. Ordinary `virtctl`, `oc`, and plugin action operands continue to resolve **only** `metadata.name`. A user must explicitly request display-name lookup; that mode matches effective names, including fallback names, and never silently falls back to resource-name lookup. A separate combined inventory search offers exact and case-insensitive subsequence search over both fields, returns **all** matches with namespace/name/UID, matched field, and naming condition, and reports ambiguity for a single-target request. Exact display-name lookup may return multiple matches during reconciliation, in `Disabled`, or without HCO. Mutations use explicit canonical identity and recheck UID or resource version as needed to avoid stale-target actions. The plugin must report rejected edits, provisional names, conflict repair, and the settled assigned name. Namespace scope and list RBAC are explicit: annotation discovery may require LIST where a direct named GET would suffice. Neither `oc get --field-selector` nor Kubernetes label selectors select annotation values, and [field selectors](https://kubernetes.io/docs/concepts/overview/working-with-objects/field-selectors/) do not provide regex or prefix operators. Regex search is possible future CLI work, not an accepted base requirement.

Until that CLI exists, this pipeline illustrates **combined exact, case-sensitive discovery within one authorized namespace**. It lists VMs, computes the effective name locally, and returns every match rather than selecting one:

```bash
oc get vm -n production -o json |
  jq -r --arg query 'SAP Database Server' '
    .items[]
    | (.metadata.annotations["kubevirt.io/display-name"] // "") as $stored
    | (if $stored == "" then .metadata.name else $stored end) as $display
    | select(.metadata.name == $query or $display == $query)
    | [.metadata.namespace, .metadata.name, .metadata.uid, $display]
    | @tsv
  '
```

The pipeline is a manual workaround, not the proposed supported CLI: it does not label the matched field, provide case-insensitive subsequence matching, a complete cross-namespace search, or the single-target ambiguity and stale-identity checks.

Default `AutoResolve` GitOps must not fight a controller-written suffix. A naming-service-managed request or an equivalent field-ownership contract declares the requested base name separately from the assigned VM annotation; the service reports provisional/conflict/settled status and owns any repair write. GitOps must read back the assigned name and not continuously restore the conflicting base value. Direct VM manifests remain usable; when they declare the annotation, the integration must provide a supported way to relinquish that field after creation or to express intent separately. Exact edit declarations reject known conflicts. Request API, ownership, readback, retry, and dry-run behavior must be specified before delivery. Dry-run can validate current input but cannot promise the final assigned name.

### Monitoring contract

Add a `display_name` label to the existing `kubevirt_vm_info` gauge in **`kubevirt/kubevirt-observability-controller`**. Its value is the VM's effective name: the nonempty top-level `metadata.annotations["kubevirt.io/display-name"]`, otherwise `metadata.name`. Absence, an explicit empty value, or a clear therefore reports the resource name. Read from VM metadata, not VMI-template annotations. The collector reports the latest observed value, including a repair suffix or rolled-back edit; the naming condition remains the source of settlement and availability status.

The observability-controller already [watches KubeVirt resources and exposes their metrics](https://github.com/kubevirt/kubevirt-observability-controller#overview), with its own [`kubevirt_vm_info` collector](https://github.com/kubevirt/kubevirt-observability-controller/blob/main/pkg/monitoring/metrics/vmstats_collector.go). Add the label, collector tests, and metric documentation there. This requires companion observability-controller work, with no planned change to `kubevirt/kubevirt` or `kubevirt/api`. Deployments still using the legacy KubeVirt exporter need the supported observability-controller migration before this metric requirement is met; this proposal does not require a legacy-exporter backport. Define the supported exporter version and scrape ownership so old and new sources do not double-count VMs during rollout.

Keep `name` and `namespace` as canonical metric labels and join keys; preserve other existing identity labels where the deployed exporter provides them. Use `display_name` for inventory presentation, not alert identity or cross-metric joins. The label follows the same fallback rule whenever the updated collector runs, including standalone KubeVirt, HCO gate-off, and `Disabled` policy, without implying managed uniqueness. Do not add it to every VM/VMI metric.

A name change creates a new Prometheus label set. The previous series stops receiving samples while retained historical samples keep the old text. Emit one current `kubevirt_vm_info` sample per VM and measure series churn from edits and suffix assignment. Audit observability-controller rules, dashboards, HCO integrations, and remote-write consumers for implicit full-label matching, aggregation, time-window queries, and display-text exposure. Verify rename and clear behavior without changing counts, joins, alert identity, or alert duration. During mixed-version rollout, consumers must tolerate an absent `display_name` label and fall back to canonical text.

### Default eventual uniqueness and explicit opt-out

Proposed policy modes are `AutoResolve` (default), `Warn` (transition only), and `Disabled` (explicit customer opt-out); these names and the policy API are illustrative. The scope is one namespace and the equivalence rule applies to **effective names**, whether the annotation is present or absent. `AutoResolve` accepts direct VM creation, then repairs a conflicting new VM by writing a unique display annotation. It rejects a **known** conflict on explicit set/change/clear; validation is advisory against races, which the reconciler must repair. `Disabled` accepts duplicates without repair and must be a deliberate, visible choice. `Warn` inventories and reports legacy conflicts without renaming; it is not a silent permanent default or a uniqueness guarantee. `AutoResolve` promises convergence after writes quiesce and the service is healthy, not uniqueness at every instant. A time bound and availability target must be established by implementation testing before release.

New HCO-managed deployments with the HCO feature gate active start in `AutoResolve`. Existing deployments enter a visible activation-pending state. The service records a namespace inventory boundary, inventories every VM's effective name, invalid legacy values, and VM pool templates, and reports conflicts with namespace/name/UID. Administrators resolve conflicts among VMs present at that boundary or explicitly opt out; activation does not silently rename those legacy VMs. Creates during the pending transition remain visible as provisional and must be included in the final inventory. Once active, the resolver may suffix a conflicting VM created after the boundary, even if its create preceded activation. Unlike strict pre-persistence enforcement, no scan/write admission fence or precreate claims are required: the active reconciler also observes writes after activation. Pending status must not claim uniqueness, and an interrupted scan resumes safely with the same protected legacy set or restarts a clearly reported inventory. Unrelated updates to legacy VMs must not be rejected solely because an unchanged value predates validation. There is no blanket annotation backfill.

The naming service observes persisted VMs and maintains durable, UID-bound settled assignment state (including the last accepted annotation or absence), separate from a requested base name. It records the base request before overwriting a new VM's annotation, so a restart does not lose intent. A VM without current settled state is `Pending` or `Unknown`, including immediately after successful direct create and during service outage. A condition surface available to UI and CLI reports the canonical VM identity, requested/current/assigned effective names, last observed VM version, and `Pending`, `Conflict`, `Settled`, or `Unavailable`; a stale condition must never be presented as current. The exact API is open. When several VMs request an equivalent name, an existing settled holder retains it; if none is settled, the controller chooses a winner deterministically by creation timestamp then UID. It assigns the losing **new** VM a stable suffix derived from its persisted UID or durably recorded randomness, checks the entire resulting effective name under the chosen equivalence and length rules, and retries with a longer/different suffix if necessary. Reconciliation writes use resource-version preconditions and are idempotent across restarts, delete/recreate, and retries. It must never relabel an unrelated existing VM to make room. A suffix is not guaranteed to be a fixed short length or globally collision-free without checking.

For an explicit edit or clear, validation rejects known conflicts and rejects edits while that VM's initial assignment is unsettled. The last settled name remains held by that UID until its edit settles or rolls back; a new VM requesting that name waits for the outcome before final assignment. If concurrent writes pass validation and collide, the service rolls the losing edit or clear back to its durable last settled annotation using a conditional update; it does **not** silently suffix that edit. It reports the rejected/rolled-back intent and preserves unrelated VM changes. A newer edit wins only after its own current value is re-evaluated; stale rollback may not overwrite it. If rollback cannot complete, status remains visibly conflicted and automatic convergence is not yet achieved. An API success can therefore be followed by a visible rollback. This behavior, including recovery after a controller outage, needs a tested failure and status contract.

Automatic suffixing applies by default to **new VM creation**, including child VMs created through clone, import, and new-target restore, even when their requested name came from inherited metadata. The base request and final assigned name remain distinct and visible. A creation without the annotation normally keeps it absent; a conflicting fallback gains the repair annotation. Clearing a repaired name is allowed only if the fallback is available. Clone/restore/import clients must report the settled name, not treat initial API success as final assignment. Existing VMs are not renamed on activation. Pool templates that repeatedly overwrite child repairs require a separate supported policy or KubeVirt change; convergence cannot be claimed while such a loop exists.

### Lifecycle and runtime

| Action | Proposed behavior | Integration or enforcement impact |
| --- | --- | --- |
| Install/enable base feature | New HCO-managed deployments with the HCO gate active start with `AutoResolve`; existing VMs keep absent annotations and show resource-name fallback. | Existing deployments detect legacy conflicts and activate after remediation or explicit opt-out; no blanket backfill or restart. Standalone KubeVirt has no managed policy from this proposal. |
| Console creation | Optional explicit name; otherwise fallback. | Return provisional identity and show the settled name or conflict repair when reconciliation completes. |
| CLI/YAML/GitOps creation | Ordinary direct VM creation remains usable. Conflicting new VMs may persist briefly before repair. | Stable suffix, visible condition and readback; GitOps field ownership must avoid restoring a repaired value. |
| VM pool create/scale/template update | Children without a display annotation use their distinct resource-name fallbacks. A static display annotation copied from a multi-replica template can produce provisional duplicates; a later pool update can overwrite a child's repair. | Define template and pool-owned child validation or per-child naming policy, including rollout and existing-pool activation. Prove that an HCO-managed integration suffices before retaining the no-KubeVirt-code target. |
| Edit running or stopped VM | Change the top-level annotation after its initial assignment settles. | Reject known conflicts; roll back raced conflicts to last settled annotation; no naming-induced guest/VMI/network/storage action. |
| Clear/reset | Remove annotation and show resource name if available. | Reject known fallback conflict; roll back raced conflicting clear without populating a default. |
| Bulk changes | Report each VM's result and support stable retries. | No cross-VM transaction promise. |
| MTV/Forklift import | Request source inventory text. | Report provisional and final assigned name; suffix on collision. |
| Clone | Proposed workflow strips source display key and target falls back unless an explicit name is requested. | Existing Clone API copies annotations unless a client filters them; inherited conflicts on a new child also receive a suffix. Report settled target name. |
| Restore to missing/new target | Snapshot annotation is inherited unless workflow overrides/clears it. | Suffix a conflicting new target; show final name and avoid a repair loop with the restore workflow. |
| Restore over existing target | Preserve target's current top-level display annotation in inspected implementation. | Treat an actual annotation change as an edit; do not assume new-target patches run. |
| Snapshot | Ordinary VM metadata can be captured. | Settled assignment state is not transferable identity. |
| Backup/restore or disaster recovery | Respect destination identity and policy. | Verify each integration and rebuild UID-bound settlement state; do not generalize VMRestore behavior. |
| Start/stop/restart/pause/delete/migrate after discovery | Act on selected canonical identity. | Reject ambiguous or stale lookup; show identity in confirmation. |
| Live migration | Name remains VM metadata and may be edited. | Test concurrent name-only edits; no cosmetic migration block is proposed. |
| Delete VM | Remove VM, then retire its UID-bound settled assignment. | Reconcile before reusing its name; a newly persisted VM can still be provisional. |
| Recreate same resource name | New UID is a new identity. | Establish a new assignment; ignore stale status for the old UID. |
| Shared Secrets/ConfigMaps and dependencies | Retain names and references. | No automatic dependent rename. |
| Policy activation/change/disable | New deployments auto-resolve by default; upgrades resolve legacy conflicts or explicitly opt out. | Visible pending/active/degraded state and an inventory-to-watch handoff; re-enable rescans all VMs. |
| Operator outage/upgrade/uninstall | Defined availability and recovery behavior. | Mark assignments stale/unavailable; stop promising prompt convergence; recover without overwriting newer edits or leaving admission blocked. |
| Monitoring/inventory/backup integrations | Observability-controller exposes the effective name in `kubevirt_vm_info{display_name=...}`; joins retain canonical identity. | Deliver the collector label and docs; verify scrape ownership, mutable-label compatibility, and older exporters. No default naming-health metric is proposed. |

Clone [annotation filters](https://github.com/kubevirt/kubevirt/blob/c418f32d6fefd864f19d3a7f436dd6bfa8827036/staging/src/kubevirt.io/api/clone/v1beta1/types.go) can request the proposed strip behavior; this is not today's unconditional default. Filters are regular expressions, so the negative pattern anchors and escapes the dot to match only this key:

```yaml
spec:
  annotationFilters:
    - "*"
    - '!^kubevirt[.]io/display-name$'
```

For a restore **to a new target**, an existing annotation can be overridden using the Restore API's JSON patch string. The slash in the key is escaped as `~1`. This excerpt is illustrative; an absent annotation needs an `add` at the appropriate existing parent or a client-side omission strategy, rather than an unconditional `remove` patch:

```yaml
spec:
  patches:
    - '{"op":"replace","path":"/metadata/annotations/kubevirt.io~1display-name","value":"SAP Database Server (2)"}'
```

Storage preparation may precede VM creation. Naming settlement must not be mistaken for a transaction across VM and storage resources; clone and restore status needs to report the assigned name after VM creation. Normal live migration and restore have their own availability effects. A name edit itself neither changes `spec.template` nor requires a migration. `metadata.name`, UID, storage references, owner references, guest hostname, MAC/IP, Service selectors, and runtime configuration stay stable as a result of that edit.

### Naming-service deployment and ownership

**Proposed packaging: a dedicated HCO-managed naming service.** At inspected HCO revision [`7ecd1cb`](https://github.com/kubevirt/hyperconverged-cluster-operator/commit/7ecd1cbc75775ba9579e54854db2026957eec7cc), HCO [manages supporting webhook operands](https://github.com/kubevirt/hyperconverged-cluster-operator/blob/7ecd1cbc75775ba9579e54854db2026957eec7cc/controllers/operandhandler/operandHandler.go#L59-L67) and could deploy the naming service's Deployment, Service, settlement/status resources, validation webhook, RBAC, and certificates. HCO configures, health-checks, and upgrades those resources and defines disablement and uninstall. The customer policy source of truth is the HyperConverged CR, with `v1` and supported `v1beta1` configuration and upgrade behavior defined before release. Neither a separately installed operator nor a new repository is required; code location is an implementation choice. The service observes persisted VM UIDs, settles assignments, repairs new-VM conflicts, validates known edit/clear conflicts, and publishes status. Its webhook has **no precreate reservation side effect**. Keep validation availability separate from HCO's inspected [single-replica webhook Deployment](https://github.com/kubevirt/hyperconverged-cluster-operator/blob/7ecd1cbc75775ba9579e54854db2026957eec7cc/deploy/operator.yaml#L132-L145) unless that deployment receives an availability redesign. The naming webhook runs alongside KubeVirt's existing [VM validating webhook](https://github.com/kubevirt/kubevirt/blob/c418f32d6fefd864f19d3a7f436dd6bfa8827036/pkg/virt-operator/resource/generate/components/webhooks.go). Avoiding `virt-api` and `virt-operator` changes still depends on pool, clone, and restore tests on supported versions.

The earlier strict-uniqueness prototype created a claim during admission. [Admission does not guarantee persistence](https://kubernetes.io/docs/reference/access-authn-authz/extensible-admission-controllers/#side-effects): on OpenShift Virtualization 4.22.12, a later KubeVirt validation rejection left an unbound claim; a corrected retry could bind after a prototype change, but a retry-pinned name could remain reserved indefinitely. Twenty overlapping creates persisted one VM in another test, yet that did not prove safe claim release. This VEP now chooses post-persistence settlement, so direct `oc create vm` needs no precreate claim or completion signal. These tests remain evidence for **why strict admission-time claims were not selected**, not evidence that the new controller algorithm works. A coordinated request may still solve GitOps intent and field ownership; it cannot replace ordinary direct creation.

API machinery and KubeVirt reviewers should examine the new settlement and validation race model: whether VM watch/list handoff, UID-bound assignment records, conditional repair, and eventual status reporting can meet the desired convergence objective without KubeVirt code changes. If the product later requires strict uniqueness **at API success**, it must reopen the direct-create completion and claim-release design; merely moving admission between HCO and KubeVirt does not supply an outcome signal. An upstream API-machinery dependency would then be possible, not assumed for this eventual model.

Kubernetes [ResourceQuota issue #25484](https://github.com/kubernetes/kubernetes/issues/25484) recorded quota usage increasing during admission even when later validation rejected object creation. Its specific validation case was fixed, but maintainers discussed the broader need for quota/storage atomicity and the absence of a post-storage outcome hook. The open [cross-resource ValidatingAdmissionPolicy request #133631](https://github.com/kubernetes/kubernetes/issues/133631) explicitly names uniqueness against other resources as a use case. A cross-resource read, even if added, would not serialize concurrent VM writes. These precedents explain why validation only rejects **known** edit conflicts in this design.

OpenShift [Route](https://github.com/openshift/api/blob/master/route/v1/types.go#L1355-L1372) demonstrates a related post-persistence contract: duplicate requested hosts can be stored, while the [router rejects a competing route for active host use](https://github.com/openshift/router/blob/main/pkg/router/controller/unique_host.go#L1135-L1175) and reports admission status. VM display names differ because the naming service changes the losing new VM's annotation to an assigned suffix and allows mutable edits. The comparison supports explicit requested-versus-assigned status, but does not prove this VM design or its convergence time.

The naming service and client owners must also cover KubeVirt's existing VMClone and VMRestore APIs. Supported client/request workflows may supply clone annotation filters, new-target restore patches, and final assigned-name reporting. In the earlier strict-claim prototype, a VM-create mutator stripped an inherited display name when KubeVirt's restore marker was present; a Restore CREATE mutator preserved an explicit display-name patch. On one tested version, diskless and storage-backed new-target Restore with an explicit patch completed with the requested name. A sequential storage-backed Clone completed with a fallback name; temporary verifier pods read the copied data from Restore and Clone PVCs. Concurrent snapshots of one PVC caused a CSI error, so successful tests ran in sequence. Under the **new** contract, raw Clone/Restore can initially persist a duplicate target and must later expose repair and the final assigned name. Existing-target restore keeps the target's top-level name in the inspected implementation. These observations do not test the proposed post-persistence resolver, supported-version matrix, import, or failure recovery.

Two packaging alternatives are:

1. **KubeVirt-owned naming.** `virt-api` could validate edits and a KubeVirt controller could settle names, making the policy available to standalone KubeVirt. This requires `kubevirt/kubevirt` changes, upstream lifecycle gating, and a different ownership and release plan. It may be needed if HCO cannot integrate with pool or other lifecycle controllers.
2. **Webhook and controller in HCO's existing binaries.** HCO could validate edits from its webhook binary and reconcile assignments in its manager. This avoids a dedicated service Deployment but couples naming availability and scale to HCO's binaries. Its current single-replica webhook deployment would need an availability redesign. It does not by itself solve pool, clone, restore, or GitOps integration.

[VEP-190 structured plugins](../190-kubevirt-structured-plugins/vep.md#admission-policies-and-webhooks) are another possible **registration wrapper**, not a replacement for the naming service. A `Plugin` can reference admission webhooks, but VEP-190 assigns their deployment and deletion to another component. Its launcher and node hooks act on VM runtime lifecycle rather than reconciling persisted VM metadata across a namespace. At inspected KubeVirt revision [`bf19f733`](https://github.com/kubevirt/kubevirt/blob/bf19f7334a2fe318f8c1e654880c8844d0265ff3/staging/src/kubevirt.io/api/plugin/v1alpha1/types.go), the admission-reference fields exist but `PluginStatus` is empty and no runtime consumer of those references was found outside API/generated code; the `Plugins` gate is [Alpha](https://github.com/kubevirt/kubevirt/blob/bf19f7334a2fe318f8c1e654880c8844d0265ff3/pkg/virt-config/featuregate/active.go). Requiring a `Plugin` now would add a KubeVirt gate and version dependency without providing settlement, suffix assignment, status, or pool integration. HCO should deploy the webhook and controller directly. Plugin registration can be reconsidered if it gains relevant readiness/lifecycle support on the supported release matrix.

The naming service, status/settlement storage, and validation must be ready before `AutoResolve` reports active. The validation webhook must have no claim side effect, and its failure behavior must distinguish naming edits from unrelated/status updates. HCO must specify whether creates fail closed while settlement is unavailable or are accepted as visibly unresolved; it cannot promise prompt repair while the service is down. Shutdown and uninstall must not leave an orphaned fail-closed webhook or falsely report settled status. HCO-managed deployment, RBAC, certificates, high availability, readiness, bounded reconciliation lag, and recovery are initial-delivery requirements. Admission-serving replicas must remain available through rolling upgrades if edit validation is fail-closed.

### Feature gate and policy controls

The annotation uses existing VM metadata and needs no new VM spec field or schema generation. Pool template propagation may need a behavior change if HCO cannot keep child repairs stable. The [KubeVirt feature lifecycle policy](../../../docs/feature-lifecycle.md#can-a-vep-choose-to-not-include-a-feature-gate) expects a VEP to justify omitting a KubeVirt gate. Under the proposed scope, behavior is implemented in the HCO-managed naming service and lifecycle availability is managed by HCO; KubeVirt binaries gain no behavior to gate if the no-code feasibility target holds. The display-name metric label is companion observability-controller behavior and reads existing metadata independently of the HCO naming gate; any collector rollout control belongs to that component, not a new KubeVirt gate. HCO's [feature-gate configuration](https://github.com/kubevirt/hyperconverged-cluster-operator/blob/main/docs/cluster-configuration.md#featuregates) therefore provides the proposed lifecycle control for the naming service (provisional gate name `vmDisplayName`, subject to HCO review). This is an HCO-owned gate, not a gate propagated to the KubeVirt CR. At Alpha it is off until explicitly enabled; at Beta it is on by default after activation; at GA the HCO lifecycle gate is removed. **Whenever the HCO gate is active, `AutoResolve` is the namespace policy default**, subject to visible activation-pending status on existing deployments. `Disabled` is the explicit customer opt-out, not the lifecycle gate's off state, and `Warn` is an activation transition rather than an ordinary default. HCO must define the policy and gate consistently in `v1` and supported `v1beta1` APIs and ensure clients can observe gate and activation state.

HCO gate-off or rollback leaves annotation data untouched: no defaulting, stripping, or backfill. Existing readers can still render stored text safely, but supported editing and display-name lookup clients report that the managed feature is unavailable; ordinary canonical-name VM operations continue. Generic metadata clients may write the annotation without a uniqueness guarantee while the service is inactive. HCO must not claim `AutoResolve` status in that state. Re-enabling requires a new conflict and pool scan because intervening writes may have introduced collisions; legacy duplicates remain activation-pending until resolved or explicitly opted out. A customer who wants the supported display-name workflow without repair uses `Disabled` instead of turning off the lifecycle gate. Standalone KubeVirt has neither this HCO gate nor the managed guarantee.

KubeVirt community review is still needed for the key, contract, and HCO-only scope of this VEP. A printer column for an annotation is separate work: verify JSONPath quoting for the slash-containing key and table-parser compatibility before proposing one. Console, importer, CLI, enforcement, and observability-controller integration each need owners and trackers for initial delivery. No production component is delivered by this document edit.

## API Examples

The proposed annotation is on the VM object, not the VMI template. Existing VM spec is omitted from this metadata excerpt:

```yaml
apiVersion: kubevirt.io/v1
kind: VirtualMachine
metadata:
  name: finance-vm-001
  namespace: production
  annotations:
    kubevirt.io/display-name: "SAP Database Server"
```

If the annotation is absent or empty, a reader shows `finance-vm-001`. Removing it leaves `metadata.name` and references unchanged. Under `AutoResolve`, a known fallback conflict rejects the clear, while a racing one may be rolled back after persistence. A new VM created without the key may later receive a suffixed annotation to resolve a fallback conflict. Under an explicit `Disabled` opt-out, two VMs can continue showing the same effective name; discovery still returns both canonical identities.

## Alternatives

| Alternative | Flexible human text | Native selection | Lifecycle and implementation cost |
| --- | --- | --- | --- |
| Standard display-name annotation **(proposed)** | Spaces, punctuation, Unicode | No native annotation selector; client-side discovery. | No VM spec field; required client/lifecycle integration, post-persistence settlement, and known-edit validation. |
| Label-based alias | Kubernetes label value syntax and length restrictions. | Native `kubectl -l`; `virtctl -l` only for commands/releases that actually support it. | Existing metadata mechanism; cannot preserve some source inventory strings. |
| `spec.displayName` with selectable fields | Flexible strings subject to schema rules. | Native exact field selection where CRD selectable fields are supported; no prefix/regex. | New API/schema field and spec-generation/lifecycle implications; uniqueness still needs coordination. |

A label is a valid distinct alias/selector proposal, not merely a `virtctl` workaround. Fabian's [`myalias=true` with `-l myalias`](https://github.com/kubevirt/enhancements/pull/461#discussion_r3978652517) is a presence-selector example; `-l kubevirt.io/alias=sap-db` is a key/value example. Neither defines a standardized KubeVirt alias API. `kubectl get vm -l ...` is supported native Kubernetes selection. The inspected [current `virtctl stop` implementation](https://github.com/kubevirt/kubevirt/blob/c418f32d6fefd864f19d3a7f436dd6bfa8827036/pkg/virtctl/vm/stop.go) requires one positional VM and does not register `-l`, so that particular reviewer example is not treated as a supported current command. Support for other `virtctl` actions must be checked against the target release before documenting them. The label alternative retains native Kubernetes selection and can be considered independently of this VEP.

The annotation is preferred for presentation semantics, existing description-edit precedent, arbitrary inventory text, simple absence/fallback, and no spec mutation on rename. Its real costs are client-side list/filter discovery, inconsistent readers until integrated, and required settlement/status infrastructure for eventual uniqueness. A spec field makes exact field selection convenient where supported but changes the VM API and generation on edit; it would still need coordination or post-persistence repair for uniqueness. A separate VMInfo resource adds API, controller, RBAC, and synchronization cost. Literal object rename remains a separate, unresolved RFE.

## Scalability

Optional annotation storage adds no blanket writes or backfill; conflict repair writes a new VM's annotation. Default `AutoResolve` adds a UID-bound settlement/status record per VM, validation for affected edits, and reconciliation on VM events. A metadata edit produces a normal VM update/watch event. Occasional CLI lookup can list and filter without a persistent full-object cache; metadata-only LIST still returns ObjectMeta and can still be large. Console watchers already load resources and apply filters before local pagination on inspected paths, but authorization scope, fetch limits, and incomplete pages must be measured before claiming complete search. Bound suggestions and fetch complete filtered results through a proven path. Benchmark namespace- and cluster-scoped discovery and metadata-only list/watch at representative fleet sizes, including large inventories; measure update frequency and response size. Do not add a search backend without evidence.

Measure settlement/status writes, validation latency, time from conflicting VM persistence to repair, and scale and availability during recovery after outages. Define a convergence objective and a visible degraded state; `Warn` transition reporting should be bounded and must not become an unreviewed alert or metric stream.

The `display_name` label adds no fan-out to the current VM inventory, but changes to the effective name can increase retained historical series. Benchmark label size, rename churn, scrape cost, and remote-write impact. Avoid copying mutable display text onto all runtime metrics.

## Update/Rollback Compatibility

Existing and older VMs have no key and still display their resource name. Direct VM create and direct annotation edits remain usable under active HCO `AutoResolve`, subject to known-conflict validation and later repair/rollback. A full-object replace can remove an annotation it does not preserve; merge patch or server-side apply of the chosen key avoids clobbering unrelated metadata but does not bypass validation. Managed-field ownership, GitOps drift, and older clients need tests. Supported readers must handle empty and unexpected values during mixed-version rollout. Disablement of any future UI integration does not erase stored annotations.

Default policy activation on existing deployments cannot silently rewrite legacy collisions or retroactively reject every existing VM. The transition reports conflicts and lets administrators resolve them or explicitly opt out. The service must establish UID-bound settlement for conflict-free legacy VMs, hand off its inventory to a continuing watch, and remain visibly pending while unresolved legacy conflicts remain; it need not provide a strict scan/write fence because new races are provisional and reconciled. Upgrades, policy disablement, and operator removal must specify status, validation, and recovery behavior. An annotation edit alone does not alter the VM spec or its runtime, but pool, clone, restore, import, and backup integrations must be checked by supported release. Literal name-based policies remain unaffected and may still require the original full-rename RFE.

Observability-controller upgrades add `display_name`; older exporters and rollback versions omit it. Dashboards and other consumers must handle both schemas and retain canonical queries. Metric-source migration must select one authoritative source or explicitly deduplicate across sources using canonical identity. The display-name feature does not depend on adding the label to the legacy `virt-controller` exporter.

Kubernetes audit records follow the configured [audit policy](https://kubernetes.io/docs/tasks/debug/debug-cluster/audit/); this proposal cannot promise old and new names at every audit level. If an implementation needs Events, it must specify emitter, retry, deduplication, and success criteria; Events are not a durable history.

## Functional Testing Approach

Product implementation work, not this VEP edit, needs coverage for:

- Absent, set, change, clear, and explicitly empty annotations; fallback, no blanket backfill, and conflict-only repair writes; Unicode, length/whitespace/control rules, and safe text rendering.
- Name-only edits to running/stopped VMs during ordinary operation and live migration, with no naming-induced VMI, guest, network, or storage change.
- Observability-controller `kubevirt_vm_info` label values for absent, empty, set, changed, cleared, suffixed, and rolled-back VM annotations; Unicode and escaping; top-level versus template placement; one current sample per VM; rename series turnover, canonical joins/counts, alert identity and duration, time-window queries, mixed exporter versions, and scrape-source migration.
- UI case-insensitive subsequence search over canonical and effective names, complete authorized result scope, bounded autocomplete, CLI identity output, explicit resource-name versus effective-name lookup, and zero/one/many ambiguity handling, including allowed cross-mode matches.
- Clone default workflow strip/override/automatic collision suffix; new-target restore inheritance and override/suffix; in-place target annotation preservation; snapshots made before display-name support.
- VM pool creation, scale-out, template updates, and rollout with absent or static display annotations; child fallback names, pool-owned updates that could overwrite repairs, loop prevention, existing-pool activation, and supported-version behavior.
- Default `AutoResolve`, explicit `Disabled`, visible activation-pending/`Warn`, explicit and fallback collisions, allowed display-name/resource-name cross-mode equality, case/Unicode equivalence, and provisional versus settled lookup.
- HCO Alpha gate-off, Beta gate-on, and GA gate-removal behavior; stored annotations and canonical VM operations across gate transitions; conflict detection before reactivation after gate-off writes.
- Parallel direct creates and racing edits/clears; deterministic winner, stable UID-based suffix, suffix collision retry, last-settled rollback, same-VM retries, resource-version conflicts, tampering, delete/recreate, other-webhook rejection, dry-run, and dependency outage/recovery.
- HCO installation of the dedicated naming service, certificate/RBAC and settlement/status readiness before activation; validation and reconciliation availability during replica or leader changes and rolling upgrades; HyperConverged `v1`/`v1beta1` configuration and supported KubeVirt version skew.
- GitOps direct creation, request/VM ordering and field ownership under default `AutoResolve`, assigned-name readback/drift, and dry-run limitations.
- Clone/restore storage preparation followed by provisional VM creation, naming repair and settled-name reporting; backup/disaster-recovery settlement rebuilding in each supported integration.
- Upgrade, policy disablement, webhook/service upgrade and uninstall with visible degraded status, no unbounded repair loop, and no permanent write lockout.

Benchmark discovery and settlement operations separately, including repair latency at scale. Tests must establish the actual release-specific clone/restore behavior rather than assuming all API versions apply patches to existing targets.

## Implementation History

- 2026-09-09: Initial VEP draft. Tracker: [#457](https://github.com/kubevirt/enhancements/issues/457).
- 2026-09-16: Spec-field revision documented field selection, generation, and duplicate-name behavior.
- 2026-10-14: Proposed revision changes direction to an optional display annotation with read-time fallback. Subsequent revision makes effective-name uniqueness the namespace default with explicit opt-out and makes admission/reservations and client workflows required delivery work; community agreement remains pending.
- 2026-10-14: Deployment revision proposes KubeVirt-owned admission, compares HCO and independent-operator alternatives, and separates the Alpha/Beta lifecycle gate from the customer uniqueness policy.
- 2026-10-14: Scope revision makes HCO the deployment and lifecycle-gate owner, limits default enforcement to HCO-managed installations, and removes planned `kubevirt/kubevirt` and `kubevirt/api` implementation work.
- 2026-10-14: A standalone naming-operator prototype tested actual direct creates, UID binding, rename cleanup, diskless and storage-backed Clone/Restore, and an explicit Restore override on one OpenShift Virtualization 4.22.12 cluster. Rejected direct creates leave quarantined claims, and disabling admission during a pending Restore allowed a duplicate. Safe recovery and activation remain blockers; the no-KubeVirt-code boundary is a feasibility target.
- 2026-10-15: A new OpenShift Virtualization 4.22.12 cluster tested the prototype's same-canonical direct-create retry after a later KubeVirt validation rejection. The corrected VM persisted, and its pinned claim bound to the actual UID rather than the rejected attempt's provisional UID. Twenty overlapping creates persisted one VM; controlled delayed requests also exercised a retry that persisted before its first request. After rename, the old retry-pinned name remained reserved, whereas a normal bound claim was released and reused. A delay beyond the API server's admission deadline produced another failed create. These observations improve retry availability but do not provide a safe release or completion signal, so default enforcement and the no-KubeVirt-change delivery claim remain blocked pending API machinery and KubeVirt review.
- 2026-10-15: Internal review added Kubernetes admission and cross-resource policy precedents, compared OpenShift Route's post-persistence admission contract, and identified VM pool template annotation propagation as another required lifecycle path. None resolves the direct-create completion and claim-release blocker.
- 2026-10-15: Packaging revision proposes a dedicated naming service deployed and managed as an HCO operand, without requiring a second installed operator or new repository. HCO's existing webhook binary and KubeVirt-owned admission remain alternatives; packaging does not resolve the claim-release blocker.
- 2026-10-15: Product discussion accepted a brief, visible duplicate after direct creation and automatic suffixing of a conflicting **new** VM. This revision replaces the proposed strict `Enforced` policy with default `AutoResolve`, known-conflict rejection and raced-conflict rollback for edits/clears, and UID-bound post-persistence settlement. The earlier claim-release blocker no longer applies to this revised contract; implementation evidence for convergence, pool behavior, status, and supported lifecycle flows is still required.
- 2026-10-15: Monitoring revision assigns `kubevirt_vm_info` display-name reporting to `kubevirt-observability-controller`, preserving the no-planned-KubeVirt-code scope and adding exporter migration, mutable-label compatibility, tests, and documentation to required delivery work.

## Graduation Requirements

### Alpha

- [ ] Community agrees on the annotation key, naming term, validation contract, and ownership; issue #457 tracks implementation.
- [ ] HCO lifecycle gate and default `AutoResolve` policy are separate; the Alpha gate is off until enabled, and gate-off/reenablement behavior is documented and tested in `v1` and supported `v1beta1` configurations.
- [ ] HCO deploys and health-checks the dedicated naming service, settlement/status storage, validation, RBAC, certificates, and readiness before `AutoResolve` reports active. Supported-release tests establish whether KubeVirt code changes are avoidable.
- [ ] Default namespace conflict repair and an explicit `Disabled` opt-out are implemented with UID-bound stable assignments, collision-checked suffixes for new VMs, and safe retries and recovery.
- [ ] Existing deployments detect and resolve legacy conflicts or explicitly opt out, then activate with visible status and a reliable inventory-to-watch handoff; no blanket annotation backfill occurs.
- [ ] Parallel create/change/clear tests prove brief duplicates are visible, new-VM suffixing converges, known edit/clear conflicts reject, raced conflicts roll back, and stale settlement cannot overwrite newer VM changes.
- [ ] Observability-controller emits `kubevirt_vm_info{display_name=...}` with effective-name fallback; tests and metric documentation cover changes, clears, repairs, canonical identity, and supported exporter migration without duplicate VM counting.
- [ ] Console and a supported CLI path can set/change/clear, present, and discover effective names with canonical identity, explicit lookup modes, and safe ambiguity behavior.
- [ ] GitOps, clone, import, and restore have supported naming and conflict workflows, including automatic new-VM suffixing, final assigned-name reporting, and actionable raw Clone/Restore status.
- [ ] VM pool templates, child creation, scale, and updates have a supported rule that converges without overwriting child repairs; existing pools are covered by activation and supported-release tests.
- [ ] User-guide documents fallback, provisional duplicates, automatic suffixing, edit rollback, default `AutoResolve` and opt-out, activation, permissions, lifecycle behavior, cross-mode search results, and the remaining literal-rename gap.
- [ ] Name-only edits and supported lifecycle flows are tested without naming-induced runtime effects.

### Beta

- [ ] Complete authorized inventory search and bounded suggestions are verified at representative scale; no first-match behavior remains.
- [ ] The Beta HCO lifecycle gate is on by default; upgrades from gate-off installations use conflict detection and visible activation before reporting `AutoResolve` active.
- [ ] Import, clone, restore, GitOps, settlement, and mixed-version compatibility have passing integration tests for supported releases.
- [ ] Monitoring queries, alert identity and duration, mutable-label churn, and mixed exporter versions pass compatibility and scale tests, including rollback.
- [ ] Parallel creates and edits, delete/recreate, dependency outages, repair latency, and recovery are verified at representative scale against the stated convergence objective.
- [ ] No unresolved user-facing security, compatibility, or lifecycle issue from Alpha.

### GA

- [ ] Documentation and supported client integrations converge on the same effective-name rule and canonical-target behavior.
- [ ] The HCO lifecycle gate is removed at GA while the explicit Disabled policy remains available as the customer opt-out.
- [ ] Scale, upgrade, activation, and lifecycle evidence supports default eventual uniqueness on the supported matrix.
- [ ] Deployment, RBAC, certificates, upgrade, disablement, uninstall, GitOps ownership, and support responsibilities are documented.

## Open Decisions and Companion Trackers

Issue #457 remains the VEP tracker; no implementation tracker IDs are assigned here. Owners should create or link the required console, supported CLI, GitOps, pool, import, clone/restore, observability-controller, docs, and enforcement work before claiming an initial delivery milestone.

| Decision or blocker | Suggested decision owner | Consequence |
| --- | --- | --- |
| Annotation key and community ownership | SIG Compute and client owners | No shared convention until agreed. |
| Final length, Unicode whitespace/control rules, and admission ownership | API, UX, and CLI owners | Different writers can accept different values if the server contract is incomplete. |
| External CLI plugin commands and optional future regex | CLI owners | Discovery and support boundary remain open; native `virtctl` integration is outside this proposal. |
| UI search parity and complete inventory path at scale | Console owners | A partial result could look complete. |
| Case and Unicode equivalence for effective-name uniqueness | Policy and UX owners | Different clients can disagree about conflicts. |
| Naming request or equivalent intent API, UID-bound settlement/status, GitOps ownership, and dry-run | Naming-service and GitOps owners | Direct manifests and clients must not fight an assigned suffix; status must identify the current VM and show when a name is provisional or settled. |
| Clone, new-target restore, and import requested-versus-assigned reporting | Lifecycle and integration owners | Inherited names may acquire suffixes after VM persistence; workflows must report the settled result. |
| Naming-service code location and HCO deployment contract, HCO gate name, `v1`/`v1beta1` policy, availability, and policy-removal procedure | HCO and SIG Compute owners | HCO must order status storage, validation, and service rollout so upgrades or uninstall neither block writes permanently nor silently stop repair. A separate repository is optional; standalone KubeVirt remains outside the managed policy. |
| Post-persistence convergence protocol and status freshness | Naming-service, API machinery, and UX owners | Prove deterministic ownership, stable suffixes, rollback of raced edits/clears, stale-status detection, and a measured repair-time objective. A healthy service must converge after writes quiesce; an outage must be visible. |
| Observability-controller version, metric-source migration, and consumer compatibility | SIG Monitoring, observability-controller, and HCO owners | Add `display_name` in the external collector, define supported versions and scrape ownership, and verify label churn, canonical queries, alert continuity, and rollback. No legacy KubeVirt metric backport is planned. |
| Admission availability during controller outage | HCO and API owners | Decide whether new direct creates fail closed or persist as visibly unresolved while repair is unavailable; define a degradation threshold and recovery behavior. |
| VM pool template display annotation and child update behavior | KubeVirt pool, naming-service, and HCO owners | A multi-replica template can copy one name to several child VMs, and pool updates can replace child annotations. Decide whether to reject the template key, preserve or prohibit individual child edits, or generate distinct child names, including behavior for existing pools; test whether this needs KubeVirt changes. |
| Strict pre-persistence uniqueness if later required | Product, SIG Compute, and API machinery owners | This revision explicitly accepts brief visible duplicates. A future strict guarantee would reopen the direct-create completion/claim-release problem and likely need another VEP revision. |
| Remaining literal-rename RFE needs and customer validation | Product/customer owners | Inventory naming may not satisfy literal-name policies. |
