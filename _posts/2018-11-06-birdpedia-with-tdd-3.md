---
title: Step by step TDD of a Golang Web Application - Part 3
---

## Driving out the first of the details

The level of testing below system testing is integration testing. In our
integration tests we want to exercise all the components we control working
together. We use fakes at the boundaries so we don't have to be concerned with
tricky details like databases. This means we can test more cases more easily
than using system tests, which are purely outside-in, with no control of the
internals and no faking at the boundaries.

In this case, the area under test is from the router which assigns HTTP
requests to handler methods, right up to the database interface.

We'll use ginkgo to generate the test files for the integration tests.

```bash
$ mkdir integration_tests/api
$ (cd integration_tests/api; ginkgo bootstrap; ginkgo generate birds)
```

Since we can control the database layer through fakes, we can start with an
integration test checking the listing of birds via the GET request to /birds.
We can use the `httptest` package from the standard library to quickly spin up a
mock HTTP server. This requires an `http.Handler` in its constructor, and here
the idea is to use the `gorilla/mux` Router. As this doesn't yet exist, we can't
write a test that will compile. Also, we don't yet know the interface for the
database layer, so we can't construct a fake. I've left a comment for now.

The integration test is as follows.

*integration_tests/api/birds_test.go:*

```go
package api_test

import (
	"net/http"
	"net/http/httptest"

	"github.com/kieron-pivotal/birds/routing"

	. "github.com/onsi/ginkgo"
	. "github.com/onsi/gomega"
)

var _ = Describe("Birds", func() {

	var (
		server *httptest.Server
		router http.Handler
	)

	BeforeEach(func() {
		router = routing.NewRouter()
		server = httptest.NewServer(router)
	})

	AfterEach(func() {
		server.Close()
	})

	It("returns correct list of birds from GET request to /birds endpoint", func() {
		birdList := []map[string]string{
			{"species": "housemartin", "description": "member of eighties band"},
			{"species": "eagle", "description": "another musical bird"},
		}
		birdListJSON, err := json.Marshal(birdList)
		Expect(err).NotTo(HaveOccurred())

		// TODO: set up database fake with above data

		resp, err := http.Get(server.URL + "/birds")
		Expect(err).NotTo(HaveOccurred())
		Expect(resp.StatusCode).To(Equal(http.StatusOK))

		defer resp.Body.Close()
		body, err := ioutil.ReadAll(resp.Body)
		Expect(err).NotTo(HaveOccurred())
		Expect(body).To(Equal(birdListJSON))
	})

})
```

Our first job is to make this compile by returning a `gorilla/mux` Router in the `routing.NewRouter()` function.

*routing/routes.go:*

```go
package routing

import (
	"net/http"

	"github.com/gorilla/mux"
)

func NewRouter() http.Handler {
	return mux.NewRouter()
}
```

You will need to grab the `gorilla/mux` package at this point. If you're using _dep_,
a `dep init` and a `dep ensure` should be sufficient.

Now our test compiles, but it fails with a HTTP 404 error when performing a GET
request on the /birds endpoint. There are clearly a number of implementation
details that need to be written before this integration test will pass.
We'll dive down to the unit test level to drive out this implementation.

### Diving deeper into unit tests

The integration test is failing at this router component, so this seems the
sensible place to start with a unit test. We need it to respond correctly to a
GET request on the /birds endpoint.

We generate the unit test suite in the routing packing using ginkgo.

```bash
$ (cd routing; ginkgo bootstrap; ginkgo generate)
```

Leaving the generated test suite file alone, the first unit test for the routing component
is simply to make GET /birds return an HTTP 200 status code.

*routing/routing_test.go:*

```go
package routing_test

import (
	"net/http"
	"net/http/httptest"

	"github.com/kieron-pivotal/birds/routing"
	. "github.com/onsi/ginkgo"
	. "github.com/onsi/gomega"
)

var _ = Describe("Routing", func() {

	var (
		router http.Handler
		server *httptest.Server
	)

	BeforeEach(func() {
		router = routing.NewRouter()
		server = httptest.NewServer(router)
	})

	AfterEach(func() {
		server.Close()
	})

	It("returns 200 status code on GET /birds", func() {
		resp, err := http.Get(server.URL + "/birds")
		Expect(err).NotTo(HaveOccurred())
		Expect(resp.StatusCode).To(Equal(http.StatusOK))
	})

})
```

We can make this test pass by registering the /birds route on the router for
the GET method.

*routing/routes.go:*

```go
// ...

func NewRouter() http.Handler {
	r := mux.NewRouter()
	r.HandleFunc("/birds", func(w http.ResponseWriter, r *http.Request) {}).Methods("GET")
	return r
}
```

However that blank implementation of the handler function needs improving. It
doesn't write any data, and its implementation does not belong in the routing
file. Instead the router should delegate the handling code to another object
whose responsibility is purely to gather the correct data based on the request
and to write the data in the correct format in the response.

To force this, we will create an interface with the required method which we
can stub in the unit test. An object implementing the interface will be
injected into the router constructor, and the router can use its method.

We will use [counterfeiter]() to generate a fake from the interface. Here is
the interface with the code generation annotation.

*routing/interfaces.go:*

```go
package routing

import "net/http"

//go:generate counterfeiter -o fakes/BirdsHandler.go . BirdsHandler

type BirdsHandler interface {
	GetBirds(w http.ResponseWriter, r *http.Request)
}
```

To generate the fake struct after creating this file, run the following in a shell.

```bash
$ go generate ./...
```

Then we add a test where we inject a fake handler into the router, stubbing out
the `GetBirds()` method, and check the response. The full test file is shown
below.

*routing/routing_test.go:*

```go
package routing_test

import (
	"encoding/json"
	"io/ioutil"
	"net/http"
	"net/http/httptest"

	"github.com/kieron-pivotal/birds/routing"
	"github.com/kieron-pivotal/birds/routing/fakes"

	. "github.com/onsi/ginkgo"
	. "github.com/onsi/gomega"
)

var _ = Describe("Routing", func() {

	var (
		birdsHandler  *fakes.FakeBirdsHandler
		router        http.Handler
		server        *httptest.Server
		testBirdsJSON []byte
	)

	BeforeEach(func() {
		birdsHandler = new(fakes.FakeBirdsHandler)
		router = routing.NewRouter(birdsHandler)
		server = httptest.NewServer(router)

		testBirds := []map[string]string{
			{"species": "housemartin", "description": "member of eighties band"},
			{"species": "eagle", "description": "another musical bird"},
		}
		var err error
		testBirdsJSON, err = json.Marshal(testBirds)
		Expect(err).NotTo(HaveOccurred())
	})

	AfterEach(func() {
		server.Close()
	})

	It("returns 200 status code on GET /birds", func() {
		resp, err := http.Get(server.URL + "/birds")
		Expect(err).NotTo(HaveOccurred())
		Expect(resp.StatusCode).To(Equal(http.StatusOK))
	})

	It("returns a list of birds on GET /birds", func() {
		birdsHandler.GetBirdsStub = func(w http.ResponseWriter, r *http.Request) {
			w.Write(testBirdsJSON)
		}
		resp, err := http.Get(server.URL + "/birds")
		Expect(err).NotTo(HaveOccurred())
		defer resp.Body.Close()

		body, err := ioutil.ReadAll(resp.Body)
		Expect(err).NotTo(HaveOccurred())
		Expect(birdsHandler.GetBirdsCallCount()).To(Equal(1))
		Expect(body).To(Equal(testBirdsJSON))
	})

})
```

Finally we can change the signature of the NewRouter() function to make the test compile,
and pass the GetBirds function from the handler to the router.HandleFunc method to 
make the test pass.

*routing/routes.go:*

```go
// ...

func NewRouter(handler BirdsHandler) http.Handler {
	r := mux.NewRouter()
	r.HandleFunc("/birds", handler.GetBirds).Methods("GET")
	return r
}
```

This is it for the routing code for the time being. The integration test is
currently not compiling, as the NewRouter function requires a real handler as a
parameter now. Both this and the new interface without a real implementation
point us towards the handler as the next area of work.
