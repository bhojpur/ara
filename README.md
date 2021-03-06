# Bhojpur Ara - Automated Resource Assembly

The `Bhojpur Ara` is a software developer's tool used for automated resource assembly within
the [Bhojpur.NET Platform](https://github.com/bhojpur/platform) ecosystem. It assembles platform
aware custom `applications` and/or `services`. It is an experimental `Docker`/`OCI` image builder.
Its goal is to build independent layers, where a change to one layer does *not* invalidate the
ones sitting "above" it.

## How does it work?

The `Bhojpur Ara` software has three main capabilities.

1. __build independent layer chunks__: in a `Bhojpur Ara` project there's a `chunks/` folder which
contains individual Dockerfiles (e.g. `chunks/something/Dockerfile`). These chunk images are built
independently of each other. All of them share the same base image using a special build argument
`${base}`. The `Bhojpur Ara` can build the base image (built from `base/Dockerfile`), as well as
the chunk images. After each chunk image build, the `Bhojpur Ara` will remove the base image layer
from that image, leaving just the layers that were produced by the chunk Dockerfile.
2. __merge layers into one image__: the `Bhojpur Ara` can merge multiple OCI images/chunks (not just
those built using the `Bhopjur Ara`) by building a new manifest and image config that pulls the
layers/DiffIDs from the individual chunks and the base image they were built from.
3. __run tests against images__: to ensure that an image is capable of what we think it should be -
especially after merging - the Bhojpur Ara supports simple tests and assertions that run against
Docker images.

## Would I want to use this?

No. For example, if you're packing your custom `applications` or `services`, you're probably better
of with a regular `Docker` image build and well established means for optimizing that one (think
multi-stage builds, proper layer ordering).

If however you are building custom images, which consist of a lot of independent `concerns`, i.e.
chunks that can be strictly separated, then this might be for you. For example, if you're building
a custom image that serves as a collection of tools, the layer hierarchy imposed by regular builds
doesn't fit so well.

## Limitations and caveats

- build args are not supported at the moment
- there are virtually no tests covering this so things might just break
- consider this alpha-level software

### Requirements

- Install [Go 1.18.1](https://go.dev/doc/install) programming language

```bash
sudo wget https://go.dev/dl/go1.18.1.linux-amd64.tar.gz
sudo tar -C /usr/local -xzf ~/go1.18.1.linux-amd64.tar.gz
sudo sh -c 'echo "export PATH=$PATH:/usr/local/go/bin" >> /etc/profile'
```

Now, logout from the terminal session and login again

```bash
go version

```

Hopefully, you would see something like *go version go1.18 linux/amd64*

- Install and run `Docker` [BuildKit](https://github.com/moby/buildkit/releases) - currently 0.9.3 -
in the background. It helps in building software `applications` using Docker container runtime.

Pull and run a `Docker` registry.

*NOTE*: if you are running it in the [Bhojpur.NET Platform](https://github.com/bhojpur/platform),
this is done for you already!

```bash
sudo su -c "cd /usr; curl -L https://github.com/moby/buildkit/releases/download/v0.9.3/buildkit-v0.9.3.linux-amd64.tar.gz | tar xvz"
$ docker run -p 5000:5000 --name registry --rm registry:2
```

## Build the Bhojpur Ara suitable for your environment

Firstly, clone this `Git` repository in a preferred work folder, then issue the following commands

```bash
cd ara
go mod tidy
go get
go build -o bin/ara main.go
go build -o bin/arautl main-util.go
```

If the build was successful, then you should get *ara* executable image in a local folder.

## Getting started

```bash
# start a new project
ara project init

# add your first chunk
ara project init helloworld
echo hello world > chunks/helloworld/hello.txt
echo "COPY hello.txt /" >> chunks/helloworld/Dockerfile 

# add another chunk, just for fun
ara project init anotherchunk
echo some other chunk > chunks/anotherchunk/message.txt
echo "COPY message.txt /" >> chunks/anotherchunk/Dockerfile

# register a combination which takes in all the chunks
ara project add-combination full helloworld anotherchunk

# build the chunks
ara build us-west2-docker.pkg.dev/some-project/ara-test

# build all combinations
ara combine us-west2-docker.pkg.dev/some-project/ara-test --all

```

## Simple Usage

To learn how to use the `Bhojpur Ara`

### Initialization

```bash
$ ara project init
Starts a new Bhojpur Ara project

Usage:
  ara project init [chunk] [flags]

Flags:
  -h, --help   help for init

Global Flags:
      --addr string      address of buildkitd (default "unix:///run/buildkit/buildkitd.sock")
      --context string   context path (default "/platform/application-images")
  -v, --verbose          enable verbose logging
```

Starts a new `Bhojpur Ara` project. If you don't know where to start, this is the place.

### Building

```bash
$ ara build --help
Builds a Docker image with independent layers

Usage:
  ara build <target-ref> [flags]

Flags:
      --chunked-without-hash   disable hash qualification for chunked image
  -h, --help                   help for build
      --no-cache               disables the buildkit build cache
      --plain-output           produce plain output

Global Flags:
      --addr string      address of buildkitd (default "unix:///run/buildkit/buildkitd.sock")
      --context string   context path (default "/platform/application-images")
  -v, --verbose          enable verbose logging
```

The `Bhojpur Ara` can build regular Docker files much like `docker build` would. `build`
will build all images found under `chunks/`.

The `Bhojpur Ara` cannot reproducibly build layers but can only re-use previously built ones.
To ensure reusable layers and maximize Docker cache hits, Ara itself caches the layers it builds
in a Docker registry.

### Combining

```bash
$ ara combine --help
Combines previously built chunks into a single image

Usage:
  ara combine <target-ref> [flags]

Flags:
      --all                  build all combinations
      --build-ref string     use a different build-ref than the target-ref
      --chunks string        combine a set of chunks - format is name=chk1,chk2,chkN
      --combination string   build a specific combination
  -h, --help                 help for combine
      --no-test              disables the tests

Global Flags:
      --addr string      address of buildkitd (default "unix:///run/buildkit/buildkitd.sock")
      --context string   context path (default "/platform/application-images")
  -v, --verbose          enable verbose logging
```

The `Bhojpur Ara` can combine previously built chunks into a single image. For example:
`ara combine some.registry.com/ara --chunks foo=chunk1,chunk2` will combine `base`, `chunk1`
and `chunk2` into an image called `some.registry.com/ara:foo`.

One can pre-register such chunk combinations using `ara project add-combination`.

The `ara.yaml` file specifies the list of available combinations. Those combinations can also
reference each other:

```yaml
combiner:
  combinations:
  - name: minimal
    chunks:
    - golang
  - name: some-more
    ref:
    - minimal
    chunks:
    - node
```

#### Testing layers and merged images

During a `Bhojpur Ara` build, one can test the individual layers and the final image.

During the build, the `Bhojpur Ara` will execute the layer tests for each individual layer,
as well as the final image. This makes finding and debugging issues created by the layer merge
process tractable.

Each chunk gets its own set of tests found under `tests/chunk.yaml`.

For example:

```yaml
- desc: "it should demonstrate tests"
  command: ["echo", "hello world"]
  assert:
  - status == 0
  - stdout.indexOf("hello") != -1
  - stderr.length == 0
- desc: "it should handle exit codes"
  command: ["sh", "-c", "exit 1"]
  assert:
  - status == 1
- desc: "it should have environment variables"
  command: ["sh", "-c", "echo $MESSAGE"]
  env:
  - MESSAGE=foobar
  assert:
  - stdout.trim() == "foobar"
```

#### Assertions

All test assertions are written in [ES5 Javascript](https://github.com/robertkrimen/otto).
Three variables are available in an assertion:

- `stdout` contains the standard output produced by the command
- `stderr` contains the standard error output produced by the command
- `status` contains the exit code of the command/container.

The assertion itself must evaluate to a boolean value, otherwise the test fails.

#### Testing approach

While the test runner is standalone, the `linux+amd64` version is embedded into the
`Bhojpur Ara` binary using [go.rice](https://github.com/GeertJohan/go.rice) and `go generate` -
see [build.sh](./pkg/test/runner/build.sh).

TODO: use go:embed?

*NOTE* that if you make changes to code in the test runner you will need to re-embed the runner
into the binary in order to use it via `Bhojpur Ara`.

```bash
go generate ./...
```

The test runner binary is extracted and copied to the generated image, where it is run using an
encoded `JSON` version of the test specification - see [container.go](pkg/test/buildkit/container.go).
The exit code, stdout & stderr are captured and returned for evaluation against the assertions in
the test specification.

While of limited practical use, it is *possible* to run the test runner standalone using a
base64-encoded JSON blob as a parameter:

```bash
$ go run pkg/test/runner/main.go eyJEZXNjIjoiaXQgc2hvdWxkIGhhdmUgR28gaW4gdmVyc2lvbiAxLjEzIiwiU2tpcCI6ZmFsc2UsIlVzZXIiOiIiLCJDb21tYW5kIjpbImdvIiwidmVyc2lvbiJdLCJFbnRyeXBvaW50IjpudWxsLCJFbnYiOm51bGwsIkFzc2VydGlvbnMiOlsic3Rkb3V0LmluZGV4T2YoXCJnbzEuMTFcIikgIT0gLTEiXX0=
{"Stdout":"Z28gdmVyc2lvbiBnbzEuMTYuNCBsaW51eC9hbWQ2NAo=","Stderr":"","StatusCode":0}
```

The stdout/err are returned as base64-encoded values.
They can be extracted using `jq` e.g.:

```bash
go run pkg/test/runner/main.go eyJEZXNjIjoiaXQgc2hvdWxkIGhhdmUgR28gaW4gdmVyc2lvbiAxLjEzIiwiU2tpcCI6ZmFsc2UsIlVzZXIiOiIiLCJDb21tYW5kIjpbImdvIiwidmVyc2lvbiJdLCJFbnRyeXBvaW50IjpudWxsLCJFbnYiOm51bGwsIkFzc2VydGlvbnMiOlsic3Rkb3V0LmluZGV4T2YoXCJnbzEuMTFcIikgIT0gLTEiXX0= | jq -r '.Stdout | @base64d'
$ go version go1.18.1 linux/amd64
```

### Integration tests

There is an integration test for the build command in `pkg/ara/build_test.go` -
TestProjectChunk_test_integration and a shell script to run it.

The integration test does an end-to-end check along with editing a test and re-running to
ensure only the test image is updated.

It requires a running `BuildkitD` instance at `unix:///run/buildkit/buildkitd.sock` and a
Docker registry on `127.0.0.1:5000` (i.e. as this workspace is setup on startup).

Override the env vars `BUILDKIT_ADDR` and `TARGET_REF` as required prior to running in a
different environment.

```bash
./integration_tests.sh
```
