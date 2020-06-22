---
title: Publishing Infrastructure
toc: true
weight: 103
indent: true
---

# Publishing Infrastructure

Crossplane allows infrastructure operators to define and compose new kinds of
infrastructure resources then publish them for the application operators they
support to use, all without writing any code.

Infrastructure providers extend Crossplane, enabling it to manage a wide array
of infrastructure resources like Azure SQL servers and AWS ElastiCache clusters.
Infrastructure composition allows infrastructure operators to define, share, and
reuse new kinds of infrastructure resources that are _composed_ of these
infrastructure resources. Infrastructure operators may configure one or more
compositions of any defined resource, and may publish any defined resource to
their application operators, who may then declare that their application
requires that kind of resource.

Composition can be used to build a catalogue of kinds and configuration classes
of infrastructure that fit the needs and opinions of your organisation. As an
infrastructure operator you might define your own `MySQLInstance` resource. This
resource would allow your application operators to configure only the settings
that _your_ organisation needs - perhaps engine version and storage size. All
other settings are deferred to a selectable composition representing a
configuration class like "production" or "staging". Compositions can hide
infrastructure complexity and include policy guardrails so that applications can
easily and safely consume the infrastructure they need, while conforming to your
organisational best-practices.

> Note that composition is an **experimental** feature of Crossplane. Refer to
> [Current Limitations] for information on functionality that is planned but not
> yet implemented.

## Concepts

![Infrastructure Composition Concepts]

A _Composite_ infrastructure resource is composed of other resources. Its
configuration schema is user-defined. The `MySQLInstance` resources in the above
diagram are composite infrastructure resources.

A `Composition` specifies how Crossplane should reconcile a composite
infrastructure resource - i.e. what infrastructure resources it should compose.
For example the Azure `Composition` configures Crossplane to reconcile a
`MySQLInstance` with an Azure `MySQLServer` and a `MySQLServerFirewallRule`.

A _Requirement_ for an infrastructure resource declares that an application
requires a particular kind of composite infrastructure resource, as well as
specifying how to configure that resource. The `MySQLInstanceRequirement`
resources in the above diagram declare that the application pods each require a
`MySQLInstance`.

An `InfrastructureDefinition` defines a new kind of composite infrastructure
resource. The `InfrastructureDefinition` in the above diagram defines the
existence and schema of the `MySQLInstance` composite infrastructure resource.

An `InfrastructurePublication` publishes a defined composite infrastructure
resource, allowing application operators to declare a requirement for it. The
`InfrastructurePublication` in the above diagram allows application operators to
author a `MySQLInstanceRequirement`.

> Note that composite resources and compositions are _cluster scoped_ - they
> exist outside of any Kubernetes namespace. A requirement is a namespaced proxy
> for a composite resource. This enables an RBAC model under which application
> operators may only interact with infrastructure via requirements.

## Defining and Publishing Infrastructure

New kinds of infrastructure resource are typically defined and published by an
infrastructure operator. There are three steps to this process:

1. Define your resource and its schema.
1. Specify one or more possible ways your resource may be composed.
1. Optionally, publish your resource so that applications may require it.

### Define your Infrastructure Resource

Infrastructure resources are defined by an `InfrastructureDefinition`:

```yaml
apiVersion: apiextensions.crossplane.io/v1alpha1
kind: InfrastructureDefinition
metadata:
  # InfrastructureDefinitions follow the constraints of CustomResourceDefinition
  # names. They must be named <plural>.<group>, per the plural and group names
  # configured by the crdSpecTemplate below.
  name: mysqlinstances.example.org
spec:
  # Composite infrastructure resources may optionally expose a connection secret
  # - a Kubernetes Secret containing all of the details a pod might need to
  # connect to the infrastructure resource. Resources that wish to expose a
  # connection secret must declare what keys they support. These keys form a
  # 'contract' - any composition that intends to be compatible with this
  # infrastructure resource must compose resources that supply these connection
  # secret keys.
  connectionSecretKeys:
  - username
  - password
  - hostname
  - port
  # You can specify a default Composition resource to be selected if there is
  # no composition selector or reference was supplied on the Custom Resource.
  defaultCompositionRef:
    name: example-azure
  # Enforced composition will be selected for all instances of this type and it
  # will override the selectors and referencers that are different.
  #
  #enforcedCompositionRef:
  #  name: securemysql.acme.org
  #
  # A template for the spec of a CustomResourceDefinition. Only the group,
  # version, names, validation, and additionalPrinterColumns fields of a CRD
  # spec are supported.
  crdSpecTemplate:
    group: example.org
    version: v1alpha1
    names:
      kind: MySQLInstance
      listKind: MySQLInstanceList
      plural: mysqlinstances
      singular: mysqlinstance
    validation:
      # This schema defines the configuration fields that the composite resource
      # supports. It uses the same structural OpenAPI schema as a Kubernetes CRD
      # - for example, this resource supports a spec.parameters.version enum.
      # The following fields are reserved for Crossplane's use, and will be
      # overwritten if included in this validation schema:
      #
      # - soec.resourceRef
      # - spec.resourceRefs
      # - spec.requirementRef
      # - spec.writeConnectionSecretToRef
      # - spec.reclaimPolicy
      # - status.conditions
      openAPIV3Schema:
        type: object
        properties:
          spec:
            type: object
            properties:
              parameters:
                type: object
                properties:
                  version:
                    description: MySQL engine version
                    type: string
                    enum: ["5.6", "5.7"]
                  storageGB:
                    type: integer
                  location:
                    description: Geographic location of this MySQL server.
                    type: string
                required:
                - version
                - storageGB
                - location
            required:
            - parameters
```

Refer to the Kubernetes documentation on [structural schemas] for full details
on how to configure the `openAPIV3Schema` for your composite resource.

`kubectl describe` can be used to confirm that a new composite infrastructure
resource was successfully defined. Note the `Established` condition and events,
which indicate the process was successful.

```console
$ kubectl describe infrastructuredefinition mysqlinstances.example.org

Name:         mysqlinstances.example.org
Namespace:
Labels:       <none>
Annotations:  API Version:  apiextensions.crossplane.io/v1alpha1
Kind:         InfrastructureDefinition
Metadata:
  Creation Timestamp:  2020-05-15T05:30:44Z
  Finalizers:
    finalizer.apiextensions.crossplane.io
    published.apiextensions.crossplane.io
  Generation:        1
  Resource Version:  1418120
  Self Link:         /apis/apiextensions.crossplane.io/v1alpha1/infrastructuredefinitions/mysqlinstances.example.org
  UID:               f8fedfaf-4dfd-4b8a-8228-6af0f4abd7a0
Spec:
  Connection Secret Keys:
    username
    password
    hostname
    port
  Default Composition Ref:
    Name: example-azure
  Crd Spec Template:
    Group:  example.org
    Names:
      Kind:       MySQLInstance
      List Kind:  MySQLInstanceList
      Plural:     mysqlinstances
      Singular:   mysqlinstance
    Validation:
      openAPIV3Schema:
        Properties:
          Spec:
            Parameters:
              Properties:
                Location:
                  Description:  Geographic location of this MySQL server.
                  Type:         string
                Storage GB:
                  Type:  integer
                Version:
                  Description:  MySQL engine version
                  Enum:
                    5.6
                    5.7
                  Type:  string
              Required:
                version
                storageGB
                location
              Type:  object
            Required:
              parameters
            Type:  object
        Type:      object
    Version:       v1alpha1
Status:
  Conditions:
    Last Transition Time:  2020-05-15T05:30:45Z
    Reason:                Successfully reconciled resource
    Status:                True
    Type:                  Synced
    Last Transition Time:  2020-05-15T05:30:45Z
    Reason:                Created CRD and started controller
    Status:                True
    Type:                  Established
Events:
  Type    Reason                          Age                  From                                                                Message
  ----    ------                          ----                 ----                                                                -------
  Normal  ApplyInfrastructureDefinition   4m10s                apiextensions/infrastructuredefinition.apiextensions.crossplane.io  waiting for CustomResourceDefinition to be established
  Normal  RenderCustomResourceDefinition  55s (x8 over 4m10s)  apiextensions/infrastructuredefinition.apiextensions.crossplane.io  Rendered CustomResourceDefinition
  Normal  ApplyInfrastructureDefinition   55s (x7 over 4m9s)   apiextensions/infrastructuredefinition.apiextensions.crossplane.io  Applied CustomResourceDefinition and (re)started composite controller
```

### Specify How Your Resource May Be Composed

Once a new kind of infrastructure resource is defined Crossplane must be
instructed how to reconcile that kind of infrastructure resource. This is done
by authoring a `Composition`.

A `Composition`:

* Declares one kind of composite infrastructure resource that it satisfies.
* Specifies a "base" configuration for one or more infrastructure resources.
* Specifies "patches" that overlay configuration values from an instance of the
  composite infrastructure resource onto each "base".

Multiple compositions may satisfy a particular kind of composite infrastructure
resource, and the author of a composite resource (or resource requirement) may
select which composition will be used. This allows an infrastructure operator to
offer their application operators a choice between multiple opinionated classes
of infrastructure, allowing them to explicitly specify only some configuration.
An infrastructure operator may offer their application operators the choice
between an "Azure" and a "GCP" composition when creating a `MySQLInstance` for
example, Or they may offer a choice between a "production" and a "staging"
`MySQLInstance` composition. They can also offer a default composition in case
application operators do not supply a composition selector or enforce a specific
composition in order to override the composition choice of users for all instances.
In all cases, the application operator may configure any value supported by the 
composite infrastructure resource's schema, with all other values being deferred
to the composition.

> Note that per [Current Limitations] it is not currently possible to specify a
> default or an enforced composition for a particular kind of infrastructure
> resource, but these options will be added in a future release of Crossplane.
> In the meantime composite resource and/or resource requirement authors must
> specify a composition for their resource to use.

The below `Composition` satisfies the `MySQLInstance` defined in the previous
section by composing an Azure SQL server, firewall rule, and resource group:

```yaml
apiVersion: apiextensions.crossplane.io/v1alpha1
kind: Composition
metadata:
  name: example-azure
  labels:
    purpose: example
    provider: azure
spec:
  # This Composition declares that it satisfies the MySQLInstance composite
  # resource defined above - i.e. it patches "from" a MySQLInstance.
  from:
    apiVersion: example.org/v1alpha1
    kind: MySQLInstance
  # This Composition reconciles an instance of a MySQLInstance by patching from
  # the MySQLInstance "to" new instances of the infrastructure resources below.
  # These resources may be the managed resources of an infrastructure provider
  # such as provider-azure, or other composite infrastructure resources.
  to:
    # A MySQLInstance that uses this Composition will be composed of an Azure
    # ResourceGroup. The "base" for this ResourceGroup specifies the base
    # configuration that may be extended or mutated by the patches below.
  - base:
      apiVersion: azure.crossplane.io/v1alpha3
      kind: ResourceGroup
      spec:
        reclaimPolicy: Delete
        providerRef:
          name: example
    # Patches copy or "overlay" the value of a field path within the composite
    # resource (the MySQLInstance) to a field path within the composed resource
    # (the ResourceGroup). In the below example any labels and annotations will
    # be propagated from the MySQLInstance to the ResourceGroup, as will the
    # location.
    patches:
    - fromFieldPath: "metadata.labels"
      toFieldPath: "metadata.labels"
    - fromFieldPath: "metadata.annotations"
      toFieldPath: "metadata.annotations"
    - fromFieldPath: "spec.parameters.location"
      toFieldPath: "spec.location"
      # Sometimes it is necessary to "transform" the value from the composite
      # resource into a value suitable for the composed resource, for example an
      # Azure based composition may represent geographical locations differently
      # from a GCP based composition that satisfies the same composite resource.
      # This can be done by providing an optional array of transforms, such as
      # the below that will transform the MySQLInstance spec.parameters.location
      # value "us-west" into the ResourceGroup spec.location value "West US".
      transforms:
      - type: map
        map:
          us-west: West US
          us-east: East US
          au-east: Australia East
    # A MySQLInstance that uses this Composition will also be composed of an
    # Azure MySQLServer.
  - base:
      apiVersion: database.azure.crossplane.io/v1beta1
      kind: MySQLServer
      spec:
        forProvider:
          # When this MySQLServer is created it must specify a ResourceGroup in
          # which it will exist. The below resourceGroupNameSelector corresponds
          # to the spec.forProvider.resourceGroupName field of the MySQLServer.
          # It selects a ResourceGroup with a matching controller reference.
          # Two resources that are part of the same composite resource will have
          # matching controller references, so this MySQLServer will always
          # select the ResourceGroup above. If this Composition included more
          # than one ResourceGroup they could be differentiated by matchLabels.
          resourceGroupNameSelector:
            matchControllerRef: true
          administratorLogin: notadmin
          sslEnforcement: Disabled
          sku:
            tier: GeneralPurpose
            capacity: 8
            family: Gen5
          storageProfile:
            backupRetentionDays: 7
            geoRedundantBackup: Disabled
        providerRef:
          name: example
        writeConnectionSecretToRef:
          namespace: crossplane-system
        reclaimPolicy: Delete
    patches:
    - fromFieldPath: "metadata.labels"
      toFieldPath: "metadata.labels"
    - fromFieldPath: "metadata.annotations"
      toFieldPath: "metadata.annotations"
    - fromFieldPath: "metadata.uid"
      toFieldPath: "spec.writeConnectionSecretToRef.name"
      transforms:
        # Transform the value from the MySQLInstance using Go string formatting.
        # This can be used to prefix or suffix a string, or to convert a number
        # to a string. See https://golang.org/pkg/fmt/ for more detail.
      - type: string
        string:
          fmt: "%s-mysqlserver"
    - fromFieldPath: "spec.parameters.version"
      toFieldPath: "spec.forProvider.version"
    - fromFieldPath: "spec.parameters.location"
      toFieldPath: "spec.forProvider.location"
      transforms:
      - type: map
        map:
          us-west: West US
          us-east: East US
          au-east: Australia East
    - fromFieldPath: "spec.parameters.storageGB"
      toFieldPath: "spec.forProvider.storageProfile.storageMB"
      # Transform the value from the MySQLInstance by multiplying it by 1024 to
      # convert Gigabytes to Megabytes.
      transforms:
        - type: math
          math:
            multiply: 1024
    # In addition to a base and patches, this composed MySQLServer declares that
    # it can fulfil the connectionSecretKeys contract required by the definition
    # of the MySQLInstance. This MySQLServer writes a connection secret with a
    # username, password, and endpoint that may be used to connect to it. These
    # connection details will also be exposed via the composite resource's
    # connection secret. Exactly one composed resource must provide each secret
    # key, but different composed resources may provide different keys.
    connectionDetails:
    - fromConnectionSecretKey: username
    - fromConnectionSecretKey: password
      # The name of the required MySQLInstance connection secret key can be
      # supplied if it is different from the connection secret key exposed by
      # the MySQLServer.
    - name: hostname
      fromConnectionSecretKey: endpoint
      # In some cases it may be desirable to inject a fixed connection secret
      # value, for example to expose fixed, non-sensitive connection details
      # like standard ports that are not published to the composed resource's
      # connection secret.
    - name: port
      value: "3306"
    # A MySQLInstance that uses this Composition will also be composed of an
    # Azure MySQLServerFirewallRule.
  - base:
      apiVersion: database.azure.crossplane.io/v1alpha3
      kind: MySQLServerFirewallRule
      spec:
        forProvider:
          resourceGroupNameSelector:
            matchControllerRef: true
          serverNameSelector:
            matchControllerRef: true
          properties:
            startIpAddress: 10.10.0.0
            endIpAddress: 10.10.255.254
            virtualNetworkSubnetIdSelector:
              name: sample-subnet
        providerRef:
          name: example
        reclaimPolicy: Delete
    patches:
    - fromFieldPath: "metadata.labels"
      toFieldPath: "metadata.labels"
    - fromFieldPath: "metadata.annotations"
      toFieldPath: "metadata.annotations"
  # Some composite resources may be "dynamically provisioned" - i.e. provisioned
  # on-demand to satisfy an application's requirement for infrastructure. The
  # writeConnectionSecretsToNamespace and reclaimPolicy fields configure default
  # values used when dynamically provisioning a composite resource; they are
  # explained in more detail below.
  writeConnectionSecretsToNamespace: crossplane-system
  reclaimPolicy: Delete
```

Field paths reference a field within a Kubernetes object via a simple string.
API conventions describe the syntax as "standard JavaScript syntax for accessing
that field, assuming the JSON object was transformed into a JavaScript object,
without the leading dot, such as metadata.name". Array indices are specified via
square braces while object fields may be specified via a period or via square
braces.Kubernetes field paths do not support advanced features of JSON paths,
such as `@`, `$`, or `*`. For example given the below `Pod`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: example-pod
  annotations:
    example.org/a: example-annotation
spec:
  containers:
  - name: example-container
    image: example:latest
    command: [example]
    args: ["--debug", "--example"]
```

* `metadata.name` would contain "example-pod"
* `metadata.annotations['example.org/a']` would contain "example-annotation"
* `spec.containers[0].name` would contain "example-container"
* `spec.containers[0].args[1]` would contain "--example"

> Note that Compositions provide _intentionally_ limited functionality when
> compared to powerful templating and composition tools like Helm or Kustomize.
> This allows a Composition to be a schemafied Kubernetes-native resource that
> can be stored in and validated by the Kubernetes API server at authoring time
> rather than invocation time.

### Publish Your Infrastructure Resource

An infrastructure operator may choose to publish any defined kind of composite
infrastructure resource to their application operators. Doing so is optional -
a kind of resource that is defined but not published may still be created at the
cluster scope, but application operators may not create a namespaced requirement
for that kind of resource.

Infrastructure is published by an `InfrastructurePublication`:

```yaml
apiVersion: apiextensions.crossplane.io/v1alpha1
kind: InfrastructurePublication
metadata:
  # InfrastructurePublications must use the same name as the
  # InfrastructureDefinition that defines the resource they publish.
  name: mysqlinstances.example.org
spec:
  infrastructureDefinitionRef:
    name: mysqlinstances.example.org
```

`kubectl describe` can be used to confirm that a composite infrastructure
resource was successfully published. Note the `Established` condition and
events, which indicate the process was successful.

```console
$ kubectl describe infrastructurepublication mysqlinstances.example.org

Name:         mysqlinstances.example.org
Namespace:
Labels:       <none>
Annotations:  API Version:  apiextensions.crossplane.io/v1alpha1
Kind:         InfrastructurePublication
Metadata:
  Creation Timestamp:  2020-05-15T05:30:44Z
  Finalizers:
    published.apiextensions.crossplane.io
  Generation:        1
  Resource Version:  1418122
  Self Link:         /apis/apiextensions.crossplane.io/v1alpha1/infrastructurepublications/mysqlinstances.example.org
  UID:               534c5151-2adc-4b4b-9248-fa30740e6b32
Spec:
  Infrastructure Definition Ref:
    Name:  mysqlinstances.example.or
Status:
  Conditions:
    Last Transition Time:  2020-05-15T05:30:46Z
    Reason:                Successfully reconciled resource
    Status:                True
    Type:                  Synced
    Last Transition Time:  2020-05-15T05:30:46Z
    Reason:                Created CRD and started controller
    Status:                True
    Type:                  Established
Events:
  Type     Reason                          Age                    From                                                                 Message
  ----     ------                          ----                   ----                                                                 -------
  Normal   ApplyInfrastructurePublication  7m35s                  apiextensions/infrastructurepublication.apiextensions.crossplane.io  waiting for CustomResourceDefinition to be established
  Normal   GetInfrastructureDefinition     7m32s (x7 over 7m36s)  apiextensions/infrastructurepublication.apiextensions.crossplane.io  Got published InfrastructureDefinition
  Normal   RenderCustomResourceDefinition  7m32s (x7 over 7m36s)  apiextensions/infrastructurepublication.apiextensions.crossplane.io  Rendered CustomResourceDefinition
  Normal   ApplyInfrastructurePublication  7m32s (x5 over 7m34s)  apiextensions/infrastructurepublication.apiextensions.crossplane.io  Applied CustomResourceDefinition and (re)started requirement controller
```

### Permit Crossplane to Reconcile Your Infrastructure Resource

Typically Crossplane runs using a service account that does not have access to
reconcile arbitrary kinds of resource. A `ClusterRole` can grant Crossplane
permission to reconcile your newly defined and published resource:

```yaml
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: mysqlinstances.example.org
  labels:
    rbac.crossplane.io/aggregate-to-crossplane: "true"
rules:
- apiGroups:
  - example.org
  resources:
  - mysqlinstances
  - mysqlinstances/status
  - mysqlinstancerequirements
  - mysqlinstancerequirements/status
  verbs:
  - "*"
```

## Using Infrastructure

![Infrastructure Composition Provisioning]

Crossplane offers several ways for both infrastructure operators and application
operators to use the composite infrastructure they've defined and published:

1. Only infrastructure operators can create or manage infrastructure of a kind
   that is not published to application operators.
1. Infrastructure operators can create infrastructure of a published kind. This
   allows an application operator to create a requirement that specifically
   requires the resource the infrastructure operator created.
1. Application operators can create a requirement for infrastructure of a kind
   that has been published, and it will be provisioned on-demand.

Options one and two are frequently referred to as "static provisioning", while
option three is known as "dynamic provisioning".

> Note that infrastructure operator focused Crossplane concepts are cluster
> scoped - they exist outside any namespace. Crossplane assumes infrastructure
> operators will have similar RBAC permissions to cluster administrators, and
> will thus be permitted to manage cluster scoped resources. Application
> operator focused Crossplane concepts are namespaced. Crossplane assumes
> application operators will be permitted access to the namespace(s) in which
> their applications run, and not to cluster scoped resources.

### Creating and Managing Composite Infrastructure

An infrastructure operator may wish to author a composite resource of a kind
that is published - i.e. a resource kind that has an `InfrastructurePublication`
as well as an `InfrastructureDefinition` - so that an application operator may
later author a requirement for that exact resource. This pattern is useful for
infrastructure resources that may take several minutes to provision - the
infrastructure operator can keep a pool of resources available in advance in
order to ensure application requirements may be instantly satisfied.

In some cases an infrastructure operator may wish to use Crossplane to model
composite infrastructure that they do not wish to allow application operators to
provision. Consider a `VPCNetwork` composite resource that creates an AWS VPC
network with an internet gateway, route table, and several subnets. Defining
this resource as a composite allows the infrastructure operator to easily reuse
their configuration, but it does not make sense in their organisation to allow
application operators to create "supporting infrastructure" like a VPC network.

In both of the above scenarios the infrastructure operator may statically
provision a composite resource; i.e. author it directly rather than via its
corresponding requirement. The `MySQLInstance` composite infrastructure resource
defined above could be authored as follows:

```yaml
apiVersion: example.org/v1alpha1
kind: MySQLInstance
metadata:
  # Composite resources are cluster scoped, so there's no need for a namespace.
  name: example
spec:
  # The schema of the spec.parameters object is defined by the earlier example
  # of an InfrastructureDefinition. The location, storageGB, and version fields
  # are patched onto the ResourceGroup, MySQLServer, and MySQLServerFirewallRule
  # that this MySQLInstance composes.
  parameters:
    location: au-east
    storageGB: 20
    version: "5.7"
  # Support for a compositionRef is automatically injected into the schema of
  # all defined composite infrastructure resources. This allows the resource
  # author to explicitly reference a Composition that this composite resource
  # should use - in this case the earlier example-azure Composition. Note that
  # it is also possible to select a composition by labels - see the below
  # MySQLInstanceRequirement for an example of this approach.
  compositionRef:
    name: example-azure
  # Support for a writeConnectionSecretToRef is automatically injected into the
  # schema of all defined composite infrastructure resources. This allows the
  # resource to write a connection secret containing any details required to
  # connect to it - in this case the hostname, username, and password. Composite
  # resource authors may omit this reference if they do not need or wish to
  # write these details.
  writeConnectionSecretToRef:
    namespace: infra-secrets
    name: example-mysqlinstance
  # Support for a reclaimPolicy is automatically injected into the schema of all
  # defined composite infrastructure resources. The reclaim policy applies only
  # to published kinds of infrastructure - it controls whether the resource is
  # deleted or retained when its corresponding Requirement is deleted. If an
  # application authored a MySQLInstanceRequirement for this MySQLInstance then
  # later deleted their MySQLInstanceRequirement this MySQLInstance and all of
  # the resources it composes would be deleted. If the policy were instead set
  # to 'Retain' the MySQLInstance would be retained, for example to allow an
  # infrastructure operator to perform manual cleanup.
  reclaimPolicy: Delete
```

Any updates to the `MySQLInstance` composite infrastructure resource will be
immediately reconciled with the resources it composes. For example if more
storage were needed an update to the `spec.parameters.storageGB` field would
immediately be propagated to the `spec.forProvider.storageProfile.storageMB`
field of the composed `MySQLServer` due to the relationship established between
these two fields by the patches configured in the `example-azure` `Composition`.

`kubectl describe` may be used to examine a composite infrastructure resource.
Note the `Synced` and `Ready` conditions below. The former indicates that
Crossplane is successfully reconciling the composite resource by updating the
composed resources. The latter indicates that all composed resources are also
indicating that they are in condition `Ready`, and therefore the composite
resource should be online and ready to use. More detail about the health and
configuration of the composite resource can be determined by describing each
composite resource. The kinds and names of each composed resource are exposed as
"Resource Refs" - for example `kubectl describe mysqlserver example-zrpgr` will
describe the detailed state of the composed Azure `MySQLServer`.

```console
$ kubectl describe mysqlinstance.example.org

Name:         example
Namespace:
Labels:       <none>
Annotations:  API Version:  example.org/v1alpha1
Kind:         MySQLInstance
Metadata:
  Creation Timestamp:  2020-05-15T06:53:16Z
  Generation:          4
  Resource Version:    1425809
  Self Link:           /apis/example.org/v1alpha1/mysqlinstances/example
  UID:                 f654dd52-fe0e-47c8-aa9b-235c77505674
Spec:
  Composition Ref:
    Name:  example-azure
  Parameters:
    Location:      au-east
    Storage GB:    20
    Version:       5.7
  Reclaim Policy:  Delete
  Resource Refs:
    API Version:  azure.crossplane.io/v1alpha3
    Kind:         ResourceGroup
    Name:         example-wspmk
    UID:          4909ab46-95ef-4ba7-8f7a-e1d9ee1a6b23
    API Version:  database.azure.crossplane.io/v1beta1
    Kind:         MySQLServer
    Name:         example-zrpgr
    UID:          3afb903e-32db-4834-a6e7-31249212dca0
    API Version:  database.azure.crossplane.io/v1alpha3
    Kind:         MySQLServerFirewallRule
    Name:         example-h4zjn
    UID:          602c8412-7c33-4338-a3af-78166c17b1a0
  Write Connection Secret To Ref:
    Name:       example-mysqlinstance
    Namespace:  infra-secrets
Status:
  Conditions:
    Last Transition Time:  2020-05-15T06:56:46Z
    Reason:                Resource is available for use
    Status:                True
    Type:                  Ready
    Last Transition Time:  2020-05-15T06:53:16Z
    Reason:                Successfully reconciled resource
    Status:                True
    Type:                  Synced
Events:
  Type    Reason                   Age                  From                                  Message
  ----    ------                   ----                 ----                                  -------
  Normal  SelectComposition        10s (x7 over 3m40s)  composite/mysqlinstances.example.org  Successfully selected composition
  Normal  PublishConnectionSecret  10s (x7 over 3m40s)  composite/mysqlinstances.example.org  Successfully published connection details
  Normal  ComposeResources         10s (x7 over 3m40s)  composite/mysqlinstances.example.org  Successfully composed resources
```

### Creating an Infrastructure Requirement

Infrastructure requirements represent an application's requirement for a defined
kind of infrastructure, for example the above `MySQLInstance`. Requirements are
a proxy for the kind of resource they require, allowing application operators to
provision and consume infrastructure. An infrastructure requirement may require
pre-existing, statically provisioned infrastructure or it may dynamically
provision infrastructure on-demand. A requirement for a defined kind of
composite infrastructure resource is always suffixed with `Requirement`. A
requirement for a composite infrastructure resource of kind `MySQLInstance` is,
for example, of kind `MySQLInstanceRequirement`.

The below requirement explicitly requires the `MySQLInstance` authored in the
previous example:

```yaml
# The MySQLInstanceRequirement always has the same API group and version as the
# resource it requires. Its kind is always suffixed with Requirement.
apiVersion: example.org/v1alpha1
kind: MySQLInstanceRequirement
metadata:
  # Infrastructure requirements are namespaced.
  namespace: default
  name: example
spec:
  # The schema of the spec.parameters object is defined by the earlier example
  # of an InfrastructureDefinition. The location, storageGB, and version fields
  # are patched onto the ResourceGroup, MySQLServer, and MySQLServerFirewallRule
  # composed by the required MySQLInstance.
  parameters:
    location: au-east
    storageGB: 20
    version: "5.7"
  # Support for a resourceRef is automatically injected into the schema of all
  # published infrastructure requirement resources. The resourceRef specifies
  # the explicit MySQLInstance that is required.
  resourceRef:
    name: example
  # Support for a writeConnectionSecretToRef is automatically injected into the
  # schema of all published infrastructure requirement resources. This allows
  # the resource to write a connection secret containing any details required to
  # connect to it - in this case the hostname, username, and password.
  writeConnectionSecretToRef:
    name: example-mysqlinstance
```

A requirement may omit the `resourceRef` and instead include a `compositionRef`
(as in the previous `MySQLInstance` example) or a `compositionSelector` in order
to trigger dynamic provisioning. A requirement that does not include a reference
to an existing composite infrastructure resource will have a suitable composite
resource provisioned on demand:

```yaml
apiVersion: example.org/v1alpha1
kind: MySQLInstanceRequirement
metadata:
  namespace: default
  name: example
spec:
  parameters:
    location: au-east
    storageGB: 20
    version: "5.7"
  # Support for a compositionSelector is automatically injected into the schema
  # of all published infrastructure requirement resources. This selector selects
  # the example-azure composition by its labels.
  compositionSelector:
    matchLabels:
      purpose: example
      provider: azure
  writeConnectionSecretToRef:
    name: example-mysqlinstance
```

> Note that compositionSelector labels can form a shared language between the
> infrastructure operators who define compositions and the application operators
> who require composite resources. Compositions could be labelled by zone, size,
> or purpose in order to allow application operators to request a class of
> composite resource by describing their needs such as "east coast, production".

Like composite resources, requirements can be examined using `kubectl describe`.
The `Synced` and `Ready` conditions have the same meaning as the `MySQLInstance`
above. The "Resource Ref" indicates the name of the composite resource that was
either explicitly required, or in the case of the below requirement dynamically
provisioned.

```console
$ kubectl describe mysqlinstancerequirement.example.org example

Name:         example
Namespace:    default
Labels:       <none>
Annotations:  API Version:  example.org/v1alpha1
Kind:         MySQLInstanceRequirement
Metadata:
  Creation Timestamp:  2020-05-15T07:08:11Z
  Finalizers:
    finalizer.apiextensions.crossplane.io
  Generation:        3
  Resource Version:  1428420
  Self Link:         /apis/example.org/v1alpha1/namespaces/default/mysqlinstancerequirements/example
  UID:               d87e9580-9d2e-41a7-a198-a39851815840
Spec:
  Composition Selector:
    Match Labels:
      Provider:  azure
      Purpose:   example
  Parameters:
    Location:    au-east
    Storage GB:  20
    Version:     5.7
  Resource Ref:
    Name:  default-example-8t4tb
  Write Connection Secret To Ref:
    Name:  example-mysqlinstance
Status:
  Conditions:
    Last Transition Time:  2020-05-15T07:26:49Z
    Reason:                Resource is available for use
    Status:                True
    Type:                  Ready
    Last Transition Time:  2020-05-15T07:08:11Z
    Reason:                Successfully reconciled resource
    Status:                True
    Type:                  Synced
Events:
  Type    Reason                      Age                    From                                    Message
  ----    ------                      ----                   ----                                    -------
  Normal  ConfigureCompositeResource  8m23s                  requirement/mysqlinstances.example.org  Successfully configured composite resource
  Normal  BindCompositeResource       8m23s (x7 over 8m23s)  requirement/mysqlinstances.example.org  Composite resource is not yet ready
  Normal  BindCompositeResource       4m53s (x4 over 23m)    requirement/mysqlinstances.example.org  Successfully bound composite resource
  Normal  PropagateConnectionSecret   4m53s (x4 over 23m)    requirement/mysqlinstances.example.org  Successfully propagated connection details from composite resource
```

## Current Limitations

Defining and publishing composite infrastructure resources is an experimental
feature of Crossplane. At present the below functionality is planned but not yet
implemented:

* Updates to a composite infrastructure resource are immediately applied to the
  resources it composes, but updates to a requirement are not yet applied to the
  composite resource that was allocated to satisfy the requirement. In a future
  release of Crossplane updating a requirement will update its allocated
  composite infrastructure resource.
* A composite infrastructure resource (or dynamically provisioned requirement)
  must specify a composition reference or selector. In a future Crossplane
  release it will be possible to configure a default composition for resources
  that provide no information about what composition they desire, and to
  configure a single composition that will be enforced for all resources.
* Only three transforms are currently supported - string format, multiplication,
  and map. Crossplane intends to limit the set of supported transforms, and will
  add more as clear use cases appear.
* Compositions are mutable, and updating a composition causes all composite
  resources that use that composition to be updated accordingly. A future
  release of Crossplane may alter this behaviour.
* Composite resources that are retained when the requirement they were allocated
  to is deleted become available for allocation to another requirement. This may
  not be the desired behaviour, for example if the application that previously
  required the composite infrastructure resource wrote sensitive data to it. A
  future release of Crossplane may alter the reclaim policy options.

[Current Limitations]: #current-limitations
[Infrastructure Composition Concepts]: composition-concepts.png
[structural schemas]: https://kubernetes.io/docs/tasks/access-kubernetes-api/custom-resources/custom-resource-definitions/#specifying-a-structural-schema
[Infrastructure Composition Provisioning]: composition-provisioning.png
