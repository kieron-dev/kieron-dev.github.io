---
title: Step by step TDD of a Golang Web Application - Part 4
tags: [tutorial, tdd, birdpedia]
---

## Completing the list birds functionality

### Birds Handler

The previous test gave us the BirdsHandler interface, and we faked that in the
routing unit test.  Now we have to provide a real implementation for that
interface. Let's start with a unit test in a new handlers package.

```bash
$ mkdir handlers
$ (cd handlers; ginkgo bootstrap; ginkgo generate)
```

Here's a minimally failing test for the handler, which doesn't yet compile.

*handlers/handlers_test.go*

```go
package handlers_test

import (
	"net/http"
	"net/http/httptest"

	"github.com/kieron-pivotal/birds/handlers"
	. "github.com/onsi/ginkgo"
	. "github.com/onsi/gomega"
)

var _ = Describe("Handlers", func() {

	var (
		handler *handlers.Birds
	)

	BeforeEach(func() {
		handler = handlers.NewHandler()
	})

	It("writes a bird list as JSON to the response writer", func() {
		recorder := httptest.NewRecorder()
		handler.GetBirds(recorder, nil)

		Expect(recorder.Code).To(Equal(http.StatusOK))
	})

})
```

To make it compile, we must add the handler code.

*handlers/birds.go*

```go
package handlers

import "net/http"

type Birds struct {
}

func NewHandler() *Birds {
	b := &Birds{}
	return b
}

func (b *Birds) GetBirds(w http.ResponseWriter, r *http.Request) {

}
```

We should now finish that test by checking a correct JSON bird list is written
to the response recorder.  Thinking about the birds handler - where does it get
that bird data from? Its responsibility is purely to get the data and write it
in the correct format. So again we've found the need for a new component to do
with data storage.

Let's create another interface called `BirdStorage`, with a `GetBirds()` method
returning a slice of `Bird`s. While we're at it, we must also define that `Bird`
struct with species and description properties. Then we can inject an object of
the interface into the birds handler, and fake it in the unit test.

*handlers/interfaces.go*

```go
package handlers

import "github.com/kieron-pivotal/birds"

//go:generate counterfeiter -o fakes/FakeBirdStorage.go . BirdStorage

type BirdStorage interface {
	GetBirds() ([]birds.Bird, error)
}
```

*bird.go*

```go
package birds

type Bird struct {
	Species     string `json:"species"`
	Description string `json:"description"`
}
```

Once the fake is generated, we can modify the unit test, using it to set up the
expected data.

*handlers/handlers_test.go*

```go
package handlers_test

import (
	"encoding/json"
	"net/http"
	"net/http/httptest"

	"github.com/kieron-pivotal/birds"
	"github.com/kieron-pivotal/birds/handlers"
	"github.com/kieron-pivotal/birds/handlers/fakes"
	. "github.com/onsi/ginkgo"
	. "github.com/onsi/gomega"
)

var _ = Describe("Handlers", func() {

	var (
		handler   *handlers.Birds
		dataStore *fakes.FakeBirdStorage
	)

	BeforeEach(func() {
		dataStore = new(fakes.FakeBirdStorage)
		handler = handlers.NewHandler(dataStore)
	})

	It("writes a bird list as JSON to the response writer", func() {
		birdData := []birds.Bird{
			{Species: "robin", Description: "red breast"},
			{Species: "blackbird", Description: "black bird"},
		}
		birdJSON, err := json.Marshal(birdData)
		Expect(err).NotTo(HaveOccurred())

		dataStore.GetBirdsReturns(birdData, nil)
		recorder := httptest.NewRecorder()
		handler.GetBirds(recorder, nil)

		Expect(recorder.Code).To(Equal(http.StatusOK))
		Expect(recorder.Body.Bytes()).To(Equal(birdJSON))
	})

})
```

Finally, we add the data store parameter to the `NewHandler()` function, and use it
within the `GetBirds()` method to make the test compile and then pass.

*handlers/birds.go*

```go
package handlers

import (
	"encoding/json"
	"net/http"
)

type Birds struct {
	dataStore BirdStorage
}

func NewHandler(dataStore BirdStorage) *Birds {
	b := &Birds{
		dataStore: dataStore,
	}
	return b
}

func (b *Birds) GetBirds(w http.ResponseWriter, r *http.Request) {
	birds, _ := b.dataStore.GetBirds()
	birdsJSON, _ := json.Marshal(birds)
	w.Write(birdsJSON)
}
```

Since the `GetBirds()` method might also return an error, we should create a test for
that. Let's make the handler return an HTTP 500 error in this case.

*handlers/handlers_test.go*

```go
	// ...

	It("returns a server error when GetBirds returns an error", func() {
		dataStore.GetBirdsReturns(nil, errors.New("some error"))
		recorder := httptest.NewRecorder()
		handler.GetBirds(recorder, nil)

		Expect(recorder.Code).To(Equal(http.StatusInternalServerError))
	})
```

To make this pass, we deal with the returned error in the handler.

*handlers/birds.go*

```go
// ...

func (b *Birds) GetBirds(w http.ResponseWriter, r *http.Request) {
	birds, err := b.dataStore.GetBirds()
	if err != nil {
		w.WriteHeader(http.StatusInternalServerError)
		return
	}
	birdsJSON, _ := json.Marshal(birds)
	w.Write(birdsJSON)
}
```

That is it for the handler unit test for now. But again, we've discovered
a new component that needs a real implementation. So let's get on to unit-testing
the data store.

### Data Store

Let's create a new package for the data store, and create the test suite with ginkgo.

```bash
$ mkdir data
$ (cd data; ginkgo bootstrap; ginkgo generate)
```

There isn't much we can test at the moment. We must check that GetBirds returns
the expected list, but since we don't yet have a mechanism for adding birds to the store,
all we can check is that the store returns an empty slice of birds.

*data/data_test.go*

```go
package data_test

import (
	"github.com/kieron-pivotal/birds"
	"github.com/kieron-pivotal/birds/data"
	. "github.com/onsi/ginkgo"
	. "github.com/onsi/gomega"
)

var _ = Describe("Data", func() {

	var (
		store *data.Store
	)

	BeforeEach(func() {
		store = data.NewStore()
	})

	Context("GetBirds", func() {
		It("returns an empty slice of birds", func() {
			birdList, err := store.GetBirds()
			Expect(err).NotTo(HaveOccurred())
			Expect(birdList).To(Equal([]birds.Bird{}))
		})
	})

})
```

We can create a simple in-memory data store to make this test pass. Later we
can substitute it for a postgres data store with the same interface.

*/data/store.go*

```go
package data

import "github.com/kieron-pivotal/birds"

type Store struct {
	birds []birds.Bird
}

func NewStore() *Store {
	s := new(Store)
	s.birds = []birds.Bird{}
	return s
}

func (s *Store) GetBirds() ([]birds.Bird, error) {
	return s.birds, nil
}
```

That is enough for the data unit test for now.

### Revisting the integration test

We now have a set of unit tests testing the chain from router, via handler to
data store.  This is more than enough (as we will fake the data store) for
making our integration test green. So let's modify the integration test to set
up the real components, fake the data store, and see if it now passes.

*integration_tests/birds_test.go*

```go
package api_test

import (
	"io/ioutil"
	"net/http"
	"net/http/httptest"

	"github.com/kieron-pivotal/birds"
	"github.com/kieron-pivotal/birds/handlers"
	"github.com/kieron-pivotal/birds/handlers/fakes"
	"github.com/kieron-pivotal/birds/routing"

	. "github.com/onsi/ginkgo"
	. "github.com/onsi/gomega"
)

var _ = Describe("Birds", func() {

	var (
		server    *httptest.Server
		dataStore *fakes.FakeBirdStorage
		handler   *handlers.Birds
		router    http.Handler
	)

	BeforeEach(func() {
		dataStore = new(fakes.FakeBirdStorage)
		handler = handlers.NewHandler(dataStore)
		router = routing.NewRouter(handler)
		server = httptest.NewServer(router)
	})

	AfterEach(func() {
		server.Close()
	})

	It("returns correct list of birds from GET request to /birds endpoint", func() {
		birdList := []birds.Bird{
			{Species: "housemartin", Description: "member of eighties band"},
			{Species: "eagle", Description: "another musical bird"},
		}
		dataStore.GetBirdsReturns(birdList, nil)

		resp, err := http.Get(server.URL + "/birds")
		Expect(err).NotTo(HaveOccurred())
		Expect(resp.StatusCode).To(Equal(http.StatusOK))

		defer resp.Body.Close()
		body, err := ioutil.ReadAll(resp.Body)
		Expect(err).NotTo(HaveOccurred())
		Expect(body).To(Equal(toJson(birdList)))
	})

})
```

This does indeed now pass. So that's it for the listing birds side of the
integration tests.  Our system test still won't pass, as we haven't addressed
adding birds to the list yet. So that integration test is the [next to
write]({{ site.baseurl }}{% post_url 2018-11-06-birdpedia-with-tdd-5 %}).
