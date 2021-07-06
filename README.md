<div align="center">
<h1>Health</h1>
</div>

<p align="center">A simple and flexible health check library for Go.</p>
<div align="center">

[![Build](https://github.com/alexliesenfeld/health/actions/workflows/build.yml/badge.svg)](https://github.com/alexliesenfeld/health/actions/workflows/build.yml)
[![codecov](https://codecov.io/gh/alexliesenfeld/health/branch/main/graph/badge.svg?token=V2mVh8RvYE)](https://codecov.io/gh/alexliesenfeld/health)
[![Go Report Card](https://goreportcard.com/badge/github.com/alexliesenfeld/health)](https://goreportcard.com/report/github.com/alexliesenfeld/health)
[![GolangCI](https://golangci.com/badges/github.com/alexliesenfeld/health.svg)](https://golangci.com/r/github.com/alexliesenfeld/health)
[![Go](https://img.shields.io/github/go-mod/go-version/alexliesenfeld/health.svg)](https://github.com/alexliesenfeld/health)
[![FOSSA status](https://app.fossa.com/api/projects/git%2Bgithub.com%2Falexliesenfeld%2Fhealth.svg?type=shield)](https://app.fossa.com/projects/git%2Bgithub.com%2Falexliesenfeld%2Fhealth?ref=badge_shield)

</div>

<p align="center">
    <a href="https://docs.rs/httpmock/">Documentation</a>
    ·
    <a href="https://github.com/alexliesenfeld/health/issues">Report Bug</a>
    ·
    <a href="https://github.com/alexliesenfeld/health/issues">Request Feature</a>
</p>

## Features
This library allows you to build health checks that do not simply return 200 HTTP status codes but actually 
check if the systems that your app requires to function are actually available.

This library provides the following features:

- Request and check-based timeout management.
- Caching support to unburden checked systems during load peeks.
- Request based and fixed-schedule health checks.
- Custom HTTP request middleware to preprocess and postprocess requests.
- Authentication middleware allows separating public and private health check information.
- Provides an [http.Handler](https://golang.org/pkg/net/http/#Handler) that can be easily used with any [mux](https://golang.org/pkg/net/http/#ServeMux).
- Failure tolerance based on fail count and/or time thresholds.

This library can be used to easily integrate with the Kubernetes liveness and readiness checks.

## Example
```go
package main

import (
	"context"
	"database/sql"
	"github.com/alexliesenfeld/health"
	_ "github.com/mattn/go-sqlite3"
	"net/http"
	"time"
)

func main() {
	db, _ := sql.Open("sqlite3", "simple.sqlite")
	defer db.Close()

	router := http.NewServeMux()
	router.Handle("/health", health.New(
		health.WithTimeout(10*time.Second),
		health.WithBasicAuth("username", "password", true),
		health.WithCheck(health.Check{
			Name:  "database",
			Check: db.PingContext,
		}),
		health.WithPeriodicCheck(30*time.Second, health.Check{
			Name: "search",
			Timeout: 5*time.Second,
			Check: func(ctx context.Context) error {
				_, err := http.Get("https://www.google.com")
				return err
			},
		}),
	))

	http.ListenAndServe(":3000", router)
}
```

The request `curl -u username:password http://localhost:3000/health` would then yield the following result:

```json
{
   "status":"DOWN",
   "timestamp":"2021-07-01T08:05:08.522685Z",
   "details":{
      "database":{
         "status":"DOWN",
         "timestamp":"2021-07-01T08:05:14.603364Z",
         "error" : "check timed out"
      },
      "search":{
         "status":"UP",
         "timestamp":"2021-07-01T08:05:08.522685Z"
      }
   }
}
```

## Caching
Health responses are cached to avoid burdening the services that your program checks and to
prevent "denial of service" attacks. Caching can be configured globally and/or be fine-tuned per check. 
If you do not want to use caching altogether, you can disable it using the `health.WithDisabledCache()` 
configuration option.

## Security
The data returned by health checks usually contains sensitive information (such as service names, error messages, etc.).
You probably do not want to expose this information to everyone. For this reason, this library provides support for 
authentication middleware that allows you to hide health details or entirely block requests based on authentication 
success.

Example: Based on the example below, the authentication middleware will respond with a JSON response body that only 
contains the health status, and the corresponding HTTP status code 
(in this case HTTP status code 503 and JSON response body `{ "status":"DOWN" }`).

```go
health.New(
	health.WithBasicAuth("username", "password", true), 
	health.WithCustomAuth(true, func(r *http.Request) error {
		return fmt.Errorf("this simulates authentication failure")
	}), 
	health.WithCheck(health.Check{
		Name:    "database",
		Check: db.PingContext,
	}), 
)
```

Details, such as error messages, services names, etc. are not exposed to the caller. 
This allows you to open health endpoints to the public but only provide details to authenticated sources.

## Periodic Checks
Rather than executing a health check function on every request that is received on the health endpoint,
periodic checks execute the check function in a fixed interval. This allows to respond to HTTP requests
instantly without waiting for the check function to complete. This is especially useful if you
either (1) expect a higher request rate at the health endpoint or (2) you have checks that take a longer time
to complete.

```go
health.New(
	health.WithPeriodicCheck(15*time.Second, health.Check{
		Name:    "slow-check",
		Check:   myLongRunningCheckFunc, // your custom long running check function
	}),
)
```

## Failure Tolerant Checks
This library lets you configure failure tolerant checks that allow some degree of failure up to
predefined thresholds. The check is only considered failed, when tolerance thresholds have been crossed.

### Example: 
Let’s assume that you use a key/value store for caching, and the connection to it is checked by your application as well. 
Your app is capable of running without the key/value store, but it will result in a slowdown. 
If the key/value store is down, your whole application will appear unavailable. This is most likely not what you want.

If the connection cannot be restored for too long, however, there may be a serious problem that requires 
attention. In this case, you still may want a failing health check, so that your app can be automatically restarted 
by your infrastructure and potentially solve the problem
(such as [Kubernetes health checks](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/)). 

Failure tolerant checks allow you to provide this kind of behaviour.


## Metrics
This library does not come with built-in metrics. Its focus is on health checks.

## License
`health` is free software: you can redistribute it and/or modify it under the terms of the MIT Public License.

This program is distributed in the hope that it will be useful, but WITHOUT ANY WARRANTY; without even the implied 
warranty of MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE. See the MIT Public License for more details.

[![FOSSA Status](https://app.fossa.com/api/projects/git%2Bgithub.com%2Falexliesenfeld%2Fhealth.svg?type=large)](https://app.fossa.com/projects/git%2Bgithub.com%2Falexliesenfeld%2Fhealth?ref=badge_large)