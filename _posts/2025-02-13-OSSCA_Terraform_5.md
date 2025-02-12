---
title: "Terraform on Naver Cloud - Migrating to the Terraform Plugin Framework"
excerpt: "Key Differences Between SDKv2 and the Framework"

categories:
  - Terraform
tags:
  - [Terraform, NaverCloud]

permalink: /Terraform/OSSCA_5/

toc: true
toc_sticky: true

date: 2025-02-13
last_modified_at: 2025-02-13
---
While migrating a Terraform provider to the Plugin Framework, understanding the key changes between the two versions can be tricky. To help clarify, I'll review the official documentation comparing the SDKv2 and the Plugin Framework.

You can find the official document here:
[Plugin Framework Benefits | Terraform](https://developer.hashicorp.com/terraform/plugin/framework-benefits#interfaces-instead-of-declarative-structs)

# Overview
HashiCorp provides two Software Development Kits (SDKs) for building Terraform providers using the Go programming language. The first is the latest SDK, the Terraform Plugin Framework, and the second is the previous SDK, SDKv2, which is still used by many existing providers. While SDKv2 continues to be maintained for Terraform 1.x and earlier versions, the majority of development focus has shifted towards improving the Plugin Framework.

The Plugin Framework offers significant advantages over SDKv2 for developing new providers. Additionally, migrating existing providers to the Framework is recommended where possible. If managing large existing providers, terraform-plugin-mux can be used to migrate individual resources or data sources one at a time. For more details, refer to the migration guide from SDK to Framework.

This article is aimed at developers with experience using SDKv2 plugins and explains how the Plugin Framework makes provider development and maintenance easier. If you're new to provider development, the Plugin Framework documentation is a better starting point. For a continuous list of new or improved features in the Plugin Framework, check the framework feature comparison.

# Key Improvements in the Plugin Framework

## Concise Abstractions
The Plugin Framework uses clear concepts and object types, helping developers understand and customize implementations more easily.

### Separate Packages for Each Type
The Framework exposes functionality for each concept as separate packages dedicated to that concept. For example, the datasource package provides functions for implementing data sources, while the provider package contains functions for implementing providers. This separation makes it clear when and how to use each type.

In contrast, SDKv2 requires developers to implement abstract, recursive types like helper/schema.Resource and helper/schema.Schema. These generic abstractions make understanding the specific requirements of each type more difficult. For instance, while data sources require both schema and read functions, blocks only require a schema.

### Improved Data Access
The Framework exposes data in a much more transparent way compared to SDKv2. In SDKv2, most data handling uses helper/schema.ResourceData, which can include merged configuration, plan, or state values depending on the operation. SDKv2 also does not clearly expose the concepts of null or unknown values, often returning zero values of basic Go types, such as an empty string ("") for a null or unknown schema.TypeString.

The Framework, on the other hand, separately exposes the source of data and allows checking if a value is null or unknown.

### More Control Over Built-in Behaviors
The Framework provides greater control over built-in behaviors that might cause confusion or issues in SDKv2. For example, in SDKv2, properties with Computed: true would automatically retain the previous saved state when the user removed them from the configuration. The Framework allows you to specify whether this behavior should be used based on your use case.

### Idiomatic Go Patterns
Since the development of SDKv2, Go's coding standards have evolved. The Framework allows you to use more familiar and modern patterns when writing provider code, making it easier to understand and use.

### Interfaces Instead of Declarative Structs
SDKv2 primarily uses declarative struct types like helper/schema.Resource. Over time, these types have accumulated features that are difficult to distinguish between required and optional functionalities. They also led to runtime errors due to misconfigured code, requiring additional testing and effort to avoid issues.

Instead of declarative structs, the Framework exposes interface types like resource.Resource and resource.ResourceWithImportState. These interfaces cause compiler errors if necessary methods are missing, making development easier.

For example, in SDKv2, a managed resource implementation might lack methods like Read and Delete, which can only be identified through testing:

```
// Example of managed resource definition in SDKv2
&schema.Resource{
  Schema: map[string]*schema.Schema{ /* ... */ },
  CreateContext: func(ctx context.Context, d *schema.ResourceData, meta any) diag.Diagnostics { /* ... */ },
  // Missing Read and Delete methods
}
```

In contrast, with the Framework, the absence of methods like Read() and Delete() in the resource.Resource interface causes a compiler error, making it easier to identify missing functionality upfront.

```
// Ensure the provider definition type satisfies the framework interface
// The missing methods will cause a Go compiler error
var _ resource.Resource = &ThingResource{}

// Example managed resource definition for provider type
type ThingResource struct{}

func (r ThingResource) Create(ctx context.Context, req resource.CreateRequest, resp *resource.CreateResponse) { /* ... */ }
```
### Request and Response Patterns
Terraform providers function similarly to HTTP servers, receiving remote procedure call (RPC) requests from Terraform and returning responses for each operation. SDKv2 and the Framework use different approaches to define the logic for handling these RPC requests.

The Framework adopts a request-response pattern, exposing most of the functions required for RPC requests and including request or response actions in the function signatures. This approach allows the Framework to improve these types over time without requiring updates to the provider code's method signatures. SDKv2, on the other hand, used function fields with signatures that needed to maintain backward compatibility. This resulted in duplicated fields like Create, CreateContext, and CreateWithoutTimeout, which made it harder to use tools like static analysis.

For example, in SDKv2, the CreateContext and ReadContext fields have similar but generic function signatures:

```
// Example managed resource definition in SDKv2
&schema.Resource{
  Schema: map[string]*schema.Schema{ /* ... */ },
  CreateContext: func(ctx context.Context, d *schema.ResourceData, meta any) diag.Diagnostics { /* ... */ },
  ReadContext: func(ctx context.Context, d *schema.ResourceData, meta any) diag.Diagnostics { /* ... */ },
  // Other fields omitted for brevity
}
```
In the Framework, the Create and Read methods adopt a request-response pattern, making it clear what data is available for each operation:

```
// Example managed resource definition for provider type
// Other required methods omitted for brevity
type ThingResource struct{}

func (r ThingResource) Create(ctx context.Context, req resource.CreateRequest, resp *resource.CreateResponse) { /* ... */ }

func (r ThingResource) Read(ctx context.Context, req resource.ReadRequest, resp *resource.ReadResponse) { /* ... */ }
```
### Context Availability
Server-based Go programs typically use context.Context to store and pass data across requests. SDKv2 and the Framework both support context.Context for provider requests, but SDKv2 mixes fields that have context parameters with those that do not. This approach introduces redundant functionality and makes necessary changes difficult to apply. Additionally, metadata fields that are not part of the context.Context can cause confusion.

In SDKv2:
```
// Example managed resource definition in SDKv2
&schema.Resource{
  Schema: map[string]*schema.Schema{ /* ... */ },
  CreateContext: func(ctx context.Context, d *schema.ResourceData, meta any) diag.Diagnostics { /* ... */ },
  ReadContext: func(ctx context.Context, d *schema.ResourceData, meta any) diag.Diagnostics { /* ... */ },
  // Other fields omitted for brevity
}
```

In the Framework:
```
// Example managed resource definition for provider type
// Other required methods omitted for brevity
type ThingResource struct{}

func (r ThingResource) Create(ctx context.Context, req resource.CreateRequest, resp *resource.CreateResponse) { /* ... */ }

func (r ThingResource) Read(ctx context.Context, req resource.ReadRequest, resp *resource.ReadResponse) { /* ... */ }
```

# Conclusion
The latest Terraform Plugin Framework provides several advantages over the older SDKv2, including clearer abstractions, improved data access, more control over built-in behaviors, and the use of idiomatic Go patterns. It simplifies provider development and maintenance. For new provider development, using the Framework is highly recommended, and migrating existing providers to the Framework is advised where possible.