---
title: "Terraform on Naver Cloud - Creating a Go Package"
excerpt: "Creating a Simple Go Library to Call APIs in Terraform"

categories:
  - Terraform
tags:
  - [Terraform, NaverCloud]

permalink: /Terraform/OSSCA_8/

toc: true
toc_sticky: true
published: true

date: 2025-02-13
last_modified_at: 2025-02-13
---

Before connecting the Go API to Terraform, I aim to develop a library in Terraform that simplifies API calls.

Reference Client Package
The original client package can be found here:
🔗 [hashicorp-demoapp/hashicups-client-go](https://github.com/hashicorp-demoapp/hashicups-client-go)

The package I developed can be found here:
🔗 [inpyu/hashicups-client-go](https://github.com/inpyu/hashicups-client-go?tab=readme-ov-file)

# Cafe.go
## GetCafes
``` go
func (c *Client) GetCafes() ([]Cafe, error) {
    req, err := http.NewRequest("GET", fmt.Sprintf("%s/cafes", c.HostURL), nil)
    if err != nil {
        return nil, err
    }

    body, err := c.doRequest(req, nil)
    if err != nil {
        return nil, err
    }

    cafes := []Cafe{}
    err = json.Unmarshal(body, &cafes)
    if err != nil {
        return nil, err
    }

    return cafes, nil
}
```
This function retrieves the cafes' URL and stores the request in req.
The response body is then parsed into JSON and returned as a list of cafes.

## GetCafe
``` go
func (c *Client) GetCafe(cafeID string) ([]Cafe, error) {
    req, err := http.NewRequest("GET", fmt.Sprintf("%s/cafes/%s", c.HostURL, cafeID), nil)
    if err != nil {
        return nil, err
    }

    body, err := c.doRequest(req, nil)
    if err != nil {
        return nil, err
    }

    cafes := []Cafe{}
    err = json.Unmarshal(body, &cafes)
    if err != nil {
        return nil, err
    }

    return cafes, nil
}
```
Unlike GetCafes, this function retrieves a single cafe object.
Initially, I didn't think retrieving a single cafe would be necessary, but I added this function during Terraform Provider development when I encountered a use case requiring it.
Like before, it fetches the URL, processes the response, and returns the result.

## CreateCafe
``` go
func (c *Client) CreateCafe(cafes []Cafe) (*Cafe, error) {
    rb, err := json.Marshal(cafes)
    if err != nil {
        return nil, err
    }

    req, err := http.NewRequest("POST", fmt.Sprintf("%s/cafes", c.HostURL), strings.NewReader(string(rb)))
    if err != nil {
        return nil, err
    }

    body, err := c.doRequest(req, nil)
    if err != nil {
        return nil, err
    }

    cafe := Cafe{}
    err = json.Unmarshal(body, &cafe)
    if err != nil {
        return nil, err
    }

    return &cafe, nil
}
```
This function creates a new cafe entry by sending a POST request.

## UpdateCafe
``` go
func (c *Client) UpdateCafe(cafeID string, cafes []Cafe) (*Cafe, error) {
    rb, err := json.Marshal(cafes)
    if err != nil {
        return nil, err
    }

    req, err := http.NewRequest("PUT", fmt.Sprintf("%s/cafes/%s", c.HostURL, cafeID), strings.NewReader(string(rb)))
    if err != nil {
        return nil, err
    }

    body, err := c.doRequest(req, nil)
    if err != nil {
        return nil, err
    }

    cafe := Cafe{}
    err = json.Unmarshal(body, &cafe)
    if err != nil {
        return nil, err
    }

    return &cafe, nil
}
```
This function updates an existing cafe using a PUT request.
### DeleteCafe
``` go
func (c *Client) DeleteCafe(cafeID string) error {
    req, err := http.NewRequest("DELETE", fmt.Sprintf("%s/cafes/%s", c.HostURL), nil)
    if err != nil {
        return err
    }

    body, err := c.doRequest(req, nil)
    if err != nil {
        return err
    }

    if string(body) != "Deleted cafe" {
        return errors.New(string(body))
    }

    return nil
}
```
This function deletes a cafe entry by sending a DELETE request.

# Summary
Since all these methods follow the same basic pattern, I won’t provide redundant explanations. Essentially, each function:
1. Constructs an HTTP request
2. Sends the request and retrieves the response\
3. Processes the response and returns the result

This library simplifies API interactions, making it easier to integrate with Terraform later. 🚀