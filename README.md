Templated Shared Workspace Plugin
====

This plugin implements a templated shared workspace management system.
It facilitates workspace reuse across jobs and builds. It eliminates
the need to copy, archive or move files in some common scenarios such
as Git pull-request delivery pipelines.

This differs from the Jenkins build-in custom workspace feature in the
following ways:

1. It can be used in Pipeline scripts
2. It provides a default workspace unique to each build
3. It allows jobs which are NOT linked, triggered, or in a pipeline, to share or reuse the same workspace
4. The workspace path can be computed with a user configurable template
5. It supports the Cloudbees Folder plugin
6. It supports multiple NFS volumes

Contrary to the [Shared workspace
plugin](https://wiki.jenkins-ci.org/display/JENKINS/Shared+workspace+plugin)
this plugin is only concerned about workspace sharing, and not about
the SCM.

When a workspace has been shared between builds, this plugin adds a note
to that effect to the build result page of the corresponding builds.

# Basic usage

This plugin is initially written for Pipeline jobs, so all examples are
for Pipeline script jobs.

First the plugin is instantiated in a Pipeline script:

```
// upstream job
def workspace = tsWorkspace root: '/root/path'
```

The workspace is computed by calling the `workspace.path` method when
allocating the workspace for the node using the `ws()` step.

```
// upstream job
node {
    ws (workspace.path) {
        // build something on /root/path/$JOB_NAME/$BUILD_NUMBER
    }
}
```

The plugin automatically adds the job name and the build number
to the workspace path, resulting in the folowing workspace path:
`/root/path/$JOB_NAME/$BUILD_NUMBER`. So if the `JOB_NAME`
is `builder` and the `BUILD_NUMBER` is 12, then the workspace is
`/root/path/builder/12`. If the Cloudbees Folder plugin is in use, the
`$JOB_NAME` also includes the folder name(s).

To reuse the same workspace in a downstream job, specify the name of
the upstream job when the plugin is instantiated by the downstream job:

```
// downstream job
def workspace = tsWorkspace root: '/root/path', upstream: 'builder'
```

Then set the path as shown above when allocating the workspace for
the node:

```
// downstream job
node {
    ws (workspace.path) {
        // build something on /root/path/$upstream/$mostRecentUpstreamStableBuildNumber
    }
}
```

When the `upstream` argument is specified, two important things happen
during workspace path evaluation:

1. The name of the upstream job is used instead of the name of the current (downstream) job
2. The most recent upstream stable build number is used instead of the current (downstream) job build number

So when specifying the `upstream` argument, the downstream build uses
the most recent stable upstream workspace. This is the default behaviour.

If the upstream job does not exist, an exception is thrown and the build
fails. If the history of the upstream job does not contain at least one
stable build, an exception is thrown and the build fails.

# Changing the upstream build number

By default, when the `upstream` argument is specified, the most recent
stable upstream build number is used when computing the workspace. This
behavior can be changed by passing a specific build number to the plugin:

```
def workspace = tsWorkspace root: '/root/path', upstream: 'builder', buildNumber: '13'
```

It does not make much sense to hard code a build number value, but a
build parameter value can be passed as well:

```
def workspace = tsWorkspace root: '/root/path', upstream: 'builder', buildNumber: UPSTREAM_BUILD_NUMBER
```

# Template

The plugin uses a template to compute the workspace value. The default
template is:

```
${root}/${jobName}/${buildNumber}
```

By default, the template only contains variables with special
meaning. These special variables are evaluated according to the rules
explained in the next section. The template can also contain user
defined values, environment variables and build paramters, which are
also explained below.

## Template evaluation rules

The `root`, `jobName` and `buildNumber` template elements are evaluated
according to the rules outlined in the following table:

| Element     | Value |
|-------------|-------|
| root        | The root of the workspace. Always supplied by the user. |
| jobName     | The current job name, or the `upstream` job name when specified. |
| buildNumber | The current job number, or the most recent stable `upstream` build number when the `upstream` job is specified |

The template is programmable and can be changed by the user, either
in-line, or using a configuration file (see [Using a configuration
file][]).  Environment variables and user defined variables are also
supported by the template.

## User defined variables in templates

It is possible to specify a custom template like so:

```
String myTemplate = '${root}/${myVar}'
def workspace = tsWorkspace root: '/root/path', template: myTemplate
```

Then you must pass a value for the `myVar` variable when resolving the template value:

```
node {
    ws (workspace.getPath([myVar:12])) {
        // build something in /root/path/12
    }
}
```

## Environment variables in templates

Workspace templates are evaluated in the environment context of the
current build, so you can specify environment variables in the template:

```
String myTemplate = '${root}/${env.VAR}'
def workspace = tsWorkspace root: '/root/path', template: myTemplate
```

The `env.VAR` environment variable must exist by the time the workspace path
value is evaluated. It can be set using the `withEnv` step:

```
node {
    withEnv(['VAR=12']) {
        ws (workspace.path) {
            // build something in /root/path/12
        }
    }
}
```

If the environment variable value cannot be resolved, TBD (possibly fail
the build).

## Build parameters in templates

It is possible to use build parameters in templates:

```
String myTemplate = '${root}/${BUILD_PARAM}'
def workspace = tsWorkspace root: '/root/path', template: myTemplate
```

In this case the plugin automatically resolves the build parameter value
when it computes the workspace:

```
// The job is built with BUILD_PARAM=12
node {
    ws (workspace.path) {
        // build something in /root/path/12
    }
}
```

# Advanced usage: Git pull-request

One way to build a Git pull-request is to create a job that takes in
the pull-request number as a build input parameter, and build this
pull-request in a unique workspace, named after the job name and
pull-request number.

Furthermore, in some environments, previous attempts at building
pull-requests must be preserved for a little while (for debug purposes),
so it is important to use a unique workspace for each attempt at building
the pull-request. Using both the pull-request number and the build number
in the workspace computation makes this possible.

First create a Pipeline job called `prBuilder` with an input parameter
called `PrNum`.

In the Pipeline script, create template for the workspace computation:

```
String myTemplate = '${root}/${jobName}/${prNumber}/${buildNumber}'
```

Then configure the upstream job (the job that builds the pull-request)
to use this template:

```
def workspace = tsWorkspace root: '/root/path', template: myTemplate
```

Lastly, pass the `PrNum` build parameter to the `getPath()` method
when computing the workspace:

```
node {
    ws(workspace.getPath([prNumber:PrNum]) {
        // something is build under /root/path/$JOB_NAME/$prNumber/$BUILD_NUMBER
    }
}
```

Downstream builds can reuse this workspace by specifying the same template,
and by specifying the pull-request number as well:

```
String myTemplate = '${root}/${jobName}/${prNumber}/${buildNumber}'
def workspace = tsWorkspace root: '/root/path', template: myTemplate, upstream: 'prBuilder'
node {
    ws(workspace.getPath([prNumber:PrNum]) {
        // something is build under /root/path/prBuilder/$prNumber/$upstreamStableBuildNumber
    }
}
```

As described in the [Template evaluation rules][], the `${buildNumber}`
template element takes the value of the most recent stable `${jobName}`
job. Further more, when variables are present between the `${jobName}`
template element and the `${buildNumber}` template element, these
variables are used to search the history for builds matching these
variables and values.

The search is linear and goes from the most recent build in history,
all the way to the oldest build in existence, or until a matching build
is bound. A matching build meets three criteria:

1. It is stable
2. It defines build parameters names that match the template element names between `jobName` and `buildNumber`
3. The template element values match the corresponding build parameter values

In other words, it matches the most recent stable build of the same
pull-request.

# Using a global configuration

If populated, the global configuration will be used. In some cases,
the global configuration is shared between multiple systems external to
Jenkins. To support this use case, the global configuration can be stored
outside Jenkins. See the next section for global configuration details.

# Using a global configuration file

A global configuration file can be used to:

1. Set a root path common to multiple jobs
2. Set a template common to multiple jobs
3. Use multiple NFS volumes

The configuration file must be readable by the Groovy ConfigSlurper. To
use the configuration file, use the `configfile` plugin argument:

```
def workspace = tsWorkspace configFile: 'file.config'
```

## Full configuration file

For quick reference, this is a fully populated configuration file:

```
// Readable by the Groovy ConfigSlurper
tsWorkspace {

    diskpool {
        // The disk pool is accessed from a symbolic link created from the root path
        root = '/nfs/projects/jenkins'

        // Specify some physical volumes
        disks = [
            '/nfs/projects/jenkins_disk1',
            '/nfs/projects/jenkins_disk2'
        ]

    }

    // A template for Git pull requests
    template = '${root}/${jobName}/${prNumber}/${buildNumber}'

    // Default template (for reference)
    // tempalte = ${root}/${jobName}/${buildNumber}
}
```

## Set the root path

The root path can be specified in the configuration file:

```
// Readable by the Groovy ConfigSlurper
tsWorkspace {
    diskpool {
        // The disk pool is accessed from a symbolic link created from the root path
        root = '/nfs/projects/jenkins'
    }
}
```

## Use a common template

A configuration file can be used to store a template that applies to
multiple jobs:

```
// Readable by the Groovy ConfigSlurper
tsWorkspace {
    // A template for Git pull requests
    template = '${root}/${jobName}/${prNumber}/${buildNumber}'

    // Default template (for reference)
    // tempalte = ${root}/${jobName}/${buildNumber}
}
```

## Use multiple NFS volumes

On NFS, you may need to allocate multiple physical disks to hold the
workspaces of all the different jobs, but for convenience, you want
to access them using a common path. For this use case, the plugin can
create a symbolic link to the physical disk.

Here is a configuration file example:

```
// Readable by the Groovy ConfigSlurper
tsWorkspace {
    diskpool {
        disks = [
            '/nfs/projects/jenkins_disk1',
            '/nfs/projects/jenkins_disk2'
        ]

        // The disk pool is accessed from a symbolic link created from the root path
        root = '/nfs/projects/jenkins'
    }
}
```

When the plugin first executes with a configuration file, it selects
the physical disk with the mosty available space, and creates a symbolic
link to it:

```
/nfs/projects/jenkins/$LINK -> /nfs/projects/jenkins_disk2/$JOB_NAME

```

If you use the Cloudbees Folder plugin, the `$LINK` is the part of the
`$JOB_NAME` path up to the first slash. So if the Cloudbees Folder name is
`Admin/Test/jobName`, then the symbolic link `$LINK` is to `Admin` only.


