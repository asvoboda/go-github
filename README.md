# go-github #

[![go-github release (latest SemVer)](https://img.shields.io/github/v/release/google/go-github?sort=semver)](https://github.com/google/go-github/releases)
[![GoDoc](https://img.shields.io/static/v1?label=godoc&message=reference&color=blue)](https://pkg.go.dev/github.com/google/go-github/v39/github)
[![Test Status](https://github.com/google/go-github/workflows/tests/badge.svg)](https://github.com/google/go-github/actions?query=workflow%3Atests)
[![Test Coverage](https://codecov.io/gh/google/go-github/branch/master/graph/badge.svg)](https://codecov.io/gh/google/go-github)
[![Discuss at go-github@googlegroups.com](https://img.shields.io/badge/discuss-go--github%40googlegroups.com-blue.svg)](https://groups.google.com/group/go-github)
[![CII Best Practices](https://bestpractices.coreinfrastructure.org/projects/796/badge)](https://bestpractices.coreinfrastructure.org/projects/796)

go-github is a Go client library for accessing the [GitHub API v3][].

Currently, **go-github requires Go version 1.13 or greater**.  go-github tracks
[Go's version support policy][support-policy].  We do our best not to break
older versions of Go if we don't have to, but due to tooling constraints, we
don't always test older versions.

[support-policy]: https://golang.org/doc/devel/release.html#policy

If you're interested in using the [GraphQL API v4][], the recommended library is
[shurcooL/githubv4][].

## Installation ##

go-github is compatible with modern Go releases in module mode, with Go installed:

```bash
go get github.com/google/go-github/v39
```

will resolve and add the package to the current development module, along with its dependencies.

Alternatively the same can be achieved if you use import in a package:

```go
import "github.com/google/go-github/v39/github"
```

and run `go get` without parameters.

Finally, to use the top-of-trunk version of this repo, use the following command:

```bash
go get github.com/google/go-github/v39@master
```

## Usage ##

```go
import "github.com/google/go-github/v39/github"	// with go modules enabled (GO111MODULE=on or outside GOPATH)
import "github.com/google/go-github/github" // with go modules disabled
```

Construct a new GitHub client, then use the various services on the client to
access different parts of the GitHub API. For example:

```go
client := github.NewClient(nil)

// list all organizations for user "willnorris"
orgs, _, err := client.Organizations.List(context.Background(), "willnorris", nil)
```

Some API methods have optional parameters that can be passed. For example:

```go
client := github.NewClient(nil)

// list public repositories for org "github"
opt := &github.RepositoryListByOrgOptions{Type: "public"}
repos, _, err := client.Repositories.ListByOrg(context.Background(), "github", opt)
```

The services of a client divide the API into logical chunks and correspond to
the structure of the GitHub API documentation at
https://docs.github.com/en/free-pro-team@latest/rest/reference/.

NOTE: Using the [context](https://godoc.org/context) package, one can easily
pass cancelation signals and deadlines to various services of the client for
handling a request. In case there is no context available, then `context.Background()`
can be used as a starting point.

For more sample code snippets, head over to the
[example](https://github.com/google/go-github/tree/master/example) directory.

### Authentication ###

The go-github library does not directly handle authentication. Instead, when
creating a new client, pass an `http.Client` that can handle authentication for
you. The easiest and recommended way to do this is using the [oauth2][]
library, but you can always use any other library that provides an
`http.Client`. If you have an OAuth2 access token (for example, a [personal
API token][]), you can use it with the oauth2 library using:

```go
import "golang.org/x/oauth2"

func main() {
	ctx := context.Background()
	ts := oauth2.StaticTokenSource(
		&oauth2.Token{AccessToken: "... your access token ..."},
	)
	tc := oauth2.NewClient(ctx, ts)

	client := github.NewClient(tc)

	// list all repositories for the authenticated user
	repos, _, err := client.Repositories.List(ctx, "", nil)
}
```

Note that when using an authenticated Client, all calls made by the client will
include the specified OAuth token. Therefore, authenticated clients should
almost never be shared between different users.

See the [oauth2 docs][] for complete instructions on using that library.

For API methods that require HTTP Basic Authentication, use the
[`BasicAuthTransport`](https://godoc.org/github.com/google/go-github/github#BasicAuthTransport).

GitHub Apps authentication can be provided by the [ghinstallation](https://github.com/bradleyfalzon/ghinstallation)
package.

```go
import "github.com/bradleyfalzon/ghinstallation"

func main() {
	// Wrap the shared transport for use with the integration ID 1 authenticating with installation ID 99.
	itr, err := ghinstallation.NewKeyFromFile(http.DefaultTransport, 1, 99, "2016-10-19.private-key.pem")
	if err != nil {
		// Handle error.
	}

	// Use installation transport with client.
	client := github.NewClient(&http.Client{Transport: itr})

	// Use client...
}
```

### Rate Limiting ###

GitHub imposes a rate limit on all API clients. Unauthenticated clients are
limited to 60 requests per hour, while authenticated clients can make up to
5,000 requests per hour. The Search API has a custom rate limit. Unauthenticated
clients are limited to 10 requests per minute, while authenticated clients
can make up to 30 requests per minute. To receive the higher rate limit when
making calls that are not issued on behalf of a user,
use `UnauthenticatedRateLimitedTransport`.

The returned `Response.Rate` value contains the rate limit information
from the most recent API call. If a recent enough response isn't
available, you can use `RateLimits` to fetch the most up-to-date rate
limit data for the client.

To detect an API rate limit error, you can check if its type is `*github.RateLimitError`:

```go
repos, _, err := client.Repositories.List(ctx, "", nil)
if _, ok := err.(*github.RateLimitError); ok {
	log.Println("hit rate limit")
}
```

Learn more about GitHub rate limiting at
https://docs.github.com/en/free-pro-team@latest/rest/reference/rate-limit.

### Accepted Status ###

Some endpoints may return a 202 Accepted status code, meaning that the
information required is not yet ready and was scheduled to be gathered on
the GitHub side. Methods known to behave like this are documented specifying
this behavior.

To detect this condition of error, you can check if its type is
`*github.AcceptedError`:

```go
stats, _, err := client.Repositories.ListContributorsStats(ctx, org, repo)
if _, ok := err.(*github.AcceptedError); ok {
	log.Println("scheduled on GitHub side")
}
```

### Conditional Requests ###

The GitHub API has good support for conditional requests which will help
prevent you from burning through your rate limit, as well as help speed up your
application. `go-github` does not handle conditional requests directly, but is
instead designed to work with a caching `http.Transport`. We recommend using
https://github.com/gregjones/httpcache for that.

Learn more about GitHub conditional requests at
https://docs.github.com/en/free-pro-team@latest/rest/overview/resources-in-the-rest-api#conditional-requests.

### Creating and Updating Resources ###

All structs for GitHub resources use pointer values for all non-repeated fields.
This allows distinguishing between unset fields and those set to a zero-value.
Helper functions have been provided to easily create these pointers for string,
bool, and int values. For example:

```go
// create a new private repository named "foo"
repo := &github.Repository{
	Name:    github.String("foo"),
	Private: github.Bool(true),
}
client.Repositories.Create(ctx, "", repo)
```

Users who have worked with protocol buffers should find this pattern familiar.

### Pagination ###

All requests for resource collections (repos, pull requests, issues, etc.)
support pagination. Pagination options are described in the
`github.ListOptions` struct and passed to the list methods directly or as an
embedded type of a more specific list options struct (for example
`github.PullRequestListOptions`). Pages information is available via the
`github.Response` struct.

```go
client := github.NewClient(nil)

opt := &github.RepositoryListByOrgOptions{
	ListOptions: github.ListOptions{PerPage: 10},
}
// get all pages of results
var allRepos []*github.Repository
for {
	repos, resp, err := client.Repositories.ListByOrg(ctx, "github", opt)
	if err != nil {
		return err
	}
	allRepos = append(allRepos, repos...)
	if resp.NextPage == 0 {
		break
	}
	opt.Page = resp.NextPage
}
```

For complete usage of go-github, see the full [package docs][].

[GitHub API v3]: https://docs.github.com/en/rest
[oauth2]: https://github.com/golang/oauth2
[oauth2 docs]: https://godoc.org/golang.org/x/oauth2
[personal API token]: https://github.com/blog/1509-personal-api-tokens
[package docs]: https://pkg.go.dev/github.com/google/go-github/v39/github
[GraphQL API v4]: https://developer.github.com/v4/
[shurcooL/githubv4]: https://github.com/shurcooL/githubv4

### Testing code that uses `go-github`

The repo [migueleliasweb/go-github-mock](https://github.com/migueleliasweb/go-github-mock) provides a way to mock responses. Check the repo for more details.

### Integration Tests ###

You can run integration tests from the `test` directory. See the integration tests [README](test/README.md).

## Contributing ##
I would like to cover the entire GitHub API and contributions are of course always welcome. The
calling pattern is pretty well established, so adding new methods is relatively
straightforward. See [`CONTRIBUTING.md`](CONTRIBUTING.md) for details.

## Versioning ##

In general, go-github follows [semver](https://semver.org/) as closely as we
can for tagging releases of the package. For self-contained libraries, the
application of semantic versioning is relatively straightforward and generally
understood. But because go-github is a client library for the GitHub API, which
itself changes behavior, and because we are typically pretty aggressive about
implementing preview features of the GitHub API, we've adopted the following
versioning policy:

* We increment the **major version** with any incompatible change to
	non-preview functionality, including changes to the exported Go API surface
	or behavior of the API.
* We increment the **minor version** with any backwards-compatible changes to
	functionality, as well as any changes to preview functionality in the GitHub
	API. GitHub makes no guarantee about the stability of preview functionality,
	so neither do we consider it a stable part of the go-github API.
* We increment the **patch version** with any backwards-compatible bug fixes.

Preview functionality may take the form of entire methods or simply additional
data returned from an otherwise non-preview method. Refer to the GitHub API
documentation for details on preview functionality.

## License ##

This library is distributed under the BSD-style license found in the [LICENSE](./LICENSE)
file.
