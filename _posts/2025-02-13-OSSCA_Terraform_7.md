---
title: "Terraform on Naver Cloud - Developing Go API"
excerpt: "Developing a Cafe CRUD System in Go Lang Based on Previously Analyzed Go API"

categories:
  - Terraform
tags:
  - [Terraform, NaverCloud]

permalink: /Terraform/OSSCA_7/

toc: true
toc_sticky: true
published: true

date: 2025-02-13
last_modified_at: 2025-02-13
---
I plan to develop a Cafe CRUD system in Go Lang based on the Go API I previously analyzed. I intend to follow a step-by-step development approach similar to the existing workflow.

**main.go**

```go
cafeHandler := handlers.NewCafe(db, logger)
r.Handle("/cafes", cafeHandler).Methods("GET")
r.Handle("/cafes/{id:[0-9]+}", cafeHandler).Methods("GET")
r.HandleFunc("/cafes", cafeHandler.CreateCafe).Methods("POST")
r.HandleFunc("/cafes/{id:[0-9]+}", cafeHandler.UpdateCafe).Methods("PUT")
r.HandleFunc("/cafes/{id:[0-9]+}", cafeHandler.DeleteCafe).Methods("DELETE")
```
Looking at `main.go`, the `GET` method uses the `Handle` function, whereas other methods use the HandleFunc function. What is the difference between these two functions, and why are they used differently?

**Handle Method**
```go
func (r *Router) Handle(path string, handler http.Handler) *Route {
    return r.NewRoute().Path(path).Handler(handler)
}
```
- `Handle(path string, handler http.Handler) *Route`
- This method takes an `http.Handler` interface as an argument and registers the path and handler.
- `http.Handler` refers to any type that implements the `ServeHTTP(ResponseWriter, *Request)` method.
- Example: `r.Handle("/path", someHandler)`.

**HandleFunc Method**
``` go
func (r *Router) HandleFunc(path string, f func(http.ResponseWriter, *http.Request)) *Route {
    return r.NewRoute().Path(path).HandlerFunc(f)
}
```
- `HandleFunc(path string, f func(http.ResponseWriter, *http.Request)) *Route`
- This method takes a function-type handler as an argument and registers the path and function.
- The function receives `http.ResponseWriter` and `*http.Request` as parameters.

Example:

```go
r.HandleFunc("/path", func(w http.ResponseWriter, r *http.Request) {...})
```
The key difference between these two methods is that `Handle` takes an `http.Handler` type, whereas `HandleFunc` takes a function-type handler. Thus, in the case of the GET request, `cafeHandler` as a whole can handle the request, while other requests require separate functions, leading to the use of `HandleFunc`.

### Handlers/cafe.go
As implemented in `main.go`, I will define handlers for the cafe-related API endpoints. Each function will be designed to handle requests for a specific HTTP method.

**Struct and Constructor**
``` go
// Cafe - Struct for the Cafe handler
type Cafe struct {
    con data.Connection // Database connection object
    log hclog.Logger    // Logging object
}

// NewCafe - Constructor function for the Cafe struct
func NewCafe(con data.Connection, l hclog.Logger) *Cafe {
    return &Cafe{con, l}
}
```
- The `Cafe` struct has `con` (database connection) and `log` (logging) as attributes.
- `NewCafe` creates and returns an instance of Cafe.
**ServeHTTP Method**
Since the `http.Handler` interface has a `ServeHTTP(http.ResponseWriter, *http.Request)` method, the `Cafe` struct must implement this method to satisfy the interface. Therefore, I will implement the `Read` function inside `ServeHTTP` to handle `GET` requests.

```go
func (c *Cafe) ServeHTTP(rw http.ResponseWriter, r *http.Request) {
    c.log.Info("Handle Cafe")

    vars := mux.Vars(r)
    var cafeID *int

    if vars["id"] != "" {
        cId, err := strconv.Atoi(vars["id"])
        if err != nil {
            c.log.Error("Cafe ID could not be converted to an integer", "error", err)
            http.Error(rw, "Invalid cafe ID", http.StatusInternalServerError)
            return
        }
        cafeID = &cId
    }
```
- The request is processed through `mux.Vars(r)`, which extracts URL path parameters into a map.
- For example, if the URL path is `/cafes/{id}` and the request is `/cafes/123`, `mux.Vars(r)` will return `{"id": "123"}`.
- The `vars` map is used to access the `id`, which is then assigned to `cafeID`.

```go
    cofs, err := c.con.GetCafes(cafeID)
    if err != nil {
        c.log.Error("Unable to get cafes from database", "error", err)
        http.Error(rw, "Unable to list cafes", http.StatusInternalServerError)
        return
    }

    var d []byte
    d, err = json.Marshal(cofs)
    if err != nil {
        c.log.Error("Unable to convert cafes to JSON", "error", err)
        http.Error(rw, "Unable to list cafes", http.StatusInternalServerError)
        return
    }

    rw.Write(d)
}
```
- Using `cafeID`, the `GetCafes` function of `con` is called to retrieve data.
- The retrieved data is then converted to JSON and written as a response.

---

### CreateCafe Method

```go
func (c *Cafe) CreateCafe(rw http.ResponseWriter, r *http.Request) {
    c.log.Info("Handle Cafe | CreateCafe")

    var cafes []model.Cafe
    reqBody, _ := io.ReadAll(r.Body)
    c.log.Info("Request Body", "body", string(reqBody))
    r.Body = io.NopCloser(bytes.NewBuffer(reqBody)) // Reset request body

    err := json.NewDecoder(r.Body).Decode(&cafes)
    if err != nil {
        c.log.Error("Unable to decode JSON", "error", err)
        http.Error(rw, "Unable to parse request body", http.StatusInternalServerError)
        return
    }
```
- The request body is read using `io.ReadAll`.
- The body is then parsed into the `cafes` variable, an array of `model.Cafe`.
```go

    c.log.Info("Decoded Body", "body", cafes)
    cafe := cafes[0]

    createdCafe, err := c.con.CreateCafe(cafe)
    if err != nil {
        c.log.Error("Unable to create new cafe", "error", err)
        http.Error(rw, fmt.Sprintf("Unable to create new cafe: %s", err.Error()), http.StatusInternalServerError)
        return
    }
```
- Since only one cafe object is needed for creation, cafes[0] is used.

**Troubleshooting:**

During development, I faced compatibility issues between object arrays and single objects. Despite needing a single object, I often mistakenly used arrays. This needs to be refactored.

```go
    d, err := createdCafe.ToJSON()
    if err != nil {
        c.log.Error("Unable to convert cafe to JSON", "error", err)
        http.Error(rw, "Unable to create new cafe", http.StatusInternalServerError)
        return
    }

    rw.Write(d)
}
```
- The newly created cafe object is converted to JSON and returned in the response.
---

**UpdateCafe Method**
```go
func (c *Cafe) UpdateCafe(rw http.ResponseWriter, r *http.Request) {
    c.log.Info("Handle Cafe | UpdateCafe")

    vars := mux.Vars(r)
    body := model.Cafe{}
    err := json.NewDecoder(r.Body).Decode(&body)
    if err != nil {
        c.log.Error("Unable to decode JSON", "error", err)
        http.Error(rw, "Unable to parse request body", http.StatusInternalServerError)
        return
    }

    cafeID, err := strconv.Atoi(vars["id"])
    if err != nil {
        c.log.Error("Cafe ID could not be converted to an integer", "error", err)
        http.Error(rw, "Unable to update cafe", http.StatusInternalServerError)
        return
    }

    cafe, err := c.con.UpdateCafe(cafeID, body)
```
- Similar to CreateCafe, but UpdateCafe retrieves id and updates the corresponding cafe record.
---
 
This document provides a step-by-step explanation of implementing a Cafe CRUD system in Go Lang using an API. Let me know if you need any modifications! 😊