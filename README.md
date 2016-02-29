External Workspace Jenkins Plugin
====

This plugin computes a workspace path on a file system external
to Jenkins, such as NFS. The workspace path can be reused and shared
between different jobs and builds.

# Basic usage

First the path for the initial upstream build is computed:

```
def workspace = externalWorkspace root: '/root/path', id: prNumber
```

Then the path is set when allocating the workspace for the node:

```
node {
    ws (workspace.path) {
        // build something on /root/path/$JOB_NAME/$id/$BUILD_NUMBER
    }
}
```

The plugin automatically adds the job name and the build number
to the workspace path, resulting in the folowing workspace path:
`/root/path/$JOB_NAME/$id/$BUILD_NUMBER`. If the Cloudbees Folder plugin
is in use, the `$JOB_NAME` also includes the folder name(s). Git users
can use the pull request number as the `id`.

The same workspace path can be used in a downstream job, by specifying
the name of the upstream job when the plugin is instantiated:

```
def workspace = externalWorkspace root: '/root/path', id: prNumber,
                    jobName: 'prBuilder'

```
Then set the path when allocating the workspace for the node as shown above.

# Using multiple physical disks

On NFS, you may need to allocate multiple physical disks to hold the
workspaces of all the different jobs, but for convenience, you want
to access them using a common path. For this use case, the plugin can
create a symbolic link to the physical disk.

Instead of passing the root path to the plugin, you can pass a
configuration file containing the necessary information:

```
def workspace = externalWorkspace cfgFile: '/path/to/file.config', id: prNumber
```

Here is a configuration file example:

```
// Readable by the Groovy ConfigSlurper
diskpool {
  // The list of physical disks (aka volumes)
  disks = [
    '/nfs/projects/jenkins_disk1',
    '/nfs/projects/jenkins_disk2'
  ]

  // The disk pool is accessed from a symbolic link created from the accessPath
  accessPath = '/nfs/projects/jenkins'

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


