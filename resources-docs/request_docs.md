# Request

## Overview

The `Request` resource is designed for managing a resource through HTTP requests. It allows you to define how the provider should interact with the remote system by specifying HTTP requests for create, update, and delete operations.


### Specification
Here is an example `Request` resource definition:

  ```yaml
  apiVersion: http.crossplane.io/v1alpha2
  kind: Request
  metadata:
    name: user-dan
  spec:
    forProvider:
      headers:
        Content-Type:
          - application/json
      payload:
        baseUrl: "http://host.docker.internal:5000/users"
        body: |
          {
            "username": "Dan"
          }
      mappings:
        - method: "POST"
          body: |
            {
              username: .payload.body.name, 
              managedby: "crossplane"
            }
          url: .payload.baseUrl
        - method: "GET"
          url: (.payload.baseUrl + "/" + (.response.body.id|tostring)) 
        - method: "PUT"
          body: |
            {
              username: .payload.body.name, 
            }
          url: (.payload.baseUrl + "/" + (.response.body.id|tostring)) 
        - method: "DELETE"
          url: (.payload.baseUrl + "/" + (.response.body.id|tostring)) 
  ```

- headers: Default HTTP request headers.
- payload: Customizable values for HTTP requests, with jq query support [jq Documentation](https://jqlang.github.io/jq/manual/#object-identifier-index).
- mappings: List of mappings, each specifying the HTTP method, URL, and optional request body.
-  secretInjectionConfigs: Optional Configurations for secrets receiving patches from response data.

### Secrets Injection
The DisposableRequest resource supports injecting data from secrets into the request's body and headers using the following syntax: {{ name:namespace:key }} (supported for body and headers only).

### Read-Only / Polling Mode
A `Request` doesn't require a mapping for every action. If a mapping isn't found for CREATE, UPDATE, or REMOVE, the provider logs it and skips that action - it's not treated as an error. Defining only an OBSERVE (GET) mapping therefore turns a `Request` into a periodic, read-only poller: on every reconcile it issues the GET, and - combined with `secretInjectionConfigs` - can extract fields from the response into a Secret without ever attempting to create, update, or delete anything.

This is useful for syncing data from an existing endpoint you don't manage the lifecycle of - for example, mirroring a value from an external API into a Secret so other resources can reference it.

  ```yaml
  apiVersion: http.crossplane.io/v1alpha2
  kind: Request
  metadata:
    name: sync-external-status
  spec:
    forProvider:
      headers:
        Authorization:
          - "Bearer {{ auth:default:token }}"
      payload:
        baseUrl: "https://api.example.com/v1/status"
      mappings:
        # Only an OBSERVE mapping - no CREATE, UPDATE, or REMOVE, so this
        # Request never attempts to modify anything at the remote endpoint.
        - action: OBSERVE
          url: .payload.baseUrl
      secretInjectionConfigs:
        - secretRef:
            name: external-status
            namespace: default
          keyMappings:
            - secretKey: status
              responseJQ: .body.status
  ```

## PUT Mapping - Desired State
The PUT mapping represents your desired state. The body in this mapping should be contained in the GET response. If it's not, a PUT request will be sent with the according body.

Example PUT mapping:

  ```yaml
  apiVersion: http.crossplane.io/v1alpha2
    ...
      mappings:
        ...
        - method: "PUT"
          body: |
            {
              username: .payload.body.name, 
            }
          url: (.payload.baseUrl + "/" + (.response.body.id|tostring)) 
  ```


## Status
The status field of the `Request` resource provides information about the execution status and results of the HTTP requests.

Example `Request` status:
  ```yaml
  status:
    conditions:
      ...
    cache:
      ...
    requestDetails:
      ...
    response:
      body: >-
        {
          "id":"65565b69681e0b47dcea4464",
          "todo_name":"Do Laundry",
          "reminder":"Every 1 hour",
          "responsible":"Dan"
        }
      headers:
        Content-Length:
          - '104'
        Content-Type:
          - application/json
        Date:
          - Thu, 16 Nov 2023 18:11:53 GMT
        Server:
          - uvicorn
      statusCode: 200
  ```


### Usage

Here's an example of using variables from the response:

  ```yaml
  apiVersion: http.crossplane.io/v1alpha2
  kind: Request
  metadata:
    name: user-dan
  spec:
    forProvider:
      ...
      mappings:
        - method: "GET"
          url: (.payload.baseUrl + "/" + (.response.body.id|tostring)) 
      ...
  ```
