<!--- vim: set ts=2 sw=2: --->
# action-go

## Philosophy
The goal is to build a few foundational Github Actions to make it easy to build
and test _large_ Go programs efficiently and cost effectively in Github Actions.

The problems we set out to solve are often much less pronounced in smaller
Go projects. Such projects are likely best suited for an `actions/setup-go`
based workflow that uses its built-in caching and executing builds and test
on a small number of runners independently.

We'll have three smaller actions that are callable and chainable in workflows:

- `build`: Build Go programs
- `test`: Test Go programs
- `analyze`: Merge data from parallel partitioned test workflows, summarize it, store it, analyze it

### Constraints
Github Actions has specific limitations that can make testing and building large
Go programs quite a pain. Namely:

- 10gb max of cache per repository. The module and build cache can be enormous
  when you factor cross compiling for many targets and supporting many active
  branches
- Parallel workflow jobs cannot share state and there's no graceful method of
  aggregation of shared data
- There is no shared 'status' or 'conclusion' for multiple parallel jobs

Testing large Go programs in Github Actions comes with its own set of challenges:

- Test patitioning to parallelize test execution over multiple machines is not
  supported with the standard go toolchain.
- Downloading large bundles of modules and/or build cache is slow
- Compiling the same dependencies over and over again is wasteful
- It's not easy to purge byte cache for old deps when using the a previously
  restored cache

### Strategy

Our strategy is to utilize the fastest mechanisms we can and to try and reduce
doing unnecessary work.

As Go modules will need to be present in order to build the program they'll,
always need to be present. We'll use what precious little cache space we have
for mod download cache.

As of Go 1.24, we can utilize a remote build cache for already built objects.
Instead of compiling the same dependencies over and over we'll instead download
the pre-built objects.

For most pull requests, much of our time will be spent downloading the smallest
module cache possible, compiling only changed packages and relinking the program.

For testing, we'll partition the tests based on historical execution timing data
and execute them in parallel on multiple runners.

As runners cannot share state, as there is no shared `status` or `conclusion`
on matrix jobs, and because `needs:` is not a reliable way to determine such,
we need a separate job that executes after the parallel test execution. This job
is responsible for:
  - aggregating test logs
  - generating an overall summary
  - data race identification and data race summary generation
  - aggregation, sanitization and caching of new timing data
  - determining an overall status for all of the jobs

#### Go Modules
We'll go for a strategy where we minimally save Go modules in Github Actions
cache. (Later we could also use external storage)

Rather than the full expanded module cache, we'll target the download cache
directory: `$(go mod GOMODCACHE)/cache`. Go will then be responsible for
unzipping them in parallel instead of a singular tar extract.

#### Go build cache
Go's default build cache is filesystem based. This makes a lot of sense for a
developer machine. On Github Actions, where we split builds and tests across
many runners, to take advantage of such a filesystem cache we have to store the
entire cache is some remote location and download it all on each machine to
take advantage of the build cache. This works well enough for small programs
but for large programs with many build targest we simply don't have the cache
space to keep both the modules and build cache in the actions cache.

Instead of using a filesystem cache we'll instead use a remote S3 bucket as a
storage point for our build cache via [GOCACHEPROG](https://pkg.go.dev/cmd/go/internal/cacheprog)
Since our action will be native to Github Actions we'll write this wrapper proxy
in Typescript, transpile it during build to Javascript, and spawn it using the
same node runtime that executes the action. This will require no additional
external dependencies. The node implementation of the proxy will likely be I/O
bound and thus the runtime choice shouldn't matter much.

Over time the build cache will become stale as we cycle through new code, new
Go compiler versions, and new dependencies. We could create a complicated system
where we turn on S3 access logs which we then analyze for purge objects by on
last access time. For now, we'll try a much simpler purge strategy that uses
automatic experation lifecycle configuration on the bucket where we delete
all objects after X days. The ideal setting will likely be dependent on the
repository and its unique development patterns.

```xml
<LifecycleConfiguration>
  <Rule>
    <Filter>
       <Prefix>actions-go-cache/</Prefix>
    </Filter>
    <Status>Enabled</Status>
    <Expiration>
      <Days>14</Days>
    </Expiration>
  </Rule>
</LifecycleConfiguration>
```

#### Test partitioning
Test paritioning is not supported natively via `go test` or any other standard
tool in the Go toolchain. While `go test` is capable of filtering and targeting
test cases and modules, we need something more robust to handle aggregating the
test timing and partitioning. While we could and perhaps might eventually make
this a native feature of the action, we'll avoid that complexity by using
[`gotestsum`](https://github.com/gotestyourself/gotestsum)
to execute our Go tests. `gotestsum` will allow us to execute tests, track their
timing, create partition sets, and output test results in a Github Actions
native output.

Since it'll be foundational and a requirement, installation of a pre-built
release of `gotestsum` will be built into the `test` action.

`gotestsum` partitioning algorithm is deterministic when given the same input
metadata so we won't advocate for using an extra job to determine a dynamic
matrix. Instead, we'll advocate for a pattern where a matrix range is provided
that results in the total number of test runners:
```yaml
strategy:
  fail-fast: false
  matrix:
    partition: [1, 2, 3, 4, 5, 6, 7, 8, 9, 10]
    max-partitions: 10
```

We'll need to allow flexibility in targeting the correct tests so we'll expose
enough that'll allow users to provide a command. This ought to be flexible
enough for env vars, tags, modules, etc.

## Usage

### Go Build
To build a Go program, run:

```yaml
steps:
  - uses: actions/setup-go@5
  - uses: ryancragun/action-go/build@v1
```

To take advantage of smart module caching and use remote build cache from S3, run:

```yaml
steps:
  - uses: actions/setup-go@5
    with:
      cache: false
  - uses: aws-actions/configure-aws-credentials@v4
    with:
      aws-region: us-east-1
      role-to-assume: my-cache-bucket-access-role
  - uses: ryancragun/action-go/build@v1
    with:
      cache-prog-s3-region: us-east-1
      cache-prog-s3-bucket: ryancragun-go-cache-prog
      cache-prog-s3-key: ${{ github.repository }}
```

If you'd like more control, you can set more parameters:
```yaml
steps:
  - uses: ryancragun/action-go/build@v1
    with:
      cmd: GOEXPERIMENT=boringcrypto go build -tags foo,bar ./... -o prog
      cache-prog-s3-region: us-east-1
      cache-prog-s3-bucket: ryancragun-go-cache-prog
      cache-prog-s3-key: ${{ github.repository }}
      module-cache-path: ~/go/pkg/mod/cache
      module-cache-key: hashFiles('**/go.sum')
```

### Go Test
To execute Go tests, run:

```yaml
steps:
  - uses: actions/setup-go@5
  - uses: ryancragun/action-go/test@v1
```

To take advantage of smart module caching, test splitting across multiple runner
partitions, and use remote build cache from S3, run:

```yaml
jobs:
  test:
    runs-on: ubuntu-latest
    strategy:
      fail-fast: false
      matrix:
        partition: [1, 2, 3, 4, 5, 6, 7, 8, 9, 10]
        max-partitions: 10
    steps:
      - uses: actions/setup-go@5
        with:
          cache: false
      - uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-region: us-east-1
          role-to-assume: my-cache-bucket-access-role
      - uses: ryancragun/action-go/test@v1
        with:
          partition: ${{ matrix.partition }}
          max-partitions: ${{ matrix.max-partitions }}
          cache-prog-s3-region: us-east-1
          cache-prog-s3-bucket: ryancragun-go-cache-prog

  analyze:
    runs-on: ubuntu-latest
    needs: test
    steps:
      - uses: actions/setup-go@5
        with:
          cache: false
      - uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-region: us-east-1
          role-to-assume: my-cache-bucket-access-role
      - uses: ryancragun/action-go/analyze@v1
        # Handles aggregating logs, generating summaries, sanitization and
        # caching of all test job timing files, determining success/fail.
```

If you'd like more control, you can set more parameters:
```yaml
steps:
  - uses: ryancragun/action-go/test@v1
    with:
      env:
        GOEXPERIMENT: boringcrypto
      cmd: go test -tags foo,bar ./...
      cache-prog-s3-region: us-east-1
      cache-prog-s3-bucket: ryancragun-go-cache-prog
      cache-prog-s3-key: ${{ github.repository }}
      gotestsum-version: v1.12.0
      max-partitions: ${{ matrix.max-partitions }}
      module-cache-path: ~/go/pkg/mod/cache
      module-cache-key: hashFiles('**/go.sum')
      partition: ${{ matrix.partition }}
      rerun-fails: true
      test-timing-cache-key: action-go-test-timing
      test-timing-cache-enabled: false
      test-timing-restore-key: action-go-test-timing
      # likely more for data-race logs, etc...
```
