API Conventions
===============

*This document is oriented at users who want a deeper understanding of the
Kubernetes API structure, and developers wanting to extend the Kubernetes API.
An introduction to using resources with kubectl can be found in [the object management overview](https://kubernetes.io/docs/concepts/overview/working-with-objects/object-management/).*

**Table of Contents**


  - [Types (Kinds)](#types-kinds)
    - [Resources](#resources)
    - [Objects](#objects)
      - [Metadata](#metadata)
      - [Spec and Status](#spec-and-status)
        - [Typical status properties](#typical-status-properties)
      - [References to related objects](#references-to-related-objects)
      - [Lists of named subobjects preferred over maps](#lists-of-named-subobjects-preferred-over-maps)
      - [Primitive types](#primitive-types)
      - [Constants](#constants)
      - [Unions](#unions)
    - [Lists and Simple kinds](#lists-and-simple-kinds)
  - [Differing Representations](#differing-representations)
  - [Verbs on Resources](#verbs-on-resources)
    - [PATCH operations](#patch-operations)
  - [Idempotency](#idempotency)
  - [Optional vs. Required](#optional-vs-required)
  - [Defaulting](#defaulting)
  - [Late Initialization](#late-initialization)
  - [Concurrency Control and Consistency](#concurrency-control-and-consistency)
  - [Serialization Format](#serialization-format)
  - [Units](#units)
  - [Selecting Fields](#selecting-fields)
  - [Object references](#object-references)
  - [HTTP Status codes](#http-status-codes)
      - [Success codes](#success-codes)
      - [Error codes](#error-codes)
  - [Response Status Kind](#response-status-kind)
  - [Events](#events)
  - [Naming conventions](#naming-conventions)
  - [Label, selector, and annotation conventions](#label-selector-and-annotation-conventions)
  - [WebSockets and SPDY](#websockets-and-spdy)
  - [Validation](#validation)


The conventions of the [Kubernetes API](https://kubernetes.io/docs/api/) (and related APIs in the
ecosystem) are intended to ease client development and ensure that configuration
mechanisms can be implemented that work across a diverse set of use cases
consistently.

The general style of the Kubernetes API is RESTful - clients create, update,
delete, or retrieve a description of an object via the standard HTTP verbs
(POST, PUT, DELETE, and GET) - and those APIs preferentially accept and return
JSON. Kubernetes also exposes additional endpoints for non-standard verbs and
allows alternative content types. All of the JSON accepted and returned by the
server has a schema, identified by the "kind" and "apiVersion" fields. Where
relevant HTTP header fields exist, they should mirror the content of JSON
fields, but the information should not be represented only in the HTTP header.

The following terms are defined:

* **Kind** the name of a particular object schema (e.g. the "Cat" and "Dog"
kinds would have different attributes and properties)
* **Resource** a representation of a system entity, sent or retrieved as JSON
via HTTP to the server. Resources are exposed via:
  * Collections - a list of resources of the same type, which may be queryable
  * Elements - an individual resource, addressable via a URL
* **API Group** a set of resources that are exposed together, along
with the version exposed in the "apiVersion" field as "GROUP/VERSION", e.g.
"policy.k8s.io/v1".

Each resource typically accepts and returns data of a single kind. A kind may be
accepted or returned by multiple resources that reflect specific use cases. For
instance, the kind "Pod" is exposed as a "pods" resource that allows end users
to create, update, and delete pods, while a separate "pod status" resource (that
acts on "Pod" kind) allows automated processes to update a subset of the fields
in that resource.

Resources are bound together in API groups - each group may have one or more
versions that evolve independent of other API groups, and each version within
the group has one or more resources. Group names are typically in domain name
form - the Kubernetes project reserves use of the empty group, all single
word names ("extensions", "apps"), and any group name ending in "*.k8s.io" for
its sole use. When choosing a group name, we recommend selecting a subdomain
your group or organization owns, such as "widget.mycompany.com".

Version strings should match
[DNS_LABEL](https://github.com/kubernetes/community/blob/master/contributors/design-proposals/architecture/identifiers.md)
format.



Resource collections should be all lowercase and plural, whereas kinds are
CamelCase and singular. Group names must be lower case and be valid DNS
subdomains.


## Types (Kinds)

Kinds are grouped into three categories:

1. **Objects** represent a persistent entity in the system.

   Creating an API object is a record of intent - once created, the system will
work to ensure that resource exists. All API objects have common metadata.

   An object may have multiple resources that clients can use to perform
specific actions that create, update, delete, or get.

   Examples: `Pod`, `ReplicationController`, `Service`, `Namespace`, `Node`.

2. **Lists** are collections of **resources** of one (usually) or more
(occasionally) kinds.

   The name of a list kind must end with "List". Lists have a limited set of
common metadata. All lists use the required "items" field to contain the array
of objects they return. Any kind that has the "items" field must be a list kind.

   Most objects defined in the system should have an endpoint that returns the
full set of resources, as well as zero or more endpoints that return subsets of
the full list. Some objects may be singletons (the current user, the system
defaults) and may not have lists.

   In addition, all lists that return objects with labels should support label
filtering (see [the labels documentation](https://kubernetes.io/docs/user-guide/labels/)), and most
lists should support filtering by fields (see 
[the fields documentation](https://kubernetes.io/docs/concepts/overview/working-with-objects/field-selectors/)).

   Examples: `PodList`, `ServiceList`, `NodeList`.

3. **Simple** kinds are used for specific actions on objects and for
non-persistent entities.

   Given their limited scope, they have the same set of limited common metadata
as lists.

   For instance, the "Status" kind is returned when errors occur and is not
persisted in the system.

   Many simple resources are "subresources", which are rooted at API paths of
specific resources. When resources wish to expose alternative actions or views
that are closely coupled to a single resource, they should do so using new
sub-resources. Common subresources include:

   * `/binding`: Used to bind a resource representing a user request (e.g., Pod,
PersistentVolumeClaim) to a cluster infrastructure resource (e.g., Node,
PersistentVolume).
   * `/status`: Used to write just the status portion of a resource. For
example, the `/pods` endpoint only allows updates to `metadata` and `spec`,
since those reflect end-user intent. An automated process should be able to
modify status for users to see by sending an updated Pod kind to the server to
the "/pods/&lt;name&gt;/status" endpoint - the alternate endpoint allows
different rules to be applied to the update, and access to be appropriately
restricted.
   * `/scale`: Used to read and write the count of a resource in a manner that
is independent of the specific resource schema.

   Two additional subresources, `proxy` and `portforward`, provide access to
cluster resources as described in
[accessing the cluster](https://kubernetes.io/docs/user-guide/accessing-the-cluster/).

The standard REST verbs (defined below) MUST return singular JSON objects. Some
API endpoints may deviate from the strict REST pattern and return resources that
are not singular JSON objects, such as streams of JSON objects or unstructured
text log data.

A common set of "meta" API objects are used across all API groups and are
thus considered part of the API group named `meta.k8s.io`. These types may
evolve independent of the API group that uses them and API servers may allow
them to be addressed in their generic form. Examples are `ListOptions`,
`DeleteOptions`, `List`, `Status`, `WatchEvent`, and `Scale`. For historical
reasons these types are part of each existing API group. Generic tools like
quota, garbage collection, autoscalers, and generic clients like kubectl
leverage these types to define consistent behavior across different resource
types, like the interfaces in programming languages.

The term "kind" is reserved for these "top-level" API types. The term "type"
should be used for distinguishing sub-categories within objects or subobjects.

### Resources

All JSON objects returned by an API MUST have the following fields:

* kind: a string that identifies the schema this object should have
* apiVersion: a string that identifies the version of the schema the object
should have

These fields are required for proper decoding of the object. They may be
populated by the server by default from the specified URL path, but the client
likely needs to know the values in order to construct the URL path.

### Objects

#### Metadata

Every object kind MUST have the following metadata in a nested object field
called "metadata":

* namespace: a namespace is a DNS compatible label that objects are subdivided
into. The default namespace is 'default'. See
[the namespace docs](https://kubernetes.io/docs/user-guide/namespaces/) for more.
* name: a string that uniquely identifies this object within the current
namespace (see [the identifiers docs](https://kubernetes.io/docs/user-guide/identifiers/)).
This value is used in the path when retrieving an individual object.
* uid: a unique in time and space value (typically an RFC 4122 generated
identifier, see [the identifiers docs](https://kubernetes.io/docs/user-guide/identifiers/))
used to distinguish between objects with the same name that have been deleted
and recreated

Every object SHOULD have the following metadata in a nested object field called
"metadata":

* resourceVersion: a string that identifies the internal version of this object
that can be used by clients to determine when objects have changed. This value
MUST be treated as opaque by clients and passed unmodified back to the server.
Clients should not assume that the resource version has meaning across
namespaces, different kinds of resources, or different servers. (See
[concurrency control](#concurrency-control-and-consistency), below, for more
details.)
* generation: a sequence number representing a specific generation of the
desired state. Set by the system and monotonically increasing, per-resource. May
be compared, such as for RAW and WAW consistency.
* creationTimestamp: a string representing an RFC 3339 date of the date and time
an object was created
* deletionTimestamp: a string representing an RFC 3339 date of the date and time
after which this resource will be deleted. This field is set by the server when
a graceful deletion is requested by the user, and is not directly settable by a
client. The resource will be deleted (no longer visible from resource lists, and
not reachable by name) after the time in this field except when the object has
a finalizer set. In case the finalizer is set the deletion of the object is
postponed at least until the finalizer is removed.
Once the deletionTimestamp is set, this value may not be unset or be set further
into the future, although it may be shortened or the resource may be deleted
prior to this time.
* labels: a map of string keys and values that can be used to organize and
categorize objects (see [the labels docs](https://kubernetes.io/docs/user-guide/labels/))
* annotations: a map of string keys and values that can be used by external
tooling to store and retrieve arbitrary metadata about this object (see
[the annotations docs](https://kubernetes.io/docs/user-guide/annotations/))

Labels are intended for organizational purposes by end users (select the pods
that match this label query). Annotations enable third-party automation and
tooling to decorate objects with additional metadata for their own use.

#### Spec and Status

By convention, the Kubernetes API makes a distinction between the specification
of the desired state of an object (a nested object field called "spec") and the
status of the object at the current time (a nested object field called
"status"). The specification is a complete description of the desired state,
including configuration settings provided by the user,
[default values](#defaulting) expanded by the system, and properties initialized
or otherwise changed after creation by other ecosystem components (e.g.,
schedulers, auto-scalers), and is persisted in stable storage with the API
object. If the specification is deleted, the object will be purged from the
system. The status summarizes the current state of the object in the system, and
is usually persisted with the object by automated processes but may be
generated on the fly. At some cost and perhaps some temporary degradation in
behavior, the status could be reconstructed by observation if it were lost.

When a new version of an object is POSTed or PUT, the "spec" is updated and
available immediately. Over time the system will work to bring the "status" into
line with the "spec". The system will drive toward the most recent "spec"
regardless of previous versions of that stanza. In other words, if a value is
changed from 2 to 5 in one PUT and then back down to 3 in another PUT the system
is not required to 'touch base' at 5 before changing the "status" to 3. In other
words, the system's behavior is *level-based* rather than *edge-based*. This
enables robust behavior in the presence of missed intermediate state changes.

The Kubernetes API also serves as the foundation for the declarative
configuration schema for the system. In order to facilitate level-based
operation and expression of declarative configuration, fields in the
specification should have declarative rather than imperative names and
semantics -- they represent the desired state, not actions intended to yield the
desired state.

The PUT and POST verbs on objects MUST ignore the "status" values, to avoid
accidentally overwriting the status in read-modify-write scenarios. A `/status`
subresource MUST be provided to enable system components to update statuses of
resources they manage.

Otherwise, PUT expects the whole object to be specified. Therefore, if a field
is omitted it is assumed that the client wants to clear that field's value. The
PUT verb does not accept partial updates. Modification of just part of an object
may be achieved by GETting the resource, modifying part of the spec, labels, or
annotations, and then PUTting it back. See
[concurrency control](#concurrency-control-and-consistency), below, regarding
read-modify-write consistency when using this pattern. Some objects may expose
alternative resource representations that allow mutation of the status, or
performing custom actions on the object.

All objects that represent a physical resource whose state may vary from the
user's desired intent SHOULD have a "spec" and a "status". Objects whose state
cannot vary from the user's desired intent MAY have only "spec", and MAY rename
"spec" to a more appropriate name.

Objects that contain both spec and status should not contain additional
top-level fields other than the standard metadata fields.

Some objects which are not persisted in the system - such as `SubjectAccessReview`
and other webhook style calls - may choose to add spec and status to encapsulate
a "call and response" pattern. The spec is the request (often a request for
information) and the status is the response. For these RPC like objects the only
operation may be POST, but having a consistent schema between submission and
response reduces the complexity of these clients.


##### Typical status properties

**Conditions** provide a standard mechanism for higher-level status reporting
from a controller. They are an extension mechanism which allows tools and other
controllers to collect summary information about resources without needing to
understand resource-specific status details. Conditions should complement more
detailed information about the observed status of an object written by a
controller, rather than replace it. For example, the "Available" condition of a
Deployment can be determined by examining `readyReplicas`, `replicas`, and
other properties of the Deployment. However, the "Available" condition allows
other components to avoid duplicating the availability logic in the Deployment
controller.

Objects may report multiple conditions, and new types of conditions may be
added in the future or by 3rd party controllers. Therefore, conditions are
represented using a list/slice of objects, where each condition has a similar
structure. This collection should be treated as a map with a key of `type`.

Conditions are most useful when they follow some consistent conventions:

* Conditions should be added to explicitly convey properties that users and
  components care about rather than requiring those properties to be inferred
  from other observations.  Once defined, the meaning of a Condition can not be
  changed arbitrarily - it becomes part of the API, and has the same backwards-
  and forwards-compatibility concerns of any other part of the API.

* Controllers should apply their conditions to a resource the first time they
  visit the resource, even if the `status` is Unknown. This allows other
  components in the system to know that the condition exists and the controller
  is making progress on reconciling that resource.

   * Not all controllers will observe the previous advice about reporting
     "Unknown" or "False" values. For known conditions, the absence of a
     condition status should be interpreted the same as `Unknown`, and
     typically indicates that reconciliation has not yet finished (or that the
     resource state may not yet be observable).

* For some conditions, `True` represents normal operation, and for some
  conditions, `False` represents normal operation. ("Normal-true" conditions
  are sometimes said to have "positive polarity", and "normal-false" conditions
  are said to have "negative polarity".) Without further knowledge of the
  conditions, it is not possible to compute a generic summary of the conditions
  on a resource.

* Condition type names should make sense for humans; neither positive nor
  negative polarity can be recommended as a general rule. A negative condition
  like "MemoryExhausted" may be easier for humans to understand than
  "SufficientMemory". Conversely, "Ready" or "Succeeded" may be easier to
  understand than "Failed", because "Failed=Unknown" or "Failed=False" may
  cause double-negative confusion.

* Condition type names should describe the current observed state of the
  resource, rather than describing the current state transitions. This
  typically means that the name should be an adjective ("Ready", "OutOfDisk")
  or a past-tense verb ("Succeeded", "Failed") rather than a present-tense verb
  ("Deploying"). Intermediate states may be indicated by setting the status of
  the condition to `Unknown`.

  * For state transitions which take a long period of time (rule of thumb: > 1
    minute), it is reasonable to treat the transition itself as an observed
    state. In these cases, the Condition (such as "Resizing") itself should not
    be transient, and should instead be signalled using the
    `True`/`False`/`Unknown` pattern. This allows other observers to determine
    the last update from the controller, whether successful or failed. In cases
    where the state transition is unable to complete and continued
    reconciliation is not feasible, the Reason and Message should be used to
    indicate that the transition failed.

* When designing Conditions for a resource, it's helpful to have a common
  top-level condition which summarizes more detailed conditions. Simple
  consumers may simply query the top-level condition. Although they are not a
  consistent standard, the `Ready` and `Succeeded` condition types may be used
  by API designers for long-running and bounded-execution objects, respectively.

The `FooCondition` type for some resource type `Foo` may include a subset of the
following fields, but must contain at least `type` and `status` fields:

```go
  Type               FooConditionType   `json:"type" description:"type of Foo condition"`
  Status             ConditionStatus    `json:"status" description:"status of the condition, one of True, False, Unknown"`

  // +optional
  Reason             *string            `json:"reason,omitempty" description:"one-word CamelCase reason for the condition's last transition"`
  // +optional
  Message            *string            `json:"message,omitempty" description:"human-readable message indicating details about last transition"`

  // +optional
  LastHeartbeatTime  *unversioned.Time  `json:"lastHeartbeatTime,omitempty" description:"last time we got an update on a given condition"`
  // +optional
  LastTransitionTime *unversioned.Time  `json:"lastTransitionTime,omitempty" description:"last time the condition transit from one status to another"`
```

Additional fields may be added in the future.

Do not use fields that you don't need - simpler is better.

Use of the `Reason` field is encouraged.

Use the `LastHeartbeatTime` with great caution - frequent changes to this field
can cause a large fan-out effect for some resources.

Condition types should be named in PascalCase. Short condition names are
preferred (e.g. "Ready" over "MyResourceReady").

Condition status values may be `True`, `False`, or `Unknown`. The absence of a
condition should be interpreted the same as `Unknown`.  How controllers handle
`Unknown` depends on the Condition in question.

The thinking around conditions has evolved over time, so there are several
non-normative examples in wide use.

In general, condition values may change back and forth, but some condition
transitions may be monotonic, depending on the resource and condition type.
However, conditions are observations and not, themselves, state machines, nor do
we define comprehensive state machines for objects, nor behaviors associated
with state transitions. The system is level-based rather than edge-triggered,
and should assume an Open World.

An example of an oscillating condition type is `Ready`, which indicates the
object was believed to be fully operational at the time it was last probed. A
possible monotonic condition could be `Succeeded`. A `True` status for
`Succeeded` would imply completion and that the resource was no longer
active. An object that was still active would generally have a `Succeeded`
condition with status `Unknown`.

Some resources in the v1 API contain fields called **`phase`**, and associated
`message`, `reason`, and other status fields. The pattern of using `phase` is
deprecated. Newer API types should use conditions instead. Phase was
essentially a state-machine enumeration field, that contradicted [system-design
principles](../../design-proposals/architecture/principles.md#control-logic) and
hampered evolution, since [adding new enum values breaks backward
compatibility](api_changes.md). Rather than encouraging clients to infer
implicit properties from phases, we prefer to explicitly expose the individual
conditions that clients need to monitor. Conditions also have the benefit that
it is possible to create some conditions with uniform meaning across all
resource types, while still exposing others that are unique to specific
resource types.  See [#7856](http://issues.k8s.io/7856) for more details and
discussion.

In condition types, and everywhere else they appear in the API, **`Reason`** is
intended to be a one-word, CamelCase representation of the category of cause of
the current status, and **`Message`** is intended to be a human-readable phrase
or sentence, which may contain specific details of the individual occurrence.
`Reason` is intended to be used in concise output, such as one-line
`kubectl get` output, and in summarizing occurrences of causes, whereas
`Message` is intended to be presented to users in detailed status explanations,
such as `kubectl describe` output.

Historical information status (e.g., last transition time, failure counts) is
only provided with reasonable effort, and is not guaranteed to not be lost.

Status information that may be large (especially proportional in size to
collections of other resources, such as lists of references to other objects --
see below) and/or rapidly changing, such as
[resource usage](../../design-proposals/scheduling/resources.md#usage-data), should be put into separate
objects, with possibly a reference from the original object. This helps to
ensure that GETs and watch remain reasonably efficient for the majority of
clients, which may not need that data.

Some resources report the `observedGeneration`, which is the `generation` most
recently observed by the component responsible for acting upon changes to the
desired state of the resource. This can be used, for instance, to ensure that
the reported status reflects the most recent desired status.

#### References to related objects

References to loosely coupled sets of objects, such as
[pods](https://kubernetes.io/docs/user-guide/pods/) overseen by a
[replication controller](https://kubernetes.io/docs/user-guide/replication-controller/), are usually
best referred to using a [label selector](https://kubernetes.io/docs/user-guide/labels/). In order to
ensure that GETs of individual objects remain bounded in time and space, these
sets may be queried via separate API queries, but will not be expanded in the
referring object's status.

For references to specific objects, see [Object references](#object-references).

References in the status of the referee to the referrer may be permitted, when
the references are one-to-one and do not need to be frequently updated,
particularly in an edge-based manner.

#### Lists of named subobjects preferred over maps

Discussed in [#2004](http://issue.k8s.io/2004) and elsewhere. There are
no maps of subobjects in any API objects. Instead, the convention is to
use a list of subobjects containing name fields. These conventions, and
how one can change the semantics of lists, structs and maps are
described in more details in the Kubernetes
[documentation](https://kubernetes.io/docs/reference/using-api/server-side-apply/#merge-strategy).

For example:

```yaml
ports:
  - name: www
    containerPort: 80
```

vs.

```yaml
ports:
  www:
    containerPort: 80
```

This rule maintains the invariant that all JSON/YAML keys are fields in API
objects. The only exceptions are pure maps in the API (currently, labels,
selectors, annotations, data), as opposed to sets of subobjects.

#### Primitive types

* Avoid floating-point values as much as possible, and never use them in spec.
  Floating-point values cannot be reliably round-tripped (encoded and
  re-decoded) without changing, and have varying precision and representations
  across languages and architectures.
* All numbers (e.g., uint32, int64) are converted to float64 by Javascript and
  some other languages, so any field which is expected to exceed that either in
  magnitude or in precision (specifically integer values > 53 bits) should be
  serialized and accepted as strings.
* Do not use unsigned integers, due to inconsistent support across languages and
  libraries. Just validate that the integer is non-negative if that's the case.
* Do not use enums. Use aliases for string instead (e.g., `NodeConditionType`).
* Look at similar fields in the API (e.g., ports, durations) and follow the
  conventions of existing fields.
* All public integer fields MUST use the Go `(u)int32` or Go `(u)int64` types,
  not `(u)int` (which is ambiguous depending on target platform). Internal
  types may use `(u)int`.
* Think twice about `bool` fields. Many ideas start as boolean but eventually
  trend towards a small set of mutually exclusive options.  Plan for future
  expansions by describing the policy options explicitly as a string type
  alias (e.g. `TerminationMessagePolicy`).

#### Constants

Some fields will have a list of allowed values (enumerations). These values will
be strings, and they will be in CamelCase, with an initial uppercase letter.
Examples: `ClusterFirst`, `Pending`, `ClientIP`. When an acronym or initialism
each letter in the acronym should be uppercase, such as with `ClientIP` or
`TCPDelay`. When a proper name or the name of a command-line executable is used
as a constant the proper name should be represented in consistent casing -
examples: `systemd`, `iptables`, `IPVS`, `cgroupfs`, `Docker` (as a generic
concept), `docker` (as the command-line executable). If a proper name is used
which has mixed capitalization like `eBPF` that should be preserved in a longer
constant such as `eBPFDelegation`.

All API within Kubernetes must leverage constants in this style, including
flags and configuration files. Where inconsistent constants were previously used,
new flags should be CamelCase only, and over time old flags should be updated to
accept a CamelCase value alongside the inconsistent constant. Example: the
Kubelet accepts a `--topology-manager-policy` flag that has values `none`,
`best-effort`, `restricted`, and `single-numa-node`. This flag should accept
`None`, `BestEffort`, `Restricted`, and `SingleNUMANode` going forward. If new
values are added to the flag, both forms should be supported.

#### Unions

Sometimes, at most one of a set of fields can be set.  For example, the
[volumes] field of a PodSpec has 17 different volume type-specific fields, such
as `nfs` and `iscsi`.  All fields in the set should be
[Optional](#optional-vs-required).

Sometimes, when a new type is created, the api designer may anticipate that a
union will be needed in the future, even if only one field is allowed initially.
In this case, be sure to make the field [Optional](#optional-vs-required)
In the validation, you may still return an error if the sole field is unset. Do
not set a default value for that field.

### Lists and Simple kinds

Every list or simple kind SHOULD have the following metadata in a nested object
field called "metadata":

* resourceVersion: a string that identifies the common version of the objects
returned by in a list. This value MUST be treated as opaque by clients and
passed unmodified back to the server. A resource version is only valid within a
single namespace on a single kind of resource.

Every simple kind returned by the server, and any simple kind sent to the server
that must support idempotency or optimistic concurrency should return this
value. Since simple resources are often used as input alternate actions that
modify objects, the resource version of the simple resource should correspond to
the resource version of the object.


## Differing Representations

An API may represent a single entity in different ways for different clients, or
transform an object after certain transitions in the system occur. In these
cases, one request object may have two representations available as different
resources, or different kinds.

An example is a Service, which represents the intent of the user to group a set
of pods with common behavior on common ports. When Kubernetes detects a pod
matches the service selector, the IP address and port of the pod are added to an
Endpoints resource for that Service. The Endpoints resource exists only if the
Service exists, but exposes only the IPs and ports of the selected pods. The
full service is represented by two distinct resources - under the original
Service resource the user created, as well as in the Endpoints resource.

As another example, a "pod status" resource may accept a PUT with the "pod"
kind, with different rules about what fields may be changed.

Future versions of Kubernetes may allow alternative encodings of objects beyond
JSON.


## Verbs on Resources

API resources should use the traditional REST pattern:

* GET /&lt;resourceNamePlural&gt; - Retrieve a list of type
&lt;resourceName&gt;, e.g. GET /pods returns a list of Pods.
* POST /&lt;resourceNamePlural&gt; - Create a new resource from the JSON object
provided by the client.
* GET /&lt;resourceNamePlural&gt;/&lt;name&gt; - Retrieves a single resource
with the given name, e.g. GET /pods/first returns a Pod named 'first'. Should be
constant time, and the resource should be bounded in size.
* DELETE /&lt;resourceNamePlural&gt;/&lt;name&gt;  - Delete the single resource
with the given name. DeleteOptions may specify gracePeriodSeconds, the optional
duration in seconds before the object should be deleted. Individual kinds may
declare fields which provide a default grace period, and different kinds may
have differing kind-wide default grace periods. A user provided grace period
overrides a default grace period, including the zero grace period ("now").
* DELETE /&lt;resourceNamePlural&gt; - Deletes a list of type
&lt;resourceName&gt;, e.g. DELETE /pods a list of Pods.
* PUT /&lt;resourceNamePlural&gt;/&lt;name&gt; - Update or create the resource
with the given name with the JSON object provided by the client.
* PATCH /&lt;resourceNamePlural&gt;/&lt;name&gt; - Selectively modify the
specified fields of the resource. See more information [below](#patch-operations).
* GET /&lt;resourceNamePlural&gt;&quest;watch=true - Receive a stream of JSON
objects corresponding to changes made to any resource of the given kind over
time.

### PATCH operations

The API supports three different PATCH operations, determined by their
corresponding Content-Type header:

* JSON Patch, `Content-Type: application/json-patch+json`
  * As defined in [RFC6902](https://tools.ietf.org/html/rfc6902), a JSON Patch is
a sequence of operations that are executed on the resource, e.g. `{"op": "add",
"path": "/a/b/c", "value": [ "foo", "bar" ]}`. For more details on how to use
JSON Patch, see the RFC.
* Merge Patch, `Content-Type: application/merge-patch+json`
  * As defined in [RFC7386](https://tools.ietf.org/html/rfc7386), a Merge Patch
is essentially a partial representation of the resource. The submitted JSON is
"merged" with the current resource to create a new one, then the new one is
saved. For more details on how to use Merge Patch, see the RFC.
* Strategic Merge Patch, `Content-Type: application/strategic-merge-patch+json`
  * Strategic Merge Patch is a custom implementation of Merge Patch. For a
detailed explanation of how it works and why it needed to be introduced, see
[here](/contributors/devel/sig-api-machinery/strategic-merge-patch.md).

## Idempotency

All compatible Kubernetes APIs MUST support "name idempotency" and respond with
an HTTP status code 409 when a request is made to POST an object that has the
same name as an existing object in the system. See
[the identifiers docs](https://kubernetes.io/docs/user-guide/identifiers/) for details.

Names generated by the system may be requested using `metadata.generateName`.
GenerateName indicates that the name should be made unique by the server prior
to persisting it. A non-empty value for the field indicates the name will be
made unique (and the name returned to the client will be different than the name
passed). The value of this field will be combined with a unique suffix on the
server if the Name field has not been provided. The provided value must be valid
within the rules for Name, and may be truncated by the length of the suffix
required to make the value unique on the server. If this field is specified, and
Name is not present, the server will NOT return a 409 if the generated name
exists - instead, it will either return 201 Created or 504 with Reason
`ServerTimeout` indicating a unique name could not be found in the time
allotted, and the client should retry (optionally after the time indicated in
the Retry-After header).

## Optional vs. Required

Fields must be either optional or required.

Optional fields have the following properties:

- They have the `+optional` comment tag in Go.
- They are a pointer type in the Go definition (e.g. `AwesomeFlag *SomeFlag`) or
have a built-in `nil` value (e.g. maps and slices).
- The API server should allow POSTing and PUTing a resource with this field
unset.

In most cases, optional fields should also have the `omitempty` struct tag (the 
`omitempty` option specifies that the field should be omitted from the json
encoding if the field has an empty value). However, If you want to have 
different logic for an optional field which is not provided vs. provided with 
empty values, do not use `omitempty` (e.g. https://github.com/kubernetes/kubernetes/issues/34641).

Note that for backward compatibility, any field that has the `omitempty` struct
tag will be considered to be optional, but this may change in the future and
having the `+optional` comment tag is highly recommended.

Required fields have the opposite properties, namely:

- They do not have an `+optional` comment tag.
- They do not have an `omitempty` struct tag.
- They are not a pointer type in the Go definition (e.g. `AnotherFlag SomeFlag`).
- The API server should not allow POSTing or PUTing a resource with this field
unset.

Using the `+optional` or the `omitempty` tag causes OpenAPI documentation to 
reflect that the field is optional.

Using a pointer allows distinguishing unset from the zero value for that type.
There are some cases where, in principle, a pointer is not needed for an
optional field since the zero value is forbidden, and thus implies unset. There
are examples of this in the codebase. However:

- it can be difficult for implementors to anticipate all cases where an empty
value might need to be distinguished from a zero value
- structs are not omitted from encoder output even where omitempty is specified,
which is messy;
- having a pointer consistently imply optional is clearer for users of the Go
language client, and any other clients that use corresponding types

Therefore, we ask that pointers always be used with optional fields that do not
have a built-in `nil` value.


## Defaulting

Default resource values are API version-specific, and they are applied during
the conversion from API-versioned declarative configuration to internal objects
representing the desired state (`Spec`) of the resource. Subsequent GETs of the
resource will include the default values explicitly.

Incorporating the default values into the `Spec` ensures that `Spec` depicts the
full desired state so that it is easier for the system to determine how to
achieve the state, and for the user to know what to anticipate.

API version-specific default values are set by the API server.

## Late Initialization

Late initialization is when resource fields are set by a system controller
after an object is created/updated.

For example, the scheduler sets the `pod.spec.nodeName` field after the pod is
created.

Late-initializers should only make the following types of modifications:
 - Setting previously unset fields
 - Adding keys to maps
 - Adding values to arrays which have mergeable semantics
(`patchStrategy:"merge"` attribute in the type definition).

These conventions:
 1. allow a user (with sufficient privilege) to override any system-default
 behaviors by setting the fields that would otherwise have been defaulted.
 1. enables updates from users to be merged with changes made during late
initialization, using strategic merge patch, as opposed to clobbering the
change.
 1. allow the component which does the late-initialization to use strategic
merge patch, which facilitates composition and concurrency of such components.

Although the apiserver Admission Control stage acts prior to object creation,
Admission Control plugins should follow the Late Initialization conventions
too, to allow their implementation to be later moved to a 'controller', or to
client libraries.

## Concurrency Control and Consistency

Kubernetes leverages the concept of *resource versions* to achieve optimistic
concurrency. All Kubernetes resources have a "resourceVersion" field as part of
their metadata. This resourceVersion is a string that identifies the internal
version of an object that can be used by clients to determine when objects have
changed. When a record is about to be updated, it's version is checked against a
pre-saved value, and if it doesn't match, the update fails with a StatusConflict
(HTTP status code 409).

The resourceVersion is changed by the server every time an object is modified.
If resourceVersion is included with the PUT operation the system will verify
that there have not been other successful mutations to the resource during a
read/modify/write cycle, by verifying that the current value of resourceVersion
matches the specified value.

The resourceVersion is currently backed by [etcd's
modifiedIndex](https://coreos.com/etcd/docs/latest/v2/api.html).
However, it's important to note that the application should *not* rely on the
implementation details of the versioning system maintained by Kubernetes. We may
change the implementation of resourceVersion in the future, such as to change it
to a timestamp or per-object counter.

The only way for a client to know the expected value of resourceVersion is to
have received it from the server in response to a prior operation, typically a
GET. This value MUST be treated as opaque by clients and passed unmodified back
to the server. Clients should not assume that the resource version has meaning
across namespaces, different kinds of resources, or different servers.
Currently, the value of resourceVersion is set to match etcd's sequencer. You
could think of it as a logical clock the API server can use to order requests.
However, we expect the implementation of resourceVersion to change in the
future, such as in the case we shard the state by kind and/or namespace, or port
to another storage system.

In the case of a conflict, the correct client action at this point is to GET the
resource again, apply the changes afresh, and try submitting again. This
mechanism can be used to prevent races like the following:

```
Client #1                                  Client #2
GET Foo                                    GET Foo
Set Foo.Bar = "one"                        Set Foo.Baz = "two"
PUT Foo                                    PUT Foo
```

When these sequences occur in parallel, either the change to Foo.Bar or the
change to Foo.Baz can be lost.

On the other hand, when specifying the resourceVersion, one of the PUTs will
fail, since whichever write succeeds changes the resourceVersion for Foo.

resourceVersion may be used as a precondition for other operations (e.g., GET,
DELETE) in the future, such as for read-after-write consistency in the presence
of caching.

"Watch" operations specify resourceVersion using a query parameter. It is used
to specify the point at which to begin watching the specified resources. This
may be used to ensure that no mutations are missed between a GET of a resource
(or list of resources) and a subsequent Watch, even if the current version of
the resource is more recent. This is currently the main reason that list
operations (GET on a collection) return resourceVersion.


## Serialization Format

APIs may return alternative representations of any resource in response to an
Accept header or under alternative endpoints, but the default serialization for
input and output of API responses MUST be JSON.

A protobuf encoding is also accepted for built-in resources. As proto is not
self-describing, there is an envelope wrapper which describes the type of
the contents.

All dates should be serialized as RFC3339 strings.

## Units

Units must either be explicit in the field name (e.g., `timeoutSeconds`), or
must be specified as part of the value (e.g., `resource.Quantity`). Which
approach is preferred is TBD, though currently we use the `fooSeconds`
convention for durations.

Duration fields must be represented as integer fields with units being
part of the field name (e.g. `leaseDurationSeconds`). We don't use Duration
in the API since that would require clients to implement go-compatible parsing.

## Selecting Fields

Some APIs may need to identify which field in a JSON object is invalid, or to
reference a value to extract from a separate resource. The current
recommendation is to use standard JavaScript syntax for accessing that field,
assuming the JSON object was transformed into a JavaScript object, without the
leading dot, such as `metadata.name`.

Examples:

* Find the field "current" in the object "state" in the second item in the array
"fields": `fields[1].state.current`

## Object references

Object references on a namespaced type should usually refer only to objects in
the same namespace.  Because namespaces are a security boundary, cross namespace
references can have unexpected impacts, including:
 1. leaking information about one namespace into another namespace. It's natural to place status messages or even bits of
    content about the referenced object in the original. This is a problem across namespaces.
 2. potential invasions into other namespaces. Often references give access to a piece of referred information, so being
    able to express "give me that one over there" is dangerous across namespaces without additional work for permission checks
    or opt-in's from both involved namespaces.
 3. referential integrity problems that one party cannot solve. Referencing namespace/B from namespace/A doesn't imply the
    power to control the other namespace. This means that you can refer to a thing you cannot create or update.
 4. unclear semantics on deletion. If a namespaced resource  is referenced by other namespaces, should a delete of the 
    referenced resource result in removal or should the referenced resource be force to remain.
 5. unclear semantics on creation. If a referenced resource is created after its reference, there is no way to know if it
    is the one that is expected or if it is a different one created with the same name.

Built-in types and ownerReferences do not support cross namespaces references.
If a non-built-in types chooses to have cross-namespace references the semantics of the edge cases above should be 
clearly described and the permissions issues should be resolved.
This could be done with a double opt-in (an opt-in from both the referrer and the refer-ee) or with secondary permissions
checks performed in admission. 

### Naming of the reference field

The name of the reference field should be of the format "{field}Ref", with "Ref" always included in the suffix.

The "{field}" component should be named to indicate the purpose of the reference. For example, "targetRef" in an
endpoint indicates that the object reference specifies the target.

It is okay to have the "{field}" component indicate the resource type. For example, "secretRef" when referencing
a secret. However, this comes with the risk of the field being a misnomer in the case that the field is expanded to
reference more than one type.

In the case of a list of object references, the field should be of the format "{field}Refs", with the same guidance
as the singular case above.

### Referencing resources with multiple versions

Most resources will have multiple versions. For example, core resources
will undergo version changes as it transitions from alpha to GA.

Controllers should assume that a version of a resource may change, and include appropriate error handling.

### Handling of resources that do not exist

There are multiple scenarios where a desired resource may not exist. Examples include:

- the desired version of the resource does not exist.
- race condition in the bootstrapping of a cluster resulting a resource not yet added.
- user error.

Controllers should be authored with the assumption that the referenced resource may not exist, and include
error handling to make the issue clear to the user.

### Validation of fields

Many of the values used in an object reference are used as part of the API path. For example,
the object name is used in the path to identify the object. Unsanitized, these values can be used to
attempt to retrieve other resources, such as by using values with semantic meanings such as  `..` or `/`.

Have the controller validate fields before using them as path segments in an API request, and emit an event to
tell the user that the validation has failed.

See [Object Names and IDs](https://kubernetes.io/docs/concepts/overview/working-with-objects/names/)
for more information on legal object names.

### Do not modify the referred object

To minimize potential privilege escalation vectors, do not modify the object that is being referred to,
or limit modification to objects in the same namespace and constrain the type of modification allowed
(for example, the HorizontalPodAutoscaler controller only writes to the `/scale` subresource).

### Minimize copying or printing values to the referrer object

As the permissions of the controller can differ from the permissions of the author of the object
the controller is managing, it is possible that the author of the object may not have permissions to
view the referred object. As a result, the copying of any values about the referred object to the
referrer object can be considered permissions escalations, enabling a user to read values that they
would not have access to previously.

The same scenario applies to writing information about the referred object to events.

In general, do not write or print information retrieved from the referred object to the spec, other objects, or logs.

When it is necessary, consider whether these values would be ones that the
author of the referrer object would have access to via other means (e.g. already required to
correctly populate the object reference).

### Object References Examples

The following sections illustrate recommended schemas for various object references scenarios.

The schemas outlined below are designed to enable purely additive fields as the types of referencable
objects expand, and therefore are backwards compatible.

For example, it is possible to go from a single resource type to multiple resource types without
a breaking change in the schema.

#### Single resource reference

A single kind object reference is straightforward in that the controller can hard-code most qualifiers needed to identify the object. As such as the only value needed to be provided is the name (and namespace, although cross-namespace references are discouraged):

```yaml
# for a single resource, the suffix should be Ref, with the field name
# providing an indication as to the resource type referenced.
secretRef: 
    name: foo
    # namespace would generally not be needed and is discouraged, 
    # as explained above.
    namespace: foo-namespace
```

This schema should only be used when the intention is to always have the reference only be to a single resource.
If extending to multiple resource types is possible, use the [multiple resource reference](#multiple-resource-reference).

##### Controller behavior

The operator is expected to know the version, group, and resource name of the object it needs to retrieve the value from, and can use the discovery client or construct the API path directly.

#### Multiple resource reference

Multi-kind object references are used when there is a bounded set of valid resource types that a reference can point to.

As with a single-kind object reference, the operator can supply missing fields, provided that the fields that are present are sufficient to uniquely identify the object resource type among the set of supported types.

```yaml
# guidance for the field name is the same as a single resource.
fooRef:
    group: sns.services.k8s.aws
    resource: topics
    name: foo
    namespace: foo-namespace
```

Although not always necessary to help a controller identify a resource type, “group” is included to avoid ambiguity when the resource exists in multiple groups. It also provides clarity to end users and enables copy-pasting of a reference without the referenced type changing due to a different controller handling the reference.

##### Kind vs. Resource

A common point of confusion in object references is whether to construct
references with a "kind" or "resource" field. Historically most object
references in Kubernetes have used "kind". This is not as precise as "resource".
Although each combination of "group" and "resource" must be unique within
Kubernetes, the same is not always true for "group" and "kind". It is possible
for multiple resources to make use of the same "kind".

Typically all objects in Kubernetes have a canonical primary resource - such as
“pods” representing the way to create and delete resources of the “Pod” schema.
While it is possible a resource schema cannot be directly created, such as a
“Scale” object which is only used within the “scale” subresource of a number of
workloads, most object references address the primary resource via its schema.
In the context of object references, "kind" refers to the schema, not the
resource.

If implementations of an object reference will always have a clear way to map
kinds to resources, it is acceptable to use "kind" in the object reference. In
general, this requires implementations to have a predefined mapping between
kinds and resources (this is the case for built-in references which use "kind").
Relying on dynamic kind to resource mapping is not safe. Even if a "kind" only
dynamically maps to a single resource initially, it's possible for another
resource to be mounted that refers to the same "kind", potentially breaking any
dynamic resource mapping.

If an object reference may be used to reference resources of arbitrary types and
the mapping between kind and resource could be ambiguous, "resource" should be
used in the object reference.

The Ingress API provides a good example of where "kind" is acceptable for an
object reference. The API supports a backend reference as an extension point.
Implementations can use this to support forwarding traffic to custom targets
such as a storage bucket. Importantly, the supported target types are clearly
defined by each implementation of the API and there is no ambiguity for which
resource a kind maps to. This is because each Ingress implementation has a
hard-coded mapping of kind to resource.

The object reference above would look like this if it were using "kind" instead
of "resource":

```yaml
fooRef:
    group: sns.services.k8s.aws
    kind: Topic
    name: foo
    namespace: foo-namespace
```

##### Controller behavior

The operator can store a map of (group,resource) to the version of that resource it desires. From there, it can construct the full path to the resource, and retrieve the object.

It is also possible to have the controller choose a version that it finds via the discovery client. However, as schemas can vary across different versions
of a resource, the controller must also handle these differences.

#### Generic object reference

A generic object reference is used when the desire is to provide a pointer to some object to simplify discovery for the user. For example, this could be used to reference a target object for a `core.v1.Event` that occurred.

With a generic object reference, it is not possible to extract any information about the referenced object aside from what is standard (e.g. ObjectMeta). Since any standard fields exist in any version of a resource, it is possible to not include version in this case:

```yaml
fooObjectRef:
    group: operator.openshift.io
    resource: openshiftapiservers
    name: cluster
    # namespace is unset if the resource is cluster-scoped, or lives in the 
    # same namespace as the referrer.
```

##### Controller behavior

The operator would be expected to find the resource via the discovery client (as the version is not supplied). As any retrievable field would be common to all objects, any version of the resource should do.

#### Field reference

A field reference is used when the desire is to extract a value from a specific field in a referenced object.

Field references differ from other reference types, as the operator has no knowledge of the object prior to the reference. Since the schema of an object can differ for different versions of a resource, this means that a “version” is required for this type of reference.

```yaml
fooFieldRef:
   version: v1 # version of the resource
   # group is elided in the ConfigMap example, since it has a blank group in the OpenAPI spec.
   resource: configmaps
   fieldPath: data.foo
```

The fieldPath should point to a single value, and use [the recommended field selector notation](#selecting-fields) to denote the field path.

##### Controller behavior

In this scenario, the user will supply all of the required path elements: group, version, resource, name, and possibly namespace.
As such, the controller can construct the API prefix and query it without the use of the discovery client:

```
/apis/{group}/{version}/{resource}/
```

## HTTP Status codes

The server will respond with HTTP status codes that match the HTTP spec. See the
section below for a breakdown of the types of status codes the server will send.

The following HTTP status codes may be returned by the API.

#### Success codes

* `200 StatusOK`
  * Indicates that the request completed successfully.
* `201 StatusCreated`
  * Indicates that the request to create kind completed successfully.
* `204 StatusNoContent`
  * Indicates that the request completed successfully, and the response contains
no body.
  * Returned in response to HTTP OPTIONS requests.

#### Error codes

* `307 StatusTemporaryRedirect`
  * Indicates that the address for the requested resource has changed.
  * Suggested client recovery behavior:
    * Follow the redirect.


* `400 StatusBadRequest`
  * Indicates the requested is invalid.
  * Suggested client recovery behavior:
    * Do not retry. Fix the request.


* `401 StatusUnauthorized`
  * Indicates that the server can be reached and understood the request, but
refuses to take any further action, because the client must provide
authorization. If the client has provided authorization, the server is
indicating the provided authorization is unsuitable or invalid.
  * Suggested client recovery behavior:
    * If the user has not supplied authorization information, prompt them for
the appropriate credentials. If the user has supplied authorization information,
inform them their credentials were rejected and optionally prompt them again.


* `403 StatusForbidden`
  * Indicates that the server can be reached and understood the request, but
refuses to take any further action, because it is configured to deny access for
some reason to the requested resource by the client.
  * Suggested client recovery behavior:
    * Do not retry. Fix the request.


* `404 StatusNotFound`
  * Indicates that the requested resource does not exist.
  * Suggested client recovery behavior:
    * Do not retry. Fix the request.


* `405 StatusMethodNotAllowed`
  * Indicates that the action the client attempted to perform on the resource
was not supported by the code.
  * Suggested client recovery behavior:
    * Do not retry. Fix the request.


* `409 StatusConflict`
  * Indicates that either the resource the client attempted to create already
exists or the requested update operation cannot be completed due to a conflict.
  * Suggested client recovery behavior:
    * If creating a new resource:
      * Either change the identifier and try again, or GET and compare the
fields in the pre-existing object and issue a PUT/update to modify the existing
object.
    * If updating an existing resource:
      * See `Conflict` from the `status` response section below on how to
retrieve more information about the nature of the conflict.
      * GET and compare the fields in the pre-existing object, merge changes (if
still valid according to preconditions), and retry with the updated request
(including `ResourceVersion`).


* `410 StatusGone`
  * Indicates that the item is no longer available at the server and no
forwarding address is known.
  * Suggested client recovery behavior:
    * Do not retry. Fix the request.


* `422 StatusUnprocessableEntity`
  * Indicates that the requested create or update operation cannot be completed
due to invalid data provided as part of the request.
  * Suggested client recovery behavior:
    * Do not retry. Fix the request.


* `429 StatusTooManyRequests`
  * Indicates that the either the client rate limit has been exceeded or the
server has received more requests then it can process.
  * Suggested client recovery behavior:
    * Read the `Retry-After` HTTP header from the response, and wait at least
that long before retrying.


* `500 StatusInternalServerError`
  * Indicates that the server can be reached and understood the request, but
either an unexpected internal error occurred and the outcome of the call is
unknown, or the server cannot complete the action in a reasonable time (this may
be due to temporary server load or a transient communication issue with another
server).
  * Suggested client recovery behavior:
    * Retry with exponential backoff.


* `503 StatusServiceUnavailable`
  * Indicates that required service is unavailable.
  * Suggested client recovery behavior:
    * Retry with exponential backoff.


* `504 StatusServerTimeout`
  * Indicates that the request could not be completed within the given time.
Clients can get this response ONLY when they specified a timeout param in the
request.
  * Suggested client recovery behavior:
    * Increase the value of the timeout param and retry with exponential
backoff.

## Response Status Kind

Kubernetes will always return the `Status` kind from any API endpoint when an
error occurs. Clients SHOULD handle these types of objects when appropriate.

A `Status` kind will be returned by the API in two cases:
  * When an operation is not successful (i.e. when the server would return a non
2xx HTTP status code).
  * When a HTTP `DELETE` call is successful.

The status object is encoded as JSON and provided as the body of the response.
The status object contains fields for humans and machine consumers of the API to
get more detailed information for the cause of the failure. The information in
the status object supplements, but does not override, the HTTP status code's
meaning. When fields in the status object have the same meaning as generally
defined HTTP headers and that header is returned with the response, the header
should be considered as having higher priority.

**Example:**

```console
$ curl -v -k -H "Authorization: Bearer WhCDvq4VPpYhrcfmF6ei7V9qlbqTubUc" https://10.240.122.184:443/api/v1/namespaces/default/pods/grafana

> GET /api/v1/namespaces/default/pods/grafana HTTP/1.1
> User-Agent: curl/7.26.0
> Host: 10.240.122.184
> Accept: */*
> Authorization: Bearer WhCDvq4VPpYhrcfmF6ei7V9qlbqTubUc
>

< HTTP/1.1 404 Not Found
< Content-Type: application/json
< Date: Wed, 20 May 2015 18:10:42 GMT
< Content-Length: 232
<
{
  "kind": "Status",
  "apiVersion": "v1",
  "metadata": {},
  "status": "Failure",
  "message": "pods \"grafana\" not found",
  "reason": "NotFound",
  "details": {
    "name": "grafana",
    "kind": "pods"
  },
  "code": 404
}
```

`status` field contains one of two possible values:
* `Success`
* `Failure`

`message` may contain human-readable description of the error

`reason` may contain a machine-readable, one-word, CamelCase description of why
this operation is in the `Failure` status. If this value is empty there is no
information available. The `reason` clarifies an HTTP status code but does not
override it.

`details` may contain extended data associated with the reason. Each reason may
define its own extended details. This field is optional and the data returned is
not guaranteed to conform to any schema except that defined by the reason type.

Possible values for the `reason` and `details` fields:
* `BadRequest`
  * Indicates that the request itself was invalid, because the request doesn't
make any sense, for example deleting a read-only object.
  * This is different than `status reason` `Invalid` above which indicates that
the API call could possibly succeed, but the data was invalid.
  * API calls that return BadRequest can never succeed.
  * Http status code: `400 StatusBadRequest`


* `Unauthorized`
  * Indicates that the server can be reached and understood the request, but
refuses to take any further action without the client providing appropriate
authorization. If the client has provided authorization, this error indicates
the provided credentials are insufficient or invalid.
  * Details (optional):
    * `kind string`
      * The kind attribute of the unauthorized resource (on some operations may
differ from the requested resource).
    * `name string`
      * The identifier of the unauthorized resource.
   * HTTP status code: `401 StatusUnauthorized`


* `Forbidden`
  * Indicates that the server can be reached and understood the request, but
refuses to take any further action, because it is configured to deny access for
some reason to the requested resource by the client.
  * Details (optional):
    * `kind string`
      * The kind attribute of the forbidden resource (on some operations may
differ from the requested resource).
    * `name string`
      * The identifier of the forbidden resource.
  * HTTP status code: `403 StatusForbidden`


* `NotFound`
  * Indicates that one or more resources required for this operation could not
be found.
  * Details (optional):
    * `kind string`
      * The kind attribute of the missing resource (on some operations may
differ from the requested resource).
    * `name string`
      * The identifier of the missing resource.
  * HTTP status code: `404 StatusNotFound`


* `AlreadyExists`
  * Indicates that the resource you are creating already exists.
  * Details (optional):
    * `kind string`
      * The kind attribute of the conflicting resource.
    * `name string`
      * The identifier of the conflicting resource.
  * HTTP status code: `409 StatusConflict`

* `Conflict`
  * Indicates that the requested update operation cannot be completed due to a
conflict. The client may need to alter the request. Each resource may define
custom details that indicate the nature of the conflict.
  * HTTP status code: `409 StatusConflict`


* `Invalid`
  * Indicates that the requested create or update operation cannot be completed
due to invalid data provided as part of the request.
  * Details (optional):
    * `kind string`
      * the kind attribute of the invalid resource
    * `name string`
      * the identifier of the invalid resource
    * `causes`
      * One or more `StatusCause` entries indicating the data in the provided
resource that was invalid. The `reason`, `message`, and `field` attributes will
be set.
  * HTTP status code: `422 StatusUnprocessableEntity`


* `Timeout`
  * Indicates that the request could not be completed within the given time.
Clients may receive this response if the server has decided to rate limit the
client, or if the server is overloaded and cannot process the request at this
time.
  * Http status code: `429 TooManyRequests`
  * The server should set the `Retry-After` HTTP header and return
`retryAfterSeconds` in the details field of the object. A value of `0` is the
default.


* `ServerTimeout`
  * Indicates that the server can be reached and understood the request, but
cannot complete the action in a reasonable time. This maybe due to temporary
server load or a transient communication issue with another server.
    * Details (optional):
      * `kind string`
        * The kind attribute of the resource being acted on.
      * `name string`
        * The operation that is being attempted.
  * The server should set the `Retry-After` HTTP header and return
`retryAfterSeconds` in the details field of the object. A value of `0` is the
default.
  * Http status code: `504 StatusServerTimeout`


* `MethodNotAllowed`
  * Indicates that the action the client attempted to perform on the resource
was not supported by the code.
  * For instance, attempting to delete a resource that can only be created.
  * API calls that return MethodNotAllowed can never succeed.
  * Http status code: `405 StatusMethodNotAllowed`


* `InternalError`
  * Indicates that an internal error occurred, it is unexpected and the outcome
of the call is unknown.
  * Details (optional):
    * `causes`
      * The original error.
  * Http status code: `500 StatusInternalServerError` `code` may contain the suggested HTTP return code for this status.


## Events

Events are complementary to status information, since they can provide some
historical information about status and occurrences in addition to current or
previous status. Generate events for situations users or administrators should
be alerted about.

Choose a unique, specific, short, CamelCase reason for each event category. For
example, `FreeDiskSpaceInvalid` is a good event reason because it is likely to
refer to just one situation, but `Started` is not a good reason because it
doesn't sufficiently indicate what started, even when combined with other event
fields.

`Error creating foo` or `Error creating foo %s` would be appropriate for an
event message, with the latter being preferable, since it is more informational.

Accumulate repeated events in the client, especially for frequent events, to
reduce data volume, load on the system, and noise exposed to users.

## Naming conventions

* Go field names must be PascalCase. JSON field names must be camelCase. Other
than capitalization of the initial letter, the two should almost always match.
No underscores or dashes in either.
* Field and resource names should be declarative, not imperative (SomethingDoer, 
DoneBy, DoneAt).
* Use `Node` where referring to
the node resource in the context of the cluster. Use `Host` where referring to
properties of the individual physical/virtual system, such as `hostname`,
`hostPath`, `hostNetwork`, etc.
* `FooController` is a deprecated kind naming convention. Name the kind after
the thing being controlled instead (e.g., `Job` rather than `JobController`).
* The name of a field that specifies the time at which `something` occurs should
be called `somethingTime`. Do not use `stamp` (e.g., `creationTimestamp`).
* We use the `fooSeconds` convention for durations, as discussed in the [units
subsection](#units).
  * `fooPeriodSeconds` is preferred for periodic intervals and other waiting
periods (e.g., over `fooIntervalSeconds`).
  * `fooTimeoutSeconds` is preferred for inactivity/unresponsiveness deadlines.
  * `fooDeadlineSeconds` is preferred for activity completion deadlines.
* Do not use abbreviations in the API, except where they are extremely commonly
used, such as "id", "args", or "stdin".
* Acronyms should similarly only be used when extremely commonly known. All
letters in the acronym should have the same case, using the appropriate case for
the situation. For example, at the beginning of a field name, the acronym should
be all lowercase, such as "httpGet". Where used as a constant, all letters
should be uppercase, such as "TCP" or "UDP".
* The name of a field referring to another resource of kind `Foo` by name should
be called `fooName`. The name of a field referring to another resource of kind
`Foo` by ObjectReference (or subset thereof) should be called `fooRef`.
* More generally, include the units and/or type in the field name if they could
be ambiguous and they are not specified by the value or value type.
* The name of a field expressing a boolean property called 'fooable' should be
called `Fooable`, not `IsFooable`.

### Namespace Names
* The name of a namespace must be a
[DNS_LABEL](https://github.com/kubernetes/community/blob/master/contributors/design-proposals/architecture/identifiers.md).
* The `kube-` prefix is reserved for Kubernetes system namespaces, e.g. `kube-system` and `kube-public`.
* See
[the namespace docs](https://kubernetes.io/docs/user-guide/namespaces/) for more information.

## Label, selector, and annotation conventions

Labels are the domain of users. They are intended to facilitate organization and
management of API resources using attributes that are meaningful to users, as
opposed to meaningful to the system. Think of them as user-created mp3 or email
inbox labels, as opposed to the directory structure used by a program to store
its data. The former enables the user to apply an arbitrary ontology, whereas
the latter is implementation-centric and inflexible. Users will use labels to
select resources to operate on, display label values in CLI/UI columns, etc.
Users should always retain full power and flexibility over the label schemas
they apply to labels in their namespaces.

However, we should support conveniences for common cases by default. For
example, what we now do in ReplicationController is automatically set the RC's
selector and labels to the labels in the pod template by default, if they are
not already set. That ensures that the selector will match the template, and
that the RC can be managed using the same labels as the pods it creates. Note
that once we generalize selectors, it won't necessarily be possible to
unambiguously generate labels that match an arbitrary selector.

If the user wants to apply additional labels to the pods that it doesn't select
upon, such as to facilitate adoption of pods or in the expectation that some
label values will change, they can set the selector to a subset of the pod
labels. Similarly, the RC's labels could be initialized to a subset of the pod
template's labels, or could include additional/different labels.

For disciplined users managing resources within their own namespaces, it's not
that hard to consistently apply schemas that ensure uniqueness. One just needs
to ensure that at least one value of some label key in common differs compared
to all other comparable resources. We could/should provide a verification tool
to check that. However, development of conventions similar to the examples in
[Labels](https://kubernetes.io/docs/user-guide/labels/) make uniqueness straightforward. Furthermore,
relatively narrowly used namespaces (e.g., per environment, per application) can
be used to reduce the set of resources that could potentially cause overlap.

In cases where users could be running misc. examples with inconsistent schemas,
or where tooling or components need to programmatically generate new objects to
be selected, there needs to be a straightforward way to generate unique label
sets. A simple way to ensure uniqueness of the set is to ensure uniqueness of a
single label value, such as by using a resource name, uid, resource hash, or
generation number.

Problems with uids and hashes, however, include that they have no semantic
meaning to the user, are not memorable nor readily recognizable, and are not
predictable. Lack of predictability obstructs use cases such as creation of a
replication controller from a pod, such as people want to do when exploring the
system, bootstrapping a self-hosted cluster, or deletion and re-creation of a
new RC that adopts the pods of the previous one, such as to rename it.
Generation numbers are more predictable and much clearer, assuming there is a
logical sequence. Fortunately, for deployments that's the case. For jobs, use of
creation timestamps is common internally. Users should always be able to turn
off auto-generation, in order to permit some of the scenarios described above.
Note that auto-generated labels will also become one more field that needs to be
stripped out when cloning a resource, within a namespace, in a new namespace, in
a new cluster, etc., and will need to be ignored around when updating a resource
via patch or read-modify-write sequence.

Inclusion of a system prefix in a label key is fairly hostile to UX. A prefix is
only necessary in the case that the user cannot choose the label key, in order
to avoid collisions with user-defined labels. However, I firmly believe that the
user should always be allowed to select the label keys to use on their
resources, so it should always be possible to override default label keys.

Therefore, resources supporting auto-generation of unique labels should have a
`uniqueLabelKey` field, so that the user could specify the key if they wanted
to, but if unspecified, it could be set by default, such as to the resource
type, like job, deployment, or replicationController. The value would need to be
at least spatially unique, and perhaps temporally unique in the case of job.

Annotations have very different intended usage from labels. They are
primarily generated and consumed by tooling and system extensions, or are used
by end-users to engage non-standard behavior of components.  For example, an
annotation might be used to indicate that an instance of a resource expects
additional handling by non-kubernetes controllers. Annotations may carry
arbitrary payloads, including JSON documents.  Like labels, annotation keys can
be prefixed with a governing domain (e.g. `example.com/key-name`).  Unprefixed
keys (e.g. `key-name`) are reserved for end-users.  Third-party components must
use prefixed keys.  Key prefixes under the "kubernetes.io" and "k8s.io" domains
are reserved for use by the kubernetes project and must not be used by
third-parties.

In early versions of Kubernetes, some in-development features represented new
API fields as annotations, generally with the form `something.alpha.kubernetes.io/name` or
`something.beta.kubernetes.io/name` (depending on our confidence in it). This
pattern is deprecated.  Some such annotations may still exist, but no new
annotations may be defined.  New API fields are now developed as regular fields.

Other advice regarding use of labels, annotations, taints, and other generic map keys by
Kubernetes components and tools:
  - Key names should be all lowercase, with words separated by dashes instead of camelCase
    - For instance, prefer `foo.kubernetes.io/foo-bar` over `foo.kubernetes.io/fooBar`, prefer
    `desired-replicas` over `DesiredReplicas`
  - Unprefixed keys are reserved for end-users.  All other labels and annotations must be prefixed.
  - Key prefixes under "kubernetes.io" and "k8s.io" are reserved for the Kubernetes
    project.
    - Such keys are effectively part of the kubernetes API and may be subject
      to deprecation and compatibility policies. 
  - Key names, including prefixes, should be precise enough that a user could
    plausibly understand where it came from and what it is for.
  - Key prefixes should carry as much context as possible.
    - For instance, prefer `subsystem.kubernetes.io/parameter` over `kubernetes.io/subsystem-parameter`
  - Use annotations to store API extensions that the controller responsible for
the resource doesn't need to know about, experimental fields that aren't
intended to be generally used API fields, etc. Beware that annotations aren't
automatically handled by the API conversion machinery.

## WebSockets and SPDY

Some of the API operations exposed by Kubernetes involve transfer of binary
streams between the client and a container, including attach, exec, portforward,
and logging. The API therefore exposes certain operations over upgradeable HTTP
connections ([described in RFC 2817](https://tools.ietf.org/html/rfc2817)) via
the WebSocket and SPDY protocols. These actions are exposed as subresources with
their associated verbs (exec, log, attach, and portforward) and are requested
via a GET (to support JavaScript in a browser) and POST (semantically accurate).

There are two primary protocols in use today:

1.  Streamed channels

    When dealing with multiple independent binary streams of data such as the
remote execution of a shell command (writing to STDIN, reading from STDOUT and
STDERR) or forwarding multiple ports the streams can be multiplexed onto a
single TCP connection. Kubernetes supports a SPDY based framing protocol that
leverages SPDY channels and a WebSocket framing protocol that multiplexes
multiple channels onto the same stream by prefixing each binary chunk with a
byte indicating its channel. The WebSocket protocol supports an optional
subprotocol that handles base64-encoded bytes from the client and returns
base64-encoded bytes from the server and character based channel prefixes ('0',
'1', '2') for ease of use from JavaScript in a browser.

2.  Streaming response

    The default log output for a channel of streaming data is an HTTP Chunked
Transfer-Encoding, which can return an arbitrary stream of binary data from the
server. Browser-based JavaScript is limited in its ability to access the raw
data from a chunked response, especially when very large amounts of logs are
returned, and in future API calls it may be desirable to transfer large files.
The streaming API endpoints support an optional WebSocket upgrade that provides
a unidirectional channel from the server to the client and chunks data as binary
WebSocket frames. An optional WebSocket subprotocol is exposed that base64
encodes the stream before returning it to the client.

Clients should use the SPDY protocols if their clients have native support, or
WebSockets as a fallback. Note that WebSockets is susceptible to Head-of-Line
blocking and so clients must read and process each message sequentially. In
the future, an HTTP/2 implementation will be exposed that deprecates SPDY.


## Validation

API objects are validated upon receipt by the apiserver. Validation errors are
flagged and returned to the caller in a `Failure` status with `reason` set to
`Invalid`. In order to facilitate consistent error messages, we ask that
validation logic adheres to the following guidelines whenever possible (though
exceptional cases will exist).

* Be as precise as possible.
* Telling users what they CAN do is more useful than telling them what they
CANNOT do.
* When asserting a requirement in the positive, use "must".  Examples: "must be
greater than 0", "must match regex '[a-z]+'".  Words like "should" imply that
the assertion is optional, and must be avoided.
* When asserting a formatting requirement in the negative, use "must not".
Example: "must not contain '..'".  Words like "should not" imply that the
assertion is optional, and must be avoided.
* When asserting a behavioral requirement in the negative, use "may not".
Examples: "may not be specified when otherField is empty", "only `name` may be
specified".
* When referencing a literal string value, indicate the literal in
single-quotes. Example: "must not contain '..'".
* When referencing another field name, indicate the name in back-quotes.
Example: "must be greater than \`request\`".
* When specifying inequalities, use words rather than symbols.  Examples: "must
be less than 256", "must be greater than or equal to 0".  Do not use words
like "larger than", "bigger than", "more than", "higher than", etc.
* When specifying numeric ranges, use inclusive ranges when possible.


## Automatic Resource Allocation And Deallocation

API objects often are [union](#Unions) object containing the following:
1. One or more fields identifying the `Type` specific to API object (aka the `discriminator`).
2. A set of N fields, only one of which should be set at any given time - effectively a union.

Controllers operating on the API type often allocate resources based on 
the `Type` and/or some additional data provided by user. A canonical example 
of this is the `Service` API object where resources such as IPs and network ports 
will be set in the API object based on `Type`. When the user does not specify 
resources, they will be allocated, and when the user specifies exact value, they will 
be reserved or rejected.

When the user chooses to change the `discriminator` value (e.g., from `Type X` to `Type Y`) without 
changing any other fields then the system should clear the fields that were used to represent `Type X` 
in the union along with releasing resources that were attached to `Type X`. This should automatically 
happen irrespective of how these values and resources were allocated (i.e., reserved by the user or 
automatically allocated by the system. A concrete example of this is again `Service` API. The system 
allocates resources such as `NodePorts` and `ClusterIPs` and automatically fill in the fields that 
represent them in case of the service is of type `NodePort` or `ClusterIP` (`discriminator` values). 
These resources and the fields representing them are automatically cleared when  the users changes 
service type to `ExternalName` where these resources and field values no longer apply. 	
