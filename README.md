# Goth

## Functionality

* Exposes an HTTP handler that retrieves health status of the application
* Implements some generic checkers for the following services:
  * RabbitMQ
  * PostgreSQL
  * Redis
  * HTTP
  * MongoDB
  * MySQL
  * gRPC
  * Memcached

## Usage

The library exports `Handler` and `HandlerFunc` functions which are fully compatible with `net/http`.

Additionally, library exports `Measure` function that returns summary status for all the registered health checks,
so it can be used in non-HTTP environments.

### Handler

```go
package main

import (
	"context"
	"net/http"
	"time"

	"github.com/h-varmazyar/goth"
	healthMysql "github.com/h-varmazyar/goth/checks/mysql"
    healthRabbit "github.com/h-varmazyar/goth/checks/rabbitmq"
)

func main() {
	// add some checks on instance creation
	h, _ := goth.New(goth.WithChecks(goth.Config{
		Name:      "mongodb",
		Timeout:   time.Second * 5,
		SkipOnErr: true,
		Check: func(ctx context.Context) error {
			// rabbitmq health check implementation goes here
			return nil
		}}, goth.Config{
		Name: "rabbitmq",
		Check: healthRabbit.New(healthRabbit.Config{
            DSN: "amqp://user:password@host:port",
        }),
	},
	))

	// and then add some more if needed
	h.Register(goth.Config{
		Name:      "mysql",
		Timeout:   time.Second * 2,
		SkipOnErr: false,
		Check: healthMysql.New(healthMysql.Config{
			DSN: "test:test@tcp(0.0.0.0:31726)/test?charset=utf8",
		}),
	})

	http.Handle("/status", h.Handler())
	http.ListenAndServe(":3000", nil)
}
```

### HandlerFunc
```go
package main

import (
	"context"
	"net/http"
	"time"

	"github.com/go-chi/chi"
	"github.com/h-varmazyar/goth"
	healthMysql "github.com/h-varmazyar/goth/checks/mysql"
    healthRabbit "github.com/h-varmazyar/goth/checks/rabbitmq"
)

func main() {
	// add some checks on instance creation
	h, _ := goth.New(goth.WithChecks(goth.Config{
		Name:      "rabbitmq",
		Timeout:   time.Second * 5,
		SkipOnErr: true,
		Check: healthRabbit.New(healthRabbit.Config{
          DSN: "amqp://user:password@host:port",
        })}, goth.Config{
		Name: "mongodb",
		Check: func(ctx context.Context) error {
			// mongo_db health check implementation goes here
			return nil
		},
	},
	))

	// and then add some more if needed
	h.Register(goth.Config{
		Name:      "mysql",
		Timeout:   time.Second * 2,
		SkipOnErr: false,
		Check: healthMysql.New(healthMysql.Config{
			DSN: "test:test@tcp(0.0.0.0:31726)/test?charset=utf8",
		}),
	})

	r := chi.NewRouter()
	r.Get("/status", h.HandlerFunc)
	http.ListenAndServe(":3000", nil)
}
```

For more examples please check [here](https://github.com/h-varmazyar/goth/blob/main/_examples/server.go)

## API Documentation

### `GET /status`

Get the health of the application.

- Method: `GET`
- Endpoint: `/status`
- Request:
- Params:
  - service_name: the name of registered service for getting status of certain service. this param is optional. if you want to get all services status ignore this parameter.
```
curl localhost:3000/status?service_name={name_of_registerd_check}
```
- Response:

HTTP/1.1 200 OK
```json
{
  "status": "OK",
  "timestamp": "2017-01-01T00:00:00.413567856+033:00",
  "system": {
    "version": "go1.8",
    "goroutines_count": 4,
    "total_alloc_bytes": 21321,
    "heap_objects_count": 21323,
    "alloc_bytes": 234523
  }
}
```

HTTP custom code 509
```json
{
  "status": "Partially Available",
  "timestamp": "2017-01-01T00:00:00.413567856+033:00",
  "failures": {
    "rabbitmq": "Failed during rabbitmq health check"
  },
  "system": {
    "version": "go1.8",
    "goroutines_count": 4,
    "total_alloc_bytes": 21321,
    "heap_objects_count": 21323,
    "alloc_bytes": 234523
  }
}
```

HTTP/1.1 503 Service Unavailable
```json
{
  "status": "Unavailable",
  "timestamp": "2017-01-01T00:00:00.413567856+033:00",
  "failures": {
    "mongodb": "Failed during mongodb health check"
  },
  "system": {
    "version": "go1.8",
    "goroutines_count": 4,
    "total_alloc_bytes": 21321,
    "heap_objects_count": 21323,
    "alloc_bytes": 234523
  }
}
```

## Contributing

- Fork it
- Create your feature branch (`git checkout -b my-new-feature`)
- Commit your changes (`git commit -am 'Add some feature'`)
- Push to the branch (`git push origin my-new-feature`)
- Create new Pull Request
