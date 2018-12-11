# Docker Image Deployment Automation with Controlled Tags In Circleci

DIDACTIC is a simple framework for automating the deployment of
Docker images in CircleCI.

Because CircleCI API support for "Project Build Processing" is rather
limited, DIDACTIC uses release tags (accessible via the `$CIRCLE_TAG`
environment variable) to control the deployment of a software product
to target environments. Specifically, release tags are assumed to be
*semantic version* ([semver](https://semver.org/)) strings in which
the components have a specific interpretation:

  - The leading *major*.*minor*.*patch* component specifies the source
    version.
  - The optional prerelease component identifies the deployment
    environment; the lack of a prerelease component implies the
    production environment.
  - The optional metadata component is used as an "image qualifier";
    multiple images for the same source version can exist as long as they
    have different image qualifiers. For example, images that are
    built automatically when a new base image is published have a
    qualifier of the form `auto.`*timestamp*.

CircleCI workflows that use DIDACTIC can be quite simple. There is an
initialization step, a build/push step, and a deploy step. See the
`bin/didactic` script for details. Refer to `sample-config.yml` for a
sample CircleCI configuration file.


