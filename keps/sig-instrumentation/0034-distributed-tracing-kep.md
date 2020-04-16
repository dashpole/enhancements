---
title: Leveraging Distributed Tracing to Understand Kubernetes Object Lifecycles
authors:
  - "@Monkeyanator"
  - "@dashpole"
editor: "@dashpole"
owning-sig: sig-instrumentation
participating-sigs:
  - sig-architecture
  - sig-node
  - sig-api-machinery
  - sig-scalability
  - sig-cli
reviewers:
  - "@Random-Liu"
  - "@bogdandrutu"
approvers:
  - "@brancz"
creation-date: 2018-12-04
last-updated: 2020-04-16
status: implementable
---

# Leveraging Distributed Tracing to Understand Kubernetes Object Lifecycles

## Table of Contents

<!-- toc -->
- [Summary](#summary)
- [Motivation](#motivation)
  - [Definitions](#definitions)
  - [Goals](#goals)
  - [Non-Goals](#non-goals)
- [Proposal](#proposal)
  - [Architecture](#architecture)
    - [Tracing API Requests](#tracing-api-requests)
    - [Propagating Context Through Objects](#propagating-context-through-objects)
    - [End-User Interaction](#end-user-interaction)
    - [Controller Behavior](#controller-behavior)
      - [Choosing a Parent Object, and Reading the Context](#choosing-a-parent-object-and-reading-the-context)
      - [Emitting Spans from Controllers](#emitting-spans-from-controllers)
    - [Writing the Trace Context Annotation: The Tracing Admission Controller](#writing-the-trace-context-annotation-the-tracing-admission-controller)
  - [In-tree changes](#in-tree-changes)
    - [Plumbing Context in Client-go](#plumbing-context-in-client-go)
    - [Vendor OpenTelemetry and the OT Exporter](#vendor-opentelemetry-and-the-ot-exporter)
    - [Controlling use of the OpenTelemetry library](#controlling-use-of-the-opentelemetry-library)
    - [Trace Utility Package](#trace-utility-package)
    - [Tracing kube-apiserver requests](#tracing-kube-apiserver-requests)
  - [Out-of-tree changes](#out-of-tree-changes)
    - [Tracing best-practices documentation](#tracing-best-practices-documentation)
- [Access Control](#access-control)
- [Graduation requirements](#graduation-requirements)
- [Alternatives considered](#alternatives-considered)
  - [Object &quot;Root Spans&quot;](#object-root-spans)
  - [Other OpenTelemetry Exporters](#other-opentelemetry-exporters)
  - [Controllers Update TraceContext Annotation, instead of Admission Controller](#controllers-update-tracecontext-annotation-instead-of-admission-controller)
  - [Using ObjectRef + Generation instead of an OT TraceContext](#using-objectref--generation-instead-of-an-ot-tracecontext)
- [Production Readiness Survey](#production-readiness-survey)
- [Implementation History](#implementation-history)
<!-- /toc -->

## Summary

This Kubernetes Enhancement Proposal (KEP) introduces a model for adding distributed tracing to Kubernetes object lifecycles. The inclusion of this trace instrumentation will mark a significant step in making Kubernetes processes more observable, understandable, and debuggable.


## Motivation

Debugging latency issues in Kubernetes is an involved process. There are existing tools which can be used to isolate these issues in Kubernetes, but these methods fall short for various reasons. For instance:

* **Logs**: are fragmented, and finding out which process was the bottleneck involves digging through troves of unstructured text. In addition, logs do not offer higher-level insight into overall system behavior without an extensive background on the process of interest. 
* **Events**: in Kubernetes are only kept for an hour by default, and don't integrate with visualization of analysis tools. To gain trace-like insights would require a large investment in custom tooling.
* **Latency metrics**: can only supply limited metadata because of cardinality constraints.  They are useful for showing _that_ a process was slow, but don't provide insight into _why_ it was slow.
* **Latency Logging**: is a "poor man's" version of tracing that only works within a single binary and outputs log messages.  See [github.com/kubernetes/utils/trace](https://github.com/kubernetes/utils/tree/master/trace).

Distributed tracing provides a unified view of latency information from across many components and plugins. Trace data is structured, and there are numerous established backends for visualizing and querying over it.

### Definitions

**Span**: The smallest unit of a trace.  It has a start and end time, and is attached to a single trace.
**Trace**: A collection of Spans which represents a single process.
**Trace Context**: A reference to a Trace that is designed to be propagated across component boundaries.  Sometimes referred to as the "Span Context".

### Goals

* Make it possible to visualize the progress of objects across distinct Kubernetes components
* Streamline the tedious processes involved in debugging latency issues in Kubernetes
* Make it possible to identify high-level latency regressions, and attribute them back to offending processes


### Non-Goals

* Replace existing logging, metrics, or the events API
* Trace operations from all Kubernetes resource types in a generic manner (i.e. without manual instrumentation)
* Change metrics or logging (e.g. to support trace-metric correlation)
* Access control to tracing backends

## Proposal

### Architecture

#### Tracing API Requests

In the traditional tracing model, a client sends a request to a server and receives a response back.  The kube-apierver and backing storage (e.g. etcd3) follow this model, so we can easily add tracing for API Server requests.  To enable traces to be collected for API requests, the following must be true:

1. The apiserver must propagate the http context of incoming requests through its function stack to the backing storage
1. Kubernetes client libraries must allow passing a context with API requests

To add traces to API requests, owners of the kube-apiserver and backing storage may add Spans to incoming requests, and configure sampling as they see fit.

#### Propagating Context Through Objects

Unlike the traditional RPC model, controllers in kubernetes do not communicate directly with each other directly over HTTP.  Thus, we cannot use http headers to propagate the context between controllers.  Instead, controllers write to and read from objects.  To propagate the trace context from object writer to object reader, we will store the context as an encoded string an object annotation: `<{alpha.|beta.|}trace.kubernetes.io/context`.  For how the annotaiton is written, see the [Writing the Trace Context Annotation: The Tracing Admission Controller](#writing-the-trace-context-annotation-the-tracing-admission-controller) section below.

This means two trace contexts are sent in different forms with Create/Patch/Update requests to the apiserver (one in http headers, one in an annotation).  A trace context is around 32 bytes (16 bytes for the trace ID, 8 bytes for the span ID, and some metadata). See the [w3c spec](https://w3c.github.io/trace-context/#tracestate-field) for details.

#### End-User Interaction

Fundamentally, a trace is a graph showing the fulfilment of user intent.  To make tracing in kubernetes useful, we need to ensure users have the ability to declare their intent for their write to be traced by controllers.

Add a new `--trace` argument to `kubectl`, which generates a new trace context, sets the trace context to be sampled, and uses the context when sending requests to the API Server.  The option is false by default.

Add `context.Context` arguments to k8s.io/client-go client functions.  This will allow users and components to associate API calls with the context of the involved object.  In some cases, such as object creation, we can automatically attach the SpanContext of the provided context to the created object, making propagation simpler.

This also enables kubernetes to be a composable part of a larger system. For example, if an end-user's service creates a pod as part of handing a request, it could do:
```golang
ctx, span := trace.StartSpan(preexistingContext, “create-my-pod”)
defer span.End()
pod, err := c.CoreV1().Pod(myPod.Namespace).Create(ctx, myPod)
if err != nil {
    return err
}
waitForPodToBeRunning(ctx, myPod)
return nil
```

#### Controller Behavior

##### Choosing a Parent Object, and Reading the Context

In the traditional RPC client-server tracing model, a server responds to an incoming request by performing actions including sending outgoing requests.  The server "propagates" the trace context of the incoming request by attaching it to all outgoing requests, forming a tree of requests.  In kubernetes, instead of work being driven by a single incoming request, controllers observe the state of many objects, and perform actions including writing to objects.  Unlike the traditional RPC model, a given controller action is often the result of writes from _multiple_ other controllers.  In order to form a tree from work done by controllers, a given controller action must be attributed to a single parent object.

As stated in [End-User Interaction](#end-user-interaction), a trace is a graph of the **fulfilment** of user intent.  In kubernetes, the fulfilment of user intent can be restated as the process of moving objects to their desired state.  To take v1.Pod as an example, the intent of the user/component that created the pod has been fulfilled when the pod's status moves to Running.  Therefore, an object trace in kubernetes should produce a graph of the process of moving the object from its current state to its desired state.  This means **controllers should associate a given action with the object whose state would be updated by that action**.  For example, when the scheduler assigns a pod to a node, it interacts with many objects, including Pod, Node, Binding, ResourceQuota, and others.  Using the proposed model of controller tracing, the scheduler associates process of scheduling with the Pod object, since that is the object which is moved towards its desired state.  This solves our dilema above, as it gives us a rule for associating an action with a single parent object, and allows us to form a tree of controller actions.

To choose a parent object for a set of actions, controllers must be able to read the context stored in the object.  To do this, we will provide a utility function 
```golang 
trace.WithObject(ctx context.Context, obj meta.Object) context.Context
```
, which can be used to get the context from a parent object.  For example:

```golang
func CreateChildOfMyParent(ctx context.Context, kubeclient *clientset.Interface, myParent, myObj myv1.MyObject) {
+ ctx = trace.WithObject(ctx, myParent)
  // Use the context in kubernetes API requests
  kubeclient.MyV1().MyObject().Create(ctx, myObj)
  // Use the context in http requests
  req, _ := http.NewRequest(...)
  req = req.WithContext(ctx)
  tr.RoundTrip(req)
  // Use the context in grpc requests
  grpc.DialContext(ctx, ...)
}
```

The single additional line above causes all actions performed with `ctx` to use the myParent object's trace context.  For tracing, this causes Spans to be children of the parent object's traceContext.

##### Emitting Spans from Controllers

Some controllers may want to add spans around work they perform in addition to propagating context.  If there were two `myObj` created from the `myParent` object in the example above, that would produce the following tree of spans if each of the servers (api server, http, and grpc) traces the requests:

```
parent
 \_APICreate(myObj1)
 |_APICreate(myObj2)
 |_HttpReq(myObj1)
 |_HttpReq(myObj2)
 |_GrpcReq(myObj1)
 |_GrpcReq(myObj2)
```

The tree has a single parent, with six children.  It would be helpful to be able to group these by each CreateChild call, and to be able to measure the latency of the overall CreateChild function from end-to-end.  It should look like:


```
parent
 \_CreateChild(myObj1)
 | \_APICreate(myObj1)
 | |_HttpReq(myObj1)
 | |_GrpcReq(myObj1)
 |_CreateChild(myObj2)
   \_APICreate(myObj2)
   |_HttpReq(myObj2)
   |_GrpcReq(myObj2)
```

To do this requires our controller to add a span around the function calls that should be grouped logically together.  We can achieve this by adding `trace.StartSpan` in our function above:

```golang
func CreateChildOfMyParent(ctx context.Context, kubeclient *clientset.Interface, myParent, myObj myv1.MyObject) {
  ctx = trace.WithObject(ctx, myParent)
+ ctx, span := trace.StartSpan(ctx, "CreateChild")
+ defer span.End()
  // Use the context in kubernetes API requests
  kubeclient.MyV1().MyObject().Create(ctx, myObj)
  // Use the context in http requests
  req, _ := http.NewRequest(...)
  req = req.WithContext(ctx)
  tr.RoundTrip(req)
  // Use the context in grpc requests
  grpc.DialContext(ctx, ...)
}
```

To configure trace exporting for the component, components can use a `trace.InitializeExporter` function:

```golang
func init() {
  traceutil.InitializeTraceExporter("my-component")
}
```

#### Writing the Trace Context Annotation: The Tracing Admission Controller

The annotation will be added to objects by a mutating admission controller, referred to as the Tracing Admission Controller.  The API Server will propagate the context received in the API request with requests to admission controllers.  The Tracing Admission Controller will extract the trace context from the admission request, and encode it in the object.  If the previous context was sampled, it will preserve the sampling decision of the previous trace context in the new context.  If the new context belongs to a different trace, it will create a [Link](https://github.com/open-telemetry/opentelemetry-specification/blob/master/specification/overview.md#links-between-spans) between the spans.  This link enables viewers of an "Overwritten" trace context to follow the link to see the rest of the trace.

The mutating admission controller will enforce that it is the only writer (by rejecting other updates) of the trace annotation to ensure the above behavior is enforced.

Using a mutating admission controller simplifies the implementation for components, as they only need to send the context along with API Requests.  It also decreases the likelihood that components will not adhere to the guidelines around writing to the trace context annotation.

### In-tree changes

#### Plumbing Context in Client-go

To allow users of client-go to easily propagate context to http requests to the API Server, golang's context.Context will be added to all CRUD methods in client-go.  This excludes informer-based operations.

#### Vendor OpenTelemetry and the OT Exporter

This KEP proposes the use of the [OpenTelemetry tracing framework](https://opentelemetry.io/) to create and export spans to configured backends.

Controllers must use the [OpenTelemetry exporter format](https://github.com/open-telemetry/opentelemetry-proto), which exports traces to a local port.  This format is compatible with the [OpenTelemetry Collector](https://github.com/open-telemetry/opentelemetry-collector), which allows importing and configuring exporters for trace storage backends to be done out-of-tree in addition to other useful features.

#### Controlling use of the OpenTelemetry library

As the community found in the [Metrics Stability Framework KEP](https://github.com/kubernetes/enhancements/blob/master/keps/sig-instrumentation/20190404-kubernetes-control-plane-metrics-stability.md#kubernetes-control-plane-metrics-stability), having control over how the client libraries are used in kubernetes can enable maintainers to enforce policy and make broad improvements to the quality of telemetry.  To enable future improvements to tracing, we will restrict the direct use of the OpenTelemetry library within the kubernetes code base, and provide wrapped versions of functions we wish to expose in a utility library.

#### Trace Utility Package

This package will be able to create spans from the span context embedded in the `trace.kubernetes.io/context` object annotation, in addition to embedding context from spans back into the annotation. This package will facilitate propagating traces through kubernetes objects.  The exported functions include:

```golang
// InitializeTraceExporter initializes the trace exporting service with the provided service name.
// Components should use this initializer to ensure consistent behavior when configuring the trace exporter.
func InitializeTraceExporter(service string)

// WithObject adds the trace context of obj to the Context
func WithObject(ctx context.Context, obj meta.Object) context.Context

// StartSpan is a wrapper around opentelemetry's trace.StartSpan function to control usage and enable future improvements.
func StartSpan(ctx context.Context, spanName string) (context.Context, *ottrace.Span)
```

#### Tracing kube-apiserver requests

We wrap the http server with [othttp](https://github.com/open-telemetry/opentelemetry-go/tree/master/plugin/othttp) to get spans for incoming requests, and add the [otgrpc](https://github.com/grpc-ecosystem/grpc-opentracing/tree/master/go/otgrpc) DialOption to the etcd grpc client.

### Out-of-tree changes

#### Tracing best-practices documentation

This KEP introduces a new form of instrumentation to Kubernetes, which necessitates the creation of guidelines for adding effective, standardized traces to Kubernetes components, [similar to what is found here for metrics](https://github.com/kubernetes/community/blob/master/contributors/devel/instrumentation.md).

This documentation will put forward standards for: 

* How to name spans, attributes, and annotations
* What kind of processes should be wrapped in a span
* When to link spans to other traces
* What kind of data makes for useful attributes
* How to propagate trace context as proposed above

Having these standards in place will ensure that our tracing instrumentation works well with all backends, and that reviewers have concrete criteria to cross-check PRs against. 

## Access Control

Access Control to trace storage backends is out of the scope of kubernetes.  Anyone with write access to the trace backend can create traces, attach additional spans to traces, or do other malicious things regardless of access to kubernetes objects.

That being said, using objects to propagate TraceContext potentially allows access to kubernetes objects to impact telemetry collected.  In particular, anyone with Write (create, update) access to a traced object can send a new TraceContext with a request to the API Server.  As described in [Concurrent Updates](#concurrent-updates), replacing the TraceContext will create a new trace, linked with the previous one.

## Graduation requirements

Alpha

- [] Implement context propagation (not necessarily tracing) for all in-tree controllers as described in the [Controller Behavior](#controller-behavior) section.
- [] Implement tracing of incoming and outgoing http/grpc requests in the kube-apiserver
- [] Implement the Tracing Admission Controller
- [] Add a `--trace` flag to kubectl
- [] E2e testing of context propagation
- [] User-facing documentation
- [] Implement tracing in _some_ controllers and collect feedback

Beta

- [] Solve the "unlimited lifetime" problem for TraceContext
- [] Benchmark kubernetes components using tracing
- [] Publish documentation on examples of how to use the OT Collector with kubernetes
- [] Scalability testing for high-qps components, including kube-apiserver

GA
- [] Document backwards-compatibility guarantees for Spans, if any

## Alternatives considered

### Object "Root Spans"

A previous iteration of this proposal suggested controllers should export a "Root Span" when ending a trace (described in [Controller Behavior](#controller-behavior) above).  However, that would limit a trace to being associated with a single object, since a "Root Span" defines the scope of the trace.  More generally, we shouldn't assume that the creation or update of a single object represents the entirety of end-user intent.  The user or system using kubernetes determines what the user intent is, not kubernetes controllers.

Tracing in a kubernetes cluster must be a composable component within a larger system, and allow external users or systems to define the "Root Span" that defines and bounds the scope of a trace.

### Other OpenTelemetry Exporters

This KEP suggests that we utilize the OpenTelemetry exporter format in all components.  Alternative options include:

1. Add configuration for many exporters in-tree by vendoring multiple "supported" exporters. These exporters are the only compatible backends for tracing in kubernetes.
  a. This places the kubernetes community in the position of curating supported tracing backends
2. Support *both* a curated set of in-tree exporters, and the collector exporter

### Controllers Update TraceContext Annotation, instead of Admission Controller

This would remove the need to run a mutating admission controller.  However, this would add an extra line of code before each write to the API Server in controllers.  During Alpha, prioritize a smaller number of global code changes.  This can be revisited in future stages if necessary.

### Using ObjectRef + Generation instead of an OT TraceContext 

This would allow constructing a tree of spans by using the objectref + generation as the parent pointer.  Given a root object, a graph engine could reconstruct the tree of spans without needing the additional annotation.  However, this does not have a way to control sampling, and Object + Generation cannot be propagated to non-kubernetes components.  Using an open standard allows our telemetry to be interoperable with non-kubernetes components.

## Production Readiness Survey

* Feature enablement and rollback
  - How can this feature be enabled / disabled in a live cluster?  **Feature-gate: ComponentTracing.  All components that are instrumented with tracing must be restarted to enable/disable exporting spans from that component.  Initial components that will emit spans are: kube-apiserver, kube-scheduler, kube-controller-manager, kubelet.  Others may be added in later stages.**
  - Can the feature be disabled once it has been enabled (i.e., can we roll
    back the enablement)?  **Yes, the feature gate can be disabled in all relevant components**
  - Will enabling / disabling the feature require downtime for the control
    plane?  **Yes, control-plane components must be restarted with the feature-gate disabled.**
  - Will enabling / disabling the feature require downtime or reprovisioning
    of a node?  **No, it just requires restarting the kubelet with the feature-gate disabled**
  - What happens if a cluster with this feature enabled is rolled back? What happens if it is subsequently upgraded again?  **No adverse effects in either case.**
  - Are there tests for this?  **No.  The feature hasn't been developed yet.**
* Scalability
  - Will enabling / using the feature result in any new API calls? **No (there was a recent change in the KEP to that effect).**
    Describe them with their impact keeping in mind the [supported limits][]
    (e.g. 5000 nodes per cluster, 100 pods/s churn) focusing mostly on:
     - components listing and/or watching resources they didn't before
     - API calls that may be triggered by changes of some Kubernetes
       resources (e.g. update object X based on changes of object Y)
     - periodic API calls to reconcile state (e.g. periodic fetching state,
       heartbeats, leader election, etc.)
  - Will enabling / using the feature result in supporting new API types? **No**
    How many objects of that type will be supported (and how that translates
    to limitations for users)?
  - Will enabling / using the feature result in increasing size or count
    of the existing API objects?  **Yes.  It adds an annotation to "traced" objects.  The value is a trace context, which is ~32 bytes.  Traced objects will initially include pods, replicasets, and deployments, but may expand to include others over time.  Notably, this annotation should not be added to Events.**
  - Will enabling / using the feature result in increasing time taken
    by any operations covered by [existing SLIs/SLOs][] (e.g. by adding
    additional work, introducing new steps in between, etc.)? **No**
    Please describe the details if so.
  - Will enabling / using the feature result in non-negligible increase
    of resource usage (CPU, RAM, disk IO, ...) in any components?
    Things to keep in mind include: additional in-memory state, additional
    non-trivial computations, excessive access to disks (including increased
    log volume), significant amount of data sent and/or received over
    network, etc. Think through this in both small and large cases, again
    with respect to the [supported limits][].  **The tracing client library has an in-memory cache for outgoing spans.  I believe this is limited to 1Mb by default.  This overhead would apply to all controllers that export spans.  Note that this applies to the kubelet as well, since the kubelet is one of the initial components that will be instrumented.**
* Rollout, Upgrade, and Rollback Planning
* Dependencies
  - Does this feature depend on any specific services running in the cluster
    (e.g., a metrics service)? **Yes.  In the current version of the proposal, users must run the [OpenTelemetry Collector](https://github.com/open-telemetry/opentelemetry-collector) as a daemonset to configure which backend (e.g. jager, zipkin, etc.) they want telemetry sent to.**
  - How does this feature respond to complete failures of the services on
    which it depends?  **Traces will stop being exported, and components will store spans in memory until the buffer is full.  After the buffer fills up, spans will be dropped.**
  - How does this feature respond to degraded performance or high error rates
    from services on which it depends? **If the bi-directional grpc streaming connection to the collector cannot be established or is broken, the controller retries the connection every 5 minutes (by default).**
* Monitoring requirements
  - How can an operator determine if the feature is in use by workloads?  **The operator can check for the presence of the trace context annotation.  Generally, operators are expected to have access to (and likely control over) the OpenTelemetry agent deployment and trace storage backend.**
  - How can an operator determine if the feature is functioning properly?  **TODO: does the client library add metrics about trace exporting?**
  - What are the service level indicators an operator can use to determine the
    health of the service?  **Error rate of sending traces in controllers and OpenTelemetry collector.**
  - What are reasonable service level objectives for the feature?  **Not entirely sure, but I would expect at least 99% of spans to be sent successfully, if not more.**
* Troubleshooting
  - What are the known failure modes?  **The controller is misconfigured, and cannot talk to the collector.  The collector is misconfigured, and can't send traces to the backend.**
  - How can those be detected via metrics or logs?  Logs from the component or agent based on the failure mode.
  - What are the mitigations for each of those failure modes?  **None.  You must correctly configure the collector for tracing to work.**
  - What are the most useful log messages and what logging levels do they require? **All errors are useful, and are logged as errors (no logging levels required). Failure to initialize exporters (in both controller and collector), failures exporting metrics are the most useful.**
  - What steps should be taken if SLOs are not being met to determine the
    problem?  **Look at controller and collector logs.**

## Implementation History

* [Mutating admission webhook which injects trace context](https://github.com/Monkeyanator/mutating-trace-admission-controller)
* [Instrumentation of Kubernetes components](https://github.com/Monkeyanator/kubernetes/pull/15)
* [Instrumentation of Kubernetes components for 1/24/2019 community demo](https://github.com/kubernetes/kubernetes/compare/master...dashpole:tracing)
