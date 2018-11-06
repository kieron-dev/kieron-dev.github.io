---
title: Step by step TDD of a Golang Web Application - Part 5
---

## Adding birds

We now need an integration test for adding a bird with a POST request to the
/birds endpoint. We'll follow the same process of writing unit tests for the
components involved until the integration test passes. This time will be
slightly easier since the components already exist, so we'll be adding to
existing structs and interfaces.

Like the integration test for listing birds, the area under test will be from
the router up to the data store. We'll fake the web server and the data store.

*integration_tests/birds_test.go*

```go
	// ...

	It("adds a new bird to the data store after a POST request to /birds endpoint", func() {
		bird := birds.Bird{
			Species:     "foo",
			Description: "bar",
		}
		data := url.Values{}
		data.Add("species", bird.Species)
		data.Add("description", bird.Description)
		resp, err := http.PostForm(server.URL+"/birds", data)

		Expect(err).NotTo(HaveOccurred())
		Expect(resp.StatusCode).To(Equal(http.StatusCreated))
		Expect(dataStore.AddBirdCallCount()).To(Equal(1))

		actualBird := dataStore.AddBirdArgsForCall(0)
		Expect(actualBird).To(Equal(bird))
	})
```

Here we test that the POST request ends up with the corresponding data being
sent to an `AddBird()` method I've just invented for the `BirdStorage`
interface.  We can make the test compile by extending the interface and
regenerating the fake code.

*handlers/interfaces.go*

```go
// ...

type BirdStorage interface {
	GetBirds() ([]birds.Bird, error)
	AddBird(bird birds.Bird) error
}
```

The new integration test fails at the first step returning an HTTP method not
allowed error.

So let's dive down to the unit tests, starting at the router.

*routing/routing_test.go*

```go
	// ...

	It("passes a POST to /birds off to the AddBird handler", func() {
		bird := birds.Bird{
			Species:     "foo",
			Description: "bar",
		}
		data := url.Values{}
		data.Add("species", bird.Species)
		data.Add("description", bird.Description)
		_, err := http.PostForm(server.URL+"/birds", data)
		Expect(err).NotTo(HaveOccurred())
		Expect(birdsHandler.AddBirdCallCount()).To(Equal(1))
	})
```

Here we've imagined a new method on the handler, `AddBird(),` and we check it's
called once. We need to add the method to the `BirdsHandler` interface,
regenerate the fakes code, and then the test will compile.

To make it pass, we need to add the route for a POST to /birds and attach the
handler method.

*routing/routes.go*

```go
// ...

func NewRouter(handler BirdsHandler) http.Handler {
	r := mux.NewRouter()
	r.HandleFunc("/birds", handler.GetBirds).Methods("GET")
	r.HandleFunc("/birds", handler.AddBird).Methods("POST")
	return r
}
```

Now we can move on to the handler which needs a method to fully implement its
interface. We'll write a unit test for this new AddBird method.

*handlers/handlers_test.go*

```go
	// ...

	It("calls the data store to create a bird and returns 201 status code", func() {
		bird := birds.Bird{
			Species:     "foo",
			Description: "bar",
		}
		data := url.Values{}
		data.Add("species", bird.Species)
		data.Add("description", bird.Description)

		request := httptest.NewRequest("POST", "/birds", strings.NewReader(data.Encode()))
		request.Header.Add("Content-Type", "application/x-www-form-urlencoded")
		recorder := httptest.NewRecorder()
		handler.AddBird(recorder, request)

		Expect(dataStore.AddBirdCallCount()).To(Equal(1))
		Expect(dataStore.AddBirdArgsForCall(0)).To(Equal(bird))
		Expect(recorder.Code).To(Equal(http.StatusCreated))
	})
```

To make this compile, we need to create the AddBird method on the handler.
Then by grabbing the form properties, constructing a Bird and calling the
AddBird method on the data store with it, we make the test pass.

*handlers/birds.go*

```go
func (b *Birds) AddBird(w http.ResponseWriter, r *http.Request) {
	species := r.FormValue("species")
	description := r.FormValue("description")

	bird := birds.Bird{
		Species:     species,
		Description: description,
	}
	b.dataStore.AddBird(bird)
	w.WriteHeader(http.StatusCreated)
}
```

There are a couple of error cases which we should test for here. First, the
submitted data might be imcomplete. We should require that species is a
non-empty string. Description can be optional.  Secondly, the data store might
return an error when adding the bird. For example if there is a database
connection problem.  So the error return value from `dataStore.AddBird()`
should be checked.

Here are the new tests.

*handlers/handlers_test.go*

```go
	It("returns error code 400 and does not call the data store when species is not sent", func() {
		data := url.Values{}
		data.Add("description", "foo")

		request := httptest.NewRequest("POST", "/birds", strings.NewReader(data.Encode()))
		request.Header.Add("Content-Type", "application/x-www-form-urlencoded")
		recorder := httptest.NewRecorder()
		handler.AddBird(recorder, request)

		Expect(dataStore.AddBirdCallCount()).To(Equal(0))
		Expect(recorder.Code).To(Equal(http.StatusBadRequest))
	})

	It("returns a server internal error if the data store returns an error", func() {
		data := url.Values{}
		data.Add("species", "foo")
		data.Add("description", "bar")

		request := httptest.NewRequest("POST", "/birds", strings.NewReader(data.Encode()))
		request.Header.Add("Content-Type", "application/x-www-form-urlencoded")
		recorder := httptest.NewRecorder()
		dataStore.AddBirdReturns(errors.New("cannot connect"))
		handler.AddBird(recorder, request)

		Expect(dataStore.AddBirdCallCount()).To(Equal(1))
		Expect(recorder.Code).To(Equal(http.StatusInternalServerError))
	})
```

Here is the handler code.

*handlers/birds.go*

```go
// ...

func (b *Birds) AddBird(w http.ResponseWriter, r *http.Request) {
	species := r.FormValue("species")
	description := r.FormValue("description")

	if species == "" {
		w.WriteHeader(http.StatusBadRequest)
		return
	}

	bird := birds.Bird{
		Species:     species,
		Description: description,
	}
	err := b.dataStore.AddBird(bird)
	if err != nil {
		w.WriteHeader(http.StatusInternalServerError)
		return
	}

	w.WriteHeader(http.StatusCreated)
}
```

The final component to unit test is the data store. As mentioned before, we are
faking the data store in the integration tests, so they should now pass without
any work on the data store. If you run them now, you can verify they do pass.
With some work on the in-memory data store we should be able to make the system
test pass too, so let's do that now.

*data/data_test.go*

```go
	// ...

	Context("AddBird", func() {
		It("adds a valid bird to the list store without error", func() {
			bird := birds.Bird{
				Species:     "foo",
				Description: "bar",
			}
			err := store.AddBird(bird)
			Expect(err).NotTo(HaveOccurred())

			actualBirds, err := store.GetBirds()
			Expect(err).NotTo(HaveOccurred())
			Expect(actualBirds).To(Equal([]birds.Bird{bird}))
		})
	})
```

Here is the code to make this pass.

*data/store.go*

```go
// ...

func (s *Store) AddBird(bird birds.Bird) error {
	s.birds = append(s.birds, bird)
	return nil
}
```

For completeness, we should try adding more than one bird and checking the
state of the bird list.

*data/data_test.go*

```go
	// ...

	Context("AddBird", func() {
		It("adds a valid bird to the list store without error", func() {
			bird := birds.Bird{
				Species:     "foo",
				Description: "bar",
			}
			anotherBird := birds.Bird{
				Species:     "jim",
				Description: "bob",
			}
			err := store.AddBird(bird)
			Expect(err).NotTo(HaveOccurred())

			actualBirds, err := store.GetBirds()
			Expect(err).NotTo(HaveOccurred())
			Expect(actualBirds).To(Equal([]birds.Bird{bird}))

			err = store.AddBird(anotherBird)
			Expect(err).NotTo(HaveOccurred())
			actualBirds, err = store.GetBirds()
			Expect(err).NotTo(HaveOccurred())
			Expect(actualBirds).To(Equal([]birds.Bird{bird, anotherBird}))
		})
	})
```

That passes without modification, but we at least avoid the error of
implementing AddBird like `s.birds = []birds.Bird{bird}`.

So now all our unit and integration tests are passing. We should focus again on
the system test.  This is running the `cmd/api` package which we've not touched
since the beginning. This will need to create and inject our discovered
components into the main command - code which is currently missing.

*cmd/api/main.go*

```go
package main

import (
	"net/http"

	"github.com/kieron-pivotal/birds/data"
	"github.com/kieron-pivotal/birds/handlers"
	"github.com/kieron-pivotal/birds/routing"
)

func main() {
	store := data.NewStore()
	handler := handlers.NewHandler(store)
	router := routing.NewRouter(handler)

	http.ListenAndServe(":8080", router)
}
```

Now we can run the system test. Unfortunately, we seem to have missed the
Content-Type of the GetBirds response.  We should be checking this in the
integration test, so let's add it.

*integration_tests/api/birds_test.go*

```go
	// ...

	It("returns correct list of birds from GET request to /birds endpoint", func() {
		birdList := []birds.Bird{
			{Species: "housemartin", Description: "member of eighties band"},
			{Species: "eagle", Description: "another musical bird"},
		}
		dataStore.GetBirdsReturns(birdList, nil)

		resp, err := http.Get(server.URL + "/birds")
		Expect(err).NotTo(HaveOccurred())
		Expect(resp.StatusCode).To(Equal(http.StatusOK))
		Expect(resp.Header.Get("Content-Type")).To(Equal("application/json"))

		defer resp.Body.Close()
		body, err := ioutil.ReadAll(resp.Body)
		Expect(err).NotTo(HaveOccurred())
		Expect(body).To(Equal(toJson(birdList)))
	})
```

This should be dealt with in the handler, so let's add this to a unit test
there.

*handlers/handlers_test.go*

```go
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
		Expect(recorder.Header().Get("Content-Type")).To(Equal("application/json"))
	})
```

Then setting the header in the handler code makes the unit and integration test
pass.

*handlers/birds.go*

```go
func (b *Birds) GetBirds(w http.ResponseWriter, r *http.Request) {
	birds, err := b.dataStore.GetBirds()
	if err != nil {
		w.WriteHeader(http.StatusInternalServerError)
		return
	}
	birdsJSON, _ := json.Marshal(birds)
	w.Header().Set("Content-Type", "application/json")
	w.Write(birdsJSON)
}
```

Now the system test is failing because of a property ordering problem in the
JSON output.  The system test was written before we had the `birds.Bird`
struct.  We can change the test to use this struct to generate the expected
JSON, then it passes!

*system_tests/api/birds_test.go*

```go
		// ...

		testBirds := []birds.Bird{
			{Species: "housemartin", Description: "member of eighties band"},
			{Species: "eagle", Description: "another musical bird"},
		}

		for _, bird := range testBirds {
			data := url.Values{}
			data.Add("species", bird.Species)
			data.Add("description", bird.Description)
			resp, err = http.PostForm(birdsEndpoint, data)
			Expect(err).NotTo(HaveOccurred())
			Expect(resp.StatusCode).To(Equal(http.StatusCreated))
		}

		// ...
```

We're now done with the API side of the application. We used a high level
system test to drive out more detailed integration tests. To make these pass we
had to invent components to deal with each part of the work-flow, and we used
interfaces to shape the connected components, and to help with fakes, isolating
the functionality tested in the unit tests to a single component.

We didn't drive out a postgres data store. We'll look at how to drive that out
later. In the meantime, enjoy the fact that we haven't typed `go run
cmd/server/main.go` at all during this whole process, yet we are very confident
that it behaves as required.
