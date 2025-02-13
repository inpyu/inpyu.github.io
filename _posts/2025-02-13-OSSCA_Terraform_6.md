---
title: "Terraform on Naver Cloud - Analyzing of the Go API"
excerpt: "Analysis about Terraform Provider Go API"

categories:
  - Terraform
tags:
  - [Terraform, NaverCloud]

permalink: /Terraform/OSSCA_6/

toc: true
toc_sticky: true
published: true

date: 2025-02-13
last_modified_at: 2025-02-13
---

To develop my own API in the form of a Terraform Provider, an API is required first. Therefore, this post will cover the development of a Go CRUD API for Terraform Provider development.

[GitHub Repository - Product API in Go]
🔗 https://github.com/inpyu/product-api-go

[GitHub Repository - Terraform Provider Cafe Package Client]
🔗 https://github.com/inpyu/hashicups-client-go

You can refer to the completed API in the above GitHub repositories.

🔗 https://github.com/hashicorp-demoapp/hashicups-client-go

The API is structured as shown in the diagram. This post will first cover the development of the Product API before moving on to the Client and Provider development.

# Project Structure
```plane
.
├── Dockerfile
├── LICENSE
├── Makefile
├── README.md
├── blueprint
│   ├── README.md
│   ├── config.json
│   └── stack.hcl
├── client
│   ├── http.go
│   └── http_test.go
├── conf.json
├── config
│   ├── config.go
│   └── config_test.go
├── data
│   ├── connection.go
│   ├── mockcon.go
│   └── model
│       ├── cafe.go
│       ├── cafe_test.go
│       ├── coffee.go
│       ├── coffee_test.go
│       ├── ingredient.go
│       ├── ingredient_test.go
│       ├── order.go
│       ├── order_test.go
│       ├── token.go
│       ├── token_test.go
│       ├── user.go
│       └── user_test.go
├── database
│   ├── Dockerfile
│   └── products.sql
├── docker_compose
│   ├── conf.json
│   └── docker-compose.yml
├── functional_tests
│   ├── features
│   │   ├── basic_functionality.feature
│   │   ├── coffee.feature
│   │   ├── order.feature
│   │   └── user.feature
│   ├── helper_test.go
│   └── main_test.go
├── go.mod
├── go.sum
├── handlers
│   ├── auth.go
│   ├── cafe.go
│   ├── coffee.go
│   ├── coffee_test.go
│   ├── health.go
│   ├── ingredients.go
│   ├── ingredients_test.go
│   ├── order.go
│   ├── order_test.go
│   ├── user.go
│   └── user_test.go
├── main
├── main.go
├── open_api.yaml
├── telemetry
│   └── telemetry.go
└── test_docker
    └── docker-compose.yaml
.
├── Dockerfile
├── LICENSE
├── Makefile
├── README.md
├── blueprint
│   ├── README.md
│   ├── config.json
│   └── stack.hcl
├── client
│   ├── http.go
│   └── http_test.go
├── conf.json
├── config
│   ├── config.go
│   └── config_test.go
├── data
│   ├── connection.go
│   ├── mockcon.go
│   └── model
│       ├── cafe.go
│       ├── cafe_test.go
│       ├── coffee.go
│       ├── coffee_test.go
│       ├── ingredient.go
│       ├── ingredient_test.go
│       ├── order.go
│       ├── order_test.go
│       ├── token.go
│       ├── token_test.go
│       ├── user.go
│       └── user_test.go
├── database
│   ├── Dockerfile
│   └── products.sql
├── docker_compose
│   ├── conf.json
│   └── docker-compose.yml
├── functional_tests
│   ├── features
│   │   ├── basic_functionality.feature
│   │   ├── coffee.feature
│   │   ├── order.feature
│   │   └── user.feature
│   ├── helper_test.go
│   └── main_test.go
├── go.mod
├── go.sum
├── handlers
│   ├── auth.go
│   ├── cafe.go
│   ├── coffee.go
│   ├── coffee_test.go
│   ├── health.go
│   ├── ingredients.go
│   ├── ingredients_test.go
│   ├── order.go
│   ├── order_test.go
│   ├── user.go
│   └── user_test.go
├── main
├── main.go
├── open_api.yaml
├── telemetry
│   └── telemetry.go
└── test_docker
    └── docker-compose.yaml
```
## Understanding the Directory Structure
- **client directory**
  - `client/`: Contains HTTP client-related code.
  - `http.go`: Implements the HTTP client.

- **data directory**
  - `data/`: Contains database connection and data model files.
  - `connection.go`: Handles database connections.
  - `mockcon.go`: Implements a mock database connection.
  - `model/`: Contains data models for different entities.
    - `cafe.go`, `coffee.go`, `ingredient.go`, `order.go`, `token.go`, `user.go`: Define models for cafes, coffee, ingredients, orders, tokens, and users.

- **handlers directory**
  - `handlers/`: Contains handlers that process HTTP requests.
  - `auth.go`: Handles authentication-related requests.
  - `cafe.go`, `coffee.go`, `ingredients.go`, `order.go`, `user.go`: Handle requests related to cafes, coffee, ingredients, orders, and users.

## Key Components
- `main.go:`
  - Functions like a controller in a Spring application.
  - Defines routes and request handlers.
- `client/http.go`:
  - Handles interaction with external APIs.
  - Processes incoming HTTP requests.

- `data/model`:
  - Defines entity structures.
- `data/connection.go`:
  - Implements database connection and business logic.
- `handlers`:
  - Processes HTTP requests on the server side.
  - Interacts with the database to return or manipulate data.

While Client and Handler might seem similar, they serve different purposes:
- **Client**: Gathers data and interacts with external APIs.
- **Handler**: Processes requests and interacts with the database on the server side.

---

### `main.go`
Now, let’s examine main.go, its functionalities, and the code implemented in this file.

**Router and Middleware Setup**

The application uses the gorilla/mux package to configure the router and the cors package to handle CORS settings.

```go
r := mux.NewRouter()
r.Use(hckit.TracingMiddleware)
r.Use(cors.New(cors.Options{
    AllowedOrigins: []string{"*"},
    AllowedMethods: []string{"POST", "GET", "OPTIONS", "PUT", "DELETE"},
    AllowedHeaders: []string{"Accept", "content-type", "Content-Length", "Accept-Encoding", "X-CSRF-Token", "Authorization"},
}).Handler)
```
**Registering Handlers**

Various handlers are registered for different endpoints. Each handler is initialized with a database connection and a logger.

```go
healthHandler := handlers.NewHealth(t, logger, db)
r.Handle("/health", healthHandler).Methods("GET")

coffeeHandler := handlers.NewCoffee(db, logger)
r.Handle("/coffees", coffeeHandler).Methods("GET")

ingredientsHandler := handlers.NewIngredients(db, logger)
r.Handle("/coffees/{id:[0-9]+}/ingredients", ingredientsHandler).Methods("GET")

userHandler := handlers.NewUser(db, logger)
r.HandleFunc("/signup", userHandler.SignUp).Methods("POST")

orderHandler := handlers.NewOrder(db, logger)
r.Handle("/orders", authMiddleware.IsAuthorized(orderHandler.GetUserOrders)).Methods("GET")

cafeHandler := handlers.NewCafe(db, logger)
r.Handle("/cafes", cafeHandler).Methods("GET")
r.Handle("/cafes/{id:[0-9]+}", cafeHandler).Methods("GET")
```

**Explanation of Handlers**
- Health Check Handler
  - /health: Checks the application’s health status.
- Coffee Handler
  - /coffees: Retrieves a list of coffee products.
  - Only authenticated users can create a coffee product.
- Ingredient Handler
  - Retrieves the list of ingredients for a specific coffee product.
- User Handler
  - Handles user signup and login.
- Order Handler
  - Authenticated users can view, create, update, and delete orders.

---

### `data/coffee.go`
Now, let’s analyze the model package, which defines coffee-related data and handles JSON serialization and deserialization.

**`Coffees` Type**

The Coffees type is a slice of Coffee objects.

**FromJSON Method**
``` go
func (c *Coffees) FromJSON(data io.Reader) error {
    de := json.NewDecoder(data)
    return de.Decode(c)
}
```
- Deserializes JSON data into a Coffees type.
- Reads JSON data from an io.Reader and decodes it into a Coffees object.

**ToJSON Method**
``` go
func (c *Coffees) ToJSON() ([]byte, error) {
    return json.Marshal(c)
}
```
Serializes a Coffees object into JSON format.

**`Coffee` Struct**
``` go
type Coffee struct {
	ID          int                `db:"id" json:"id"`
	Name        string             `db:"name" json:"name"`
	Teaser      string             `db:"teaser" json:"teaser"`
	Collection  string             `db:"collection" json:"collection"`
	Origin      string             `db:"origin" json:"origin"`
	Color       string             `db:"color" json:"color"`
	Description string             `db:"description" json:"description"`
	Price       float64            `db:"price" json:"price"`
	Image       string             `db:"image" json:"image"`
	CreatedAt   string             `db:"created_at" json:"-"`
	UpdatedAt   string             `db:"updated_at" json:"-"`
	DeletedAt   sql.NullString     `db:"deleted_at" json:"-"`
	Ingredients []CoffeeIngredient `json:"ingredients"`
}
```
- Represents a coffee product stored in the database.
- Uses json and db tags for serialization and database mapping.

This analysis covered the API structure and key components. The next steps involve implementing the client and Terraform Provider.