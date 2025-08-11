# Using distroless image for release image

Personal notes about distroless image for containerized software release (not specific to [Google's `distroless` images](https://github.com/GoogleContainerTools/distroless)).

## [What’s a distroless image?](https://www.docker.com/blog/is-your-container-image-really-distroless/)

> A distroless image is a container image with a minimal list of applications [...]. Distroless container images typically:
>
> * No shell
> * No package manager
> * No web client (e.g. `curl` or `wget`)

Fewer components = fewer possible exploits entrypoints, and smaller release images. Security is hard, this is an attempt to balance practicality and security.

## Distroless Images

*Pick image according to your requirements, each image has different configuration that they expected (e.g how to declare its entrypoints)*

| Project                         | Link                                                                | Description                                                                                                                                 |
| ------------------------------- | ------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------- |
| GoogleContainerTools/distroless | [GitHub](https://github.com/GoogleContainerTools/distroless)        | The original Google Distroless project providing minimal container images without package managers or shells, focused on security and size. |
| Chainguard Images               | [GitHub](https://github.com/chainguard-images)                      | Secure, minimal, and continuously updated distroless-style images with SBOMs and provenance metadata.                                       |
| Wolfi OS Distroless             | [GitHub](https://github.com/wolfi-dev)                              | Wolfi-based minimal container images optimized for security and supply chain compliance.                                                    |
| cgr.dev/chainguard              | [Website](https://edu.chainguard.dev/chainguard/chainguard-images/) | Chainguard’s public registry hosting distroless and secure base images ready for production use.                                            |

## Usage

This type of images can be suitable when your release requirements might only involves statically-linked binaries, workflows that are very familiar if you're using compiled-language.

For example, you have Go web services that are expected to run behind reverse proxy / load balancer that handle TLS termination for you, hence you can use `gcr.io/distroless/base-nossl-debian12`

## `Dockerfile` & Multi-stage build

Using multi-stage is a given "requirement" to author `Dockerfile` to build distroless image conveniently. Typically:

* You dedicate a build stage, typically named `build` using language-specific image to perform your builds.
* Pick suitable distroless image for your application stack, and copy over application artifacts from `build` stage.

### Scenario
* Node.js app bundled using [`pnpm deploy`](https://pnpm.io/cli/deploy),
* [`nitro.build` web servers targeting nodejs runtime](https://nitro.build/deploy/runtimes/node)
* Statically-linked Go app

```Dockerfile
# Build stage
FROM golang:1.22 as build

WORKDIR /go/src/app
COPY . .

# Do build related commands
RUN go mod download
RUN go vet -v
RUN go test -v

# Produce statically-linked binary
RUN CGO_ENABLED=0 go build -o /go/bin/app

# Pack it up using distroless image
FROM gcr.io/distroless/static-debian12
# Copy application artifacts from `build` stage
COPY --from=build /go/bin/app /
CMD ["/app"]
```

[Multi-stage builds reference.](https://docs.docker.com/build/building/multi-stage/)
