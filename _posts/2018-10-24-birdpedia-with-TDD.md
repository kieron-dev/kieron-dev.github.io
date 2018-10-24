---
title: Step by step TDD of a Golang Web Application - Part 1
---

## Test-driving a Go Web Application

Inspired by the article by Soham Kamani on [how to build a web application in
Go](https://www.sohamkamani.com/blog/2017/09/13/how-to-build-a-web-application-in-golang/),
I decided to build the same application, but using a test-driven-development
(TDD) approach. I wanted to see how this technique influences the design of the
code and its maintainability.

This simple application allows a user to construct a list of birds, consisting
of a bird's species and description, displayed on a web page. The bulk of the
logic is contained in a server component which provides an API for listing and
adding birds using a JSON REST interface. Birds are stored in a postgres
database. The user interface is contained in a single-page web application,
served by the same web server used for the REST API.

### First steps

We'll employ a top-down approach to driving out this application, starting at a
system test. This will start up the REST API server, check that an initial GET
on the /birds endpoint returns an empty JSON list, then add some birds by
sending POST requests to /birds, and finally verify that a GET to /birds
returns a JSON list of the added birds.

We'll be using [ginkgo](https://github.com/onsi/ginkgo) and
[gomega](https://github.com/onsi/gomega) for a BDD testing framework. You'll
need to `go get` these first if you want to code along and don't have them
already. Then we'll let ginkgo create the test file:

```bash
$ mkdir -p system_tests/api
$ (cd system_tests/api; ginkgo bootstrap; ginkgo generate birds)
```

A common ginkgo pattern for system-testing a server is to build the server
application in the test suite, and to provide a path to the generated binary
for the tests to use. So for our first failing test, let's build the
application in package birds/cmd/server and have a simple test to check it can
be run successfully.

*system_tests/api/api_test_suite.go:*

```go
package api_test

import (
	. "github.com/onsi/ginkgo"
	. "github.com/onsi/gomega"
	"github.com/onsi/gomega/gexec"

	"testing"
)

func TestApi(t *testing.T) {
	RegisterFailHandler(Fail)
	RunSpecs(t, "Api Suite")
}

var (
	binPath string
)

var _ = BeforeSuite(func() {
	var err error
	binPath, err = gexec.Build("github.com/kieron-pivotal/birds/cmd/server")
	Expect(err).NotTo(HaveOccurred())
})

var _ = AfterSuite(func() {
	gexec.CleanupBuildArtifacts()
})
```

*system_tests/api/birds_test.go:*

```go
package api_test

import (
	"os/exec"

	. "github.com/onsi/ginkgo"
	. "github.com/onsi/gomega"
	"github.com/onsi/gomega/gexec"
)

var (
	session *gexec.Session
)

var _ = Describe("API Server", func() {

	AfterEach(func() {
		session.Terminate()
	})

	It("starts OK", func() {
		cmd := exec.Command(binPath)
		var err error
		session, err = gexec.Start(cmd, GinkgoWriter, GinkgoWriter)
		Expect(err).NotTo(HaveOccurred())
	})

})
```

We can run the test using `ginkgo system_tests/api`, and we immediately see a
'failed to build' error. Following the test-driven principles, our first action
is to make that test pass in the simplest way possible. Note that this doesn't
mean implementing a web server, as we don't have a test for that yet.

By creating the following file, we satisfy the test.

*cmd/server/main.go:*

```go
package main

func main() {
}
```

Next we can check we are actually starting a web server. So we can modify the
test to check we can connect to localhost on port 8080. We use the `Eventually`
construction here as the web server will not be ready immediately and we
anticipate the requirement for some retries.

*system_tests/api/birds_test.go:*

```go
	// ...

	It("starts a server listening on port 8080", func() {
		cmd := exec.Command(binPath)
		var err error
		session, err = gexec.Start(cmd, GinkgoWriter, GinkgoWriter)
		Expect(err).NotTo(HaveOccurred())

		Eventually(func() bool {
			_, err = net.Dial("tcp", ":8080")
			return err == nil
		}).Should(BeTrue())
	})
```

To make this test pass, we can start a HTTP server in the server application.

*cmd/server/main.go:*

```go
// ...

func main() {
	http.ListenAndServe(":8080", nil)
}
```

Finally, let's write the test we intended to originally. A GET request to the
/birds endpoint should return a JSON empty list. Then a series of POSTs to /birds
should return HTTP 201 responses, and a final GET to /birds should return a
list matching the POSTed data.

We move the server startup code to a BeforeEach function to keep the test
focused on the HTTP calls. The whole test file is given below.

*system_tests/api/birds_test.go:*

```go
package api_test

import (
	"encoding/json"
	"io/ioutil"
	"net"
	"net/http"
	"net/url"
	"os/exec"

	. "github.com/onsi/ginkgo"
	. "github.com/onsi/gomega"
	"github.com/onsi/gomega/gexec"
)

var (
	session       *gexec.Session
	birdsEndpoint = "http://localhost:8080/birds"
)

var _ = Describe("API Server", func() {

	BeforeEach(func() {
		cmd := exec.Command(binPath)
		var err error
		session, err = gexec.Start(cmd, GinkgoWriter, GinkgoWriter)
		Expect(err).NotTo(HaveOccurred())

		Eventually(func() bool {
			_, err = net.Dial("tcp", ":8080")
			return err == nil
		}).Should(BeTrue())
	})

	AfterEach(func() {
		session.Terminate()
	})

	It("can add and list birds using GETs and POSTs to /birds", func() {
		resp, err := http.Get(birdsEndpoint)
		Expect(err).NotTo(HaveOccurred())
		Expect(resp.StatusCode).To(Equal(http.StatusOK))
		Expect(resp.Header.Get("Content-Type")).To(Equal("application/json"))

		defer resp.Body.Close()
		body, err := ioutil.ReadAll(resp.Body)
		Expect(err).NotTo(HaveOccurred())
		Expect(body).To(Equal([]byte("[]")))

		testBirds := []map[string]string{
			{"species": "housemartin", "description": "member of eighties band"},
			{"species": "eagle", "description": "another musical bird"},
		}

		for _, bird := range testBirds {
			data := url.Values{}
			data.Add("species", bird["species"])
			data.Add("description", bird["description"])
			resp, err = http.PostForm(birdsEndpoint, data)
			Expect(err).NotTo(HaveOccurred())
			Expect(resp.StatusCode).To(Equal(http.StatusCreated))
		}

		resp, err = http.Get(birdsEndpoint)
		Expect(err).NotTo(HaveOccurred())
		Expect(resp.StatusCode).To(Equal(http.StatusOK))
		Expect(resp.Header.Get("Content-Type")).To(Equal("application/json"))

		expectedJson, err := json.Marshal(testBirds)
		Expect(err).NotTo(HaveOccurred())

		defer resp.Body.Close()
		body, err = ioutil.ReadAll(resp.Body)
		Expect(err).NotTo(HaveOccurred())
		Expect(body).To(Equal(expectedJson))
	})

})
```

At this point we can't simply make this test pass. We need handlers for GET and
POST requests, and these must interact with some data source. It's time to
leave this test red, and dive down a testing layer to drive out the
implementation. When the lower level tests are green, we expect this test to
also pass.
