---
title: "Terraform on Naver Cloud - Creating a Custom Provider"
excerpt: "Final Stage: Developing a Custom Provider by Integrating Previously Developed Code"

categories:
  - Terraform
tags:
  - [Terraform, NaverCloud]

permalink: /Terraform/OSSCA_9/

toc: true
toc_sticky: true
published: true

date: 2025-02-13
last_modified_at: 2025-02-13
---
After starting without any knowledge of Go, I encountered numerous challenges. However, writing Terraform code proved to be even more difficult. To solidify my understanding, I will document the development process and the flow I followed while working on this project.

### Basic Setup
``` go
var (
	_ resource.Resource              = &cafeResource{}
	_ resource.ResourceWithConfigure = &cafeResource{}
)

func NewCafeResource() resource.Resource {
	return &cafeResource{}
}

type cafeResource struct {
	client *hashicups.Client
}
```
The code above follows a conventional Go approach to verifying interface implementation. It ensures that `cafeResource` implements both `resource.Resource` and `resource.ResourceWithConfigure`.

### `resource.Resource` Interface
``` go
type Resource interface {
	Metadata(context.Context, MetadataRequest, *MetadataResponse)
	Schema(context.Context, SchemaRequest, *SchemaResponse)
	Create(context.Context, CreateRequest, *CreateResponse)
	Read(context.Context, ReadRequest, *ReadResponse)
	Update(context.Context, UpdateRequest, *UpdateResponse)
	Delete(context.Context, DeleteRequest, *DeleteResponse)
}
```
This interface defines the required functions that must be implemented: `Create`, `Read`, `Update`, and `Delete`. The method names must be consistent with the interface specifications.

### `resource.ResourceWithConfigure` Interface
``` go
type ResourceWithConfigure interface {
	Resource
	Configure(context.Context, ConfigureRequest, *ConfigureResponse)
}
```
The `Configure` method allows the resource to receive provider-level settings and data. By implementing this interface in `cafeResource`, the provider’s configured client and other settings can be received and initialized.

## Defining cafeResource
### Creating NewCafeResource
``` go
func NewCafeResource() resource.Resource {
	return &cafeResource{}
}
```
This function is used to register the resource in `provider.go`:

``` go
func (p *hashicupsProvider) Resources(_ context.Context) []func() resource.Resource {
	return []func() resource.Resource{
		NewOrderResource,
		NewCafeResource,
	}
}
```
Make sure to add NewCafeResource to the list. If a DataSource has also been created, it should be added there as well.

### Defining `cafeResource`
``` go
type cafeResource struct {
	client *hashicups.Client
}
```
This struct contains the definition for the resource that interacts with the HashiCups API. The client field stores the client instance used to communicate with the API. The usage of this client will be explored in more detail later.

---
## Model Definition
``` go
type cafeResourceModel struct {
	ID          types.String `tfsdk:"id"`
	Name        types.String `tfsdk:"name"`
	Address     types.String `tfsdk:"address"`
	Description types.String `tfsdk:"description"`
	Image       types.String `tfsdk:"image"`
}
```
This model is defined to receive data from the API. The types were assigned based on existing models.

### Defining Metadata and Schema

**Metadata**
``` go
func (r *cafeResource) Metadata(_ context.Context, req resource.MetadataRequest, resp *resource.MetadataResponse) {
	resp.TypeName = req.ProviderTypeName + "_cafe"
}
```
This function defines the metadata for the resource. When writing Terraform configuration files, `_cafe` is used to identify the `cafe` resource.

**Schema**

``` go
func (r *cafeResource) Schema(_ context.Context, _ resource.SchemaRequest, resp *resource.SchemaResponse) {
	resp.Schema = schema.Schema{
		Attributes: map[string]schema.Attribute{
			"id": schema.StringAttribute{
				Computed: true,
				PlanModifiers: []planmodifier.String{
					stringplanmodifier.UseStateForUnknown(),
				},
			},
			"name": schema.StringAttribute{
				Optional: true,
			},
			"address": schema.StringAttribute{
				Optional: true,
			},
			"description": schema.StringAttribute{
				Optional: true,
			},
			"image": schema.StringAttribute{
				Optional: true,
			},
		},
	}
}
```
The schema defines the structure of the resource and describes its attributes. This allows Terraform to validate user input, store resource state, and manage updates.

## Roles of the Schema
- Defining and Validating Attributes:
    - Specifies attribute names, types, requirements (required, optional), default values, and other properties.
    - Terraform uses this to validate user input and prevent invalid configurations.
- State Management:
    - Terraform maintains the resource’s state based on the schema, ensuring consistency.
- Planning and Change Management:
    - The schema helps Terraform determine which attributes have changed between configurations.

## Key Schema Components
- Attributes: Define the structure of the resource.
- Computed: Attributes that Terraform determines automatically.
- Required/Optional: Specifies whether attributes must be provided.
- Default: Defines a default value if none is provided.
- Description: Provides documentation for attributes.
- PlanModifiers: Used to modify attribute behavior during the planning phase.
---
## Create Method
``` go
func (r *cafeResource) Create(ctx context.Context, req resource.CreateRequest, resp *resource.CreateResponse) {
	var plan cafeResourceModel
	diags := req.Plan.Get(ctx, &plan)
	resp.Diagnostics.Append(diags...)
	if resp.Diagnostics.HasError() {
		return
	}
```
The Create function extracts data from the CreateRequest. Although the request does not contain a JSON body, gRPC technology is used to structure it.

**Structure of `CreateRequest`**
``` go
type CreateRequest struct {
	Config       tfsdk.Config
	Plan         tfsdk.Plan
	ProviderMeta tfsdk.Config
}
```
By calling req.Plan.Get(ctx, &plan), the request data is automatically mapped to the cafeResourceModel.

``` go
	cafe := hashicups.Cafe{
		Name:        plan.Name.ValueString(),
		Address:     plan.Address.ValueString(),
		Description: plan.Description.ValueString(),
		Image:       plan.Image.ValueString(),
	}

	createdCafe, err := r.client.CreateCafe([]hashicups.Cafe{cafe})
	if err != nil {
		resp.Diagnostics.AddError(
			"Error creating cafe",
			"Could not create cafe, unexpected error: "+err.Error(),
		)
		return
	}
```

A `hashicups.Cafe` object is created using the values from `plan`, retrieved with `ValueString()`. This object is then passed to the API to create a new cafe resource.

---

This document provides an overview of developing a custom Terraform provider using Go, explaining the flow from basic setup to schema definition and resource creation.