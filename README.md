# Docker Image Deployment Automation with Controlled Tags In Circleci

DIDACTIC is a framework for automating the deployment of Docker images
in CircleCI.

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

Didactic is meant to be easily integrated into a CircleCi configuration.
Refer to `sample-config.yml` for a sample CircleCI configuration file.

Where a didactic client (CircleCI project build) is concerned, there are
two distinct services accessed through the didactic interface: mapping
specific docker images to release tags, and authorizing the deployment
of images to target environments. In the initial implementation of
didactic, the first service is provided by GitHub \"release assets\",
while the second is provided by a separate GitHub repository
(\"didactic-mgr\") that uses protected branches to manage who is
authorized to promote images and to trigger CircleCI builds when an image
is authorized. Either of these implementations can be replaced over time.

## `didactric.json`

The configuration file for didactic is `didactic.json`. It contains
a `default` object and some set of *project* objects; each *project* object
configures a project that has its source in a GitHub repo and that uses
CircleCI to build and deploy images. When project's build uses didactic,
didactic reads the `default` object, then overlays it will the *project*
object so the project's values override any values in the `default` object.

The following configuration parameters are currently supported:

<dl>
<dt>`"admin_user"`</dt>
<dd>
    A GitHub users with administrative privileges over all didactic repos.
    Ususally set only in the `default` object.
</dd>    
<dt>`"prerelease_environments"`</dt>
<dd>
    A list containing the names of all supported non-production deployment
    environments.
</dd>
<dt>`"production_environment"`</dt>
<dd>
    The name of the production deployment environment.
</dd>
<dt>`"autodeploy"`</dt>
<dd>
    A list containing the names of all environments to which manually
    built images can be deployed without an explicit (manual) authorization
    step. See the TRIGGER_TYPE environment variable.
</dd>
<dt>`"autodeploy_autobuilt"`</dt>
<dd>
    A list containing the names of all environments to which automatically
    built images can be deployed without an explicit (manual) authorization
    step. See the TRIGGER_TYPE environment variable.
</dd>
<dt>`"authorized_users"`</dt>
<dd>
    An object with a list for each deployment environment; the list
    identifies users that can authorize a deployment to the environment.
</dd>
<dt>`"authorized_teams"`</dt>
<dd>
    An object with a list for each deployment environment; the list
    identifies teams that can authorize a deployment to the environment.
</dd>
</dl>

## Environment Variables

Didactic uses a number of environment variables to store state between
CircleCI job steps. More specifically, it stores definitions of
environment variables in a file in a CircleCI workspace, and assumes
that job steps that use didactic will source the file to load the
environment. See `didactic-init` for more details.

The following environment variables are assumed to be set by CircleCI:
<dl>
<dt>`CIRCLE_PROJECT_REPONAME`</dt>
<dd>
    The name of the GitHub repo for the current project.
</dd>
<dt>`CIRCLE_PROJECT_USERNAME`</dt>
<dd>
    The owner of the GitHub repo for the current project.
</dd>
<dt>`CIRCLE_TAG`</dt>
<dd>
    The release tag that triggered the current CircleCI workflow.
    Set by CircleCI in `tag builds`. This must be set, and should be
    a semantic version string.
</dd>
</dd>

The following environment variables should be set by the CircleCI project,
but will be set by `didactic-init` if necessary using `citools/circle-env`.

<dl>
<dt>`INIT_RC`</dt>
<dd>
    The name of the file containing environment variable definitions.
</dd>
<dt>`LOCAL_BIN`</dt>
<dd>
    The directory where additional software should be defined for use by
    CircleCI jobs.
</dd>
<dt>`STATEDIR`</dt>
<dd>
    A workspace subdirectory in which state information can be stored
    for sharing among jobs.
</dd>
<dt>`WORKSPACE`</dt>
<dd>
    The root-level workspace directory.
</dd>
</dl>

The following environment variables can be set by the CircleCI project,
but will take on default values if not set:

<dl>
<dt>`CITOOLS_REPO`</dt>
<dd>
    The URL of the CITOOLS repository.
</dd>
<dt>`CITOOLS_TAG`</dt>
<dd>
    The git tag to use when checking out the \"citools\" repository.
    If not set `master` is used.
</dl>
<dt>`DIDACTIC_TAG`</dt>
<dd>
    The git tag to use when checking out the \"didactic\" repository.
    If not set `master` is used.
</dl>
<dt>`IMAGE_REGISTRY`</dt>
<dd>
    The docker image registry name. If not set, the default dockerhub
    registry is used.
</dd>
<dt>`IMAGE_REPO`</dt>
    The docker image repository name. If not set, `ncar` is used.
</dl>

The following environment variables are used and/or defined by didactic:

<dl>
<dt>`AUTODEPLOY_AUTOBUILT`</dt>
<dd>
    The "autodeploy_autobuilt" configuration parameter for the current
    project and deployment environment.
</dd>
<dt>`DEFAULT_DEPLOY_ENV`</dt>
<dd>
    The default deployment environment. This is the first environment in
    the "prerelease_environments" configuration parameter.
</dd>
<dt>`DEPLOY_ENV`</dt>
<dd>
    The target deployment environment. This is derived from the
    *prerelease* component of the semantic version release tag.
</dd>
<dt>`DEPLOY_TAGS`</dt>
<dd>
    A list of docker tags that should be attached to the target image
    when it is deployed.
</dd>
<dt>`DIDACTIC_CF`</dt>
<dd>
    The name of a derived didactic configuration file for the current
    project/environment; this contains a single object containing the
    values from the `default` object and the project-specific object
    from the global configuration, the first overlaid by the second.
</dd>
<dt>`DIDACTIC_INIT`</dt>
<dd>
    Always set to `true` by `didactic-init`.
</dd>
<dt>`IMAGE_METADATA`</dt>
<dd>
    The name of a local file that holds (or should hold) the metadata
    for the target docker image.
</dd>
<dt>`IMAGE_NAME`</dt>
<dd>
    The name of the target docker image. If not set, 
    `\${CIRCLE_PROJECT_REPONAME}` is used.
</dd>
<dt>`IMAGE_PATH`</dt>
<dd>
    The name of the complete docker image path, including the registry,
    repository, name, and tag.
</dd>
<dt>`IMAGE_QUALIFIER`</dt>
<dd>
    The *metata* component of the semantic version release tag, with `-`
    in place of the leading `+`. This is used to distinguish different images
    that are build from the same source.    
</dd>
<dt>`IMAGE_TAG`</dt>
<dd>
    The *metata* component of the semantic version release tag, with `-`
    in place of the leading `+`. This is used to distinguish different images
    that are build from the same source.    
</dd>
<dt>`PRERELEASE_ENVIRONMENTS`</dt>
<dd>
    The "prerelease-environments" configuration parameter.
</dd>
<dt>`PRODUCTION_ENVIRONMENT`</dt>
<dd>
    The "production-environment" configuration parameter.
</dd>
<dt>`RELEASE_TAG`</dt>
<dd>
    A copy of `CIRCLE_TAG`, set if and only if `\${CIRCLE_TAG}` is a
    valid semantic versino string.
</dd>
<dt>`GIT_REPO`</dt>
<dd>
    Usually equal to `${CIRCLE_PROJECT_USERNAME}/${CIRCLE_PROJECT_REPONAME}`;
    the username and reponame of the target project.
</dd>
<dt>`REPO_IS_DIDACTIC_TARGET`</dt>
<dd>
    Set to `true` if `${GIT_REPO}` matches the name of a project managed by
    didactic.
</dd>
<dt>`SOURCE_VERSION`</dt>
<dd>
    The *major.minor.patch* component of the semantic version release tag.
</dd>
<dt>`TRIGGER_TYPE`</dt>
<dd>
    Set to `automatic` if the CircleCI build was triggered by an automated
    process and not human action; `manual` otherwise.
</dd>
</dl>



