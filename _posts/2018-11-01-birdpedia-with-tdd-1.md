---
title: Step by step TDD of a Golang Web Application - Part 1
---

## Test-driving a Go Web Application

Inspired by the article by Soham Kamani on [how to build a web application in
Go](https://www.sohamkamani.com/blog/2017/09/13/how-to-build-a-web-application-in-golang/),
I decided to build the same application, but using a test-driven-development
(TDD) approach. I wanted to see how this technique influences the design of the
code and its maintainability.

This simple application allows a user to construct a list of birds, each with
the bird's species and description, displayed and managed in a web page. The
requirements are that the page should display the list of previously entered
birds, and provide a form with species and description input fields so the user
can add a new bird.

There are various ways this application could be built, from a traditional web
application with pages built on a web server, to a single page web application
with all the logic contained there. The common part is that the application
will be used on a browser, and that is all we need to know to write our first
test!

### End-to-end test

We start with an end-to-end (e2e) test, which, when it finally passes, shows us
that the user requirements have been met. This type of test uses a fully
deployed application just as a real user would, and checks the visible
information is as expected.

In this case, we want to perform the following:

1. Navigate to the URL of the application
1. Verify that there is a bird table, which is currently empty
1. Insert a bird species and description in the corresponding form fields and submit the form
1. Verify that the entered information now appears in the bird table
1. Add a second bird using the form
1. Verify that the second bird now also appears in the bird table
1. Navigate to the URL of the application again (i.e. reload it)
1. Verify that the two birds appear in the bird table

As this is an exercise in using Go, we'll employ an acceptance testing
framework written for Go: [agouti](https://agouti.org). We'll also use the
[Ginkgo](https://github.com/onsi/ginkgo) BDD framework along with its
[Gomega](https://github.com/onsi/gomega) matchers, so first let's ensure we
have them installed.

```bash
$ go get github.com/sclevine/agouti
$ go get github.com/onsi/ginkgo/ginkgo
$ go get github.com/onsi/gomega
```

Also make sure you have the
[chromedriver](http://chromedriver.chromium.org/downloads) installed.

Then create a project directory to house the code. Mine will be
`$GOPATH/src/github.com/kieron-pivotal/birds`.

We can create the skeleton of the end-to-end test using some ginkgo helpers:

```bash
$ mkdir e2e
$ (cd e2e; ginkgo bootstrap --agouti; ginkgo generate --agouti birds)
```

This will create a test suite file `e2e_suite_test.go`. You should follow the
advice and modify the code to select the appropriate WebDriver. In our case
this will be the `agouti.ChromeDriver()`.

*e2e/e2e_suite_test.go*

```go
package e2e_test

import (
	"os/exec"

	. "github.com/onsi/ginkgo"
	. "github.com/onsi/gomega"
	"github.com/sclevine/agouti"

	"testing"
)

func TestE2e(t *testing.T) {
	RegisterFailHandler(Fail)
	RunSpecs(t, "E2E Suite")
}

var (
	agoutiDriver *agouti.WebDriver
)

var _ = BeforeSuite(func() {
	agoutiDriver = agouti.ChromeDriver()
	Expect(agoutiDriver.Start()).To(Succeed())
})

var _ = AfterSuite(func() {
	Expect(agoutiDriver.Stop()).To(Succeed())
})
```

We'll edit the generated `birds_test.go` to perform the end-to-end testing
described above.

*e2e_tests/acceptance_test.go*

```go
package e2e_test

import (
	. "github.com/onsi/ginkgo"
	. "github.com/onsi/gomega"
	"github.com/sclevine/agouti"
	. "github.com/sclevine/agouti/matchers"
)

var _ = Describe("Birds", func() {
	var page *agouti.Page

	BeforeEach(func() {
		var err error
		page, err = agoutiDriver.NewPage()
		Expect(err).NotTo(HaveOccurred())
	})

	AfterEach(func() {
		Expect(page.Destroy()).To(Succeed())
	})

	It("should add new birds to the table and they persist between refreshes", func() {
		Expect(page.Navigate("http://localhost:8080")).To(Succeed())

		table := page.FindByID("birdsTable")
		Expect(table.Text()).NotTo(ContainSubstring("Magpie"))

		Expect(page.FindByName("species").Fill("Magpie")).To(Succeed())
		Expect(page.FindByName("description").Fill("Black and white")).To(Succeed())
		Expect(page.FindByID("birdForm").Submit()).To(Succeed())

		Expect(table.First("td")).To(HaveText("Magpie"))
		Expect(table.All("td").At(1)).To(HaveText("Black and white"))

		Expect(page.FindByName("species").Fill("Blackbird")).To(Succeed())
		Expect(page.FindByName("description").Fill("Just black")).To(Succeed())
		Expect(page.FindByID("birdForm").Submit()).To(Succeed())

		Expect(table.First("td")).To(HaveText("Magpie"))
		Expect(table.All("td").At(1)).To(HaveText("Black and white"))
		Expect(table.All("td").At(2)).To(HaveText("Blackbird"))
		Expect(table.All("td").At(3)).To(HaveText("Just black"))

		Expect(page.Navigate("http://localhost:8080")).To(Succeed())
		table = page.FindByID("birdsTable")
		Expect(table.First("td")).To(HaveText("Magpie"))
		Expect(table.All("td").At(1)).To(HaveText("Black and white"))
		Expect(table.All("td").At(2)).To(HaveText("Blackbird"))
		Expect(table.All("td").At(3)).To(HaveText("Just black"))
	})

})
```

Here we have made some reasonable assumptions about the web page. 

* It must be served at the root `/` address.
* It must contain a birds table with ID `birdsTable`.
* It must contain a form with ID `birdForm`.
* It must contain form fields `species` and `description`.

As yet we have no application, so the test fails early.

### High-level design

Our end-to-end test is good, but it doesn't drive out any high-level design.
We'll have to come up with that ourselves to progress any further.

This exercise is about writing a web application in Go, so let's assume we have
some sort of REST API server, written in Go, which serves lists of birds, and
allows new birds to be added. The front-end should be a single-page web
application written in Javascript interacting with the API. It can load the
bird list when it initially loads, and can POST a new bird when the bird form
is submitted.

We'll need a web server to serve up the static content for the web application,
which could well co-exist in the API server. Finally, we'll need some form of
data storage for the bird data.  We'll use Postgres for that. The interactions
are shown below.

![sequenceDiagram](/assets/high-level.svg)

### Summary

At this stage, we have an acceptance test which exercises the front-end of our
yet-to-be-built application. Once that passes, we'll have something very close
to our requirements, but we're a long way from that at the moment. We also know
that we need to build a couple of distinct components - the front-end web
application, and the back-end API server / web server.

For no particular reason, let's start with the back-end work and we'll see how
we test-drive that in the next post.
