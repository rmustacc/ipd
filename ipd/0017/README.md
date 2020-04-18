---
authors: Robert Mustacchi <rm@fingolfin.org>
state: published
---

# IPD 17 SMF Runtime Directory Creation Support

When writing a daemon, a common problem is that one needs to create a
directory for that daemon, such as `/var/run/<daemon-specific dir>` as
part of the daemon starting up. However, often the permissions on
`/var/run` or on other directories are such that a daemon needs to do
that with root privileges. This makes the ability to specify a user,
group, and privileges in an SMF manifest file, less valuable.

Other software in this space, like systemd, has added functionality so
that a service can have a number of [directories to be created
automatically](https://www.freedesktop.org/software/systemd/man/systemd.exec.html#RuntimeDirectory=).
This has proven useful and folks have asked for similar functionality in
illumos.

This IPD proposes extending SMF to support such a mechanism and make
this a first class feature that's supported and usable when writing
SMF manifests on illumos.

## Goals

This facility should provide the following:

1. The ability to freely specify the path that should be created.
2. The ability to override the default user, group, and directory
permissions.
3. A way for a program to refer to this, such as through the
environment.
4. The ability to try and make sure that directory is empty on start.

It's worth taking these goals apart a little bit and contrasting them
with other implementations. While the systemd implementation constrains
the general location and provides semantics for specific classes of
directories, in our case, we'd rather allow this to be flexible. People
running illumos often have software that comes from many different
prefixes in the system, whether that be `/usr`, `/opt`, or something
that they use for themselves. As a result, it seems like allowing
flexibility here, which mimics existing use, is important.

With the second goal, because not all services start up with the user
and group specified in their start method, but instead will change their
user and group ids, allowing the directories to match that of the end
state of the process is useful. As this can be used to create arbitrary
directories, there may be cases where one needs to create a specific
directory that's owned by root or to make sure a sticky bit is present.

The third option is useful for software where such options are
over ridable in the environment. This allows for less logic to have to be
placed into a start method which wraps this.

Finally, the fourth goal is for software which often has pid files or is
expecting the directory that it receives to be empty. Please see the
Design Considerations section further on for more on this and the trade
offs.

## SMF Details

Concretely, I propose we add a new section to a service manifest that
can be specified at both a service and an instance level. This entry
will be called `managed_paths` and will allow one to specify one or more
path entries. Each path entry will have the following fields:

| Field | Required | Default | Description |
|-------|----------|---------|-------------|
| path | yes | N/A | The directory path to create |
| user | no | `method_credential.user` | user to own the directory |
| group | no | `method_credential.group` | group to own the directory |
| mode | no | 0770 | Permissions for the directory |
| env | no | "" | Environment variable to export `path` in |
| empty | no | false | Ensure the target directory is empty on start up |

Semantically, this operation will only occur when a service transitions
to being **online**. Generally, that means before a `start` method is
invoked by `svc.startd`. However, if we had support for other types of
services and restarters such as a periodic one (like exists in Solaris
for cron-like activity), then that restarter would also theoretically be
able to honor this and make sure that they existed.

When this process occurs, the system will take the following steps for
each entry in `managed_paths`:

1. It will make sure that the directory `path` is created and any
directories leading up to it. `path` is subject to the standard SMF
token expansion. This allows for folks to embed the instance and service
name into it, or a property, much like is done for the start method.

2. For any intermediate directory that does not exist, the system will
create it such that it is owned by the user `root`, the group `root`,
and have the permissions `0755`. If a directory already exists, the
system will not change the permissions or ownership on it. For example,
if `/var/run/foobar` were the directory, then it would not change
anything on `/var` or `/var/run`.

3. It will make sure that the user and group on the directory is set to
the one specified. If the optional `user` and `group` properties are
missing, then it will use the one from the `method_credential`.
Similarly, it will set the permissions on the directory to `mode`. An
important consideration here is what happens if this directory already
exists. In that case, the system will still change the permissions and
ownership.  See the Design Considerations section for more on this
choice of behavior.

4. If `env` value set to a non-empty string, then the resulting path
that was created will end up set in the environment variable specified.
The empty string `""` is a synonym for not adding this to the
environment. When the environment variable already exists, then the
system will create a new value by appending the generated value,
separated by a `:`. This allows directories to be added to the PATH, or
allow multiple directories to be added to the same service-specific
name.

5. If the `empty` value is set to true, then after creating the
directory, we will attempt to recursively remove all contents inside of
it, if the directory already existed. This exists for services that may
use a PID file or similar construct. This is a best effort, meaning that
anything that was present before the emptying process began will be
removed, but if something is added by some other entity while cleaning,
then it is not guaranteed to be removed. During this process, symlinks
will explicitly not be followed. Please see the Design Considerations
section for more discussion on the rationale for this.

### Ordering

The set of entries in `managed_paths` is inherently ordered based on the
order of the entries in the service manifest. The `managed_paths`
section can contain multiple entries, these are processed in order. This
allows dependent directories to be created with the expected
permissions. For example, this allows someone to specify first
`/var/run/myservice` and then follow it with `/var/run/myservice/data`.

If there is more than one `managed_paths` section in a service, then
they will be concatenated together. All **service** level
`managed_paths` will be processed before **instance** level
`managed_paths`.

When entries are added to the environment, the starting environment will
be that as described by the `environment` section of the Method Context
(see [`smf_method(5)`](https://illumos.org/man/5/smf_method). This means
that when checking if an environment variable exists for appending, then
anything described there such as `PATH` will already exist.

## Design Considerations

This section of the IPD goes through some of the rationale behind why we
chose some of the design decisions that we made, contrasts them with the
choices made by systemd, and discusses various trade offs and open
questions in the design. 

### Full Paths vs. Logical Directories

One of the first considerations that we had was the use of full paths
and a more generic facility versus logical service-level needs. For
example, systemd has specific options that cover creating a
configuration directory, a run-time directory, logs, etc. As a result,
it hardcodes the prefix of resulting locations for those options.

A degree of uniformity in the system is important for users and
administrators. When all configuration, logs, etc. are in the same
location this makes the system more approachable and usable. However,
the question is should the service framework enforce this or not.

If we survey many illumos distributions, they do not use a single prefix
for delivering software. Different bundles of software, deliver into a
different prefix and have different preferences for how this is
configured. For example:

* `pkgsrc` delivers software into `/opt/local` including binaries and
configuration, but uses `/var` for run-time information such as the
`pkgin` database, and services.

* The `ooce` package set takes a different approach. While most data is
installed under `/opt/ooce`, configuration lives in `/etc/opt/ooce`, and
various software in `/var` is found with the `/opt/ooce` directory
appended to collect it all.

* The core OpenIndian, OmniOS, and SFE repositories look more
use `/usr`, `/var`, and `/etc` in a similar way. Each owning parts of
that corresponding name space.

Based on all these differences, it seems that there is value in allowing
illumos distributions to continue to deliver service manifests in a way
that is specific to their environment and their druthers. While illumos
does have recommended locations for all of the different pieces, there
are multiple ways to integrate packages and there's value in retaining
this flexibility. If we picked something, at least some of the above
would have to change and be frustrated by that. In general, I believe
that SMF should allow one to implement their desired policy, but not
force it.

### Environment Appending vs. Clobbering

In our current design, we always source the SMF 

XXX Allowing Full Paths
XXX Chaning Perms, existing data
XXX Environment Appending vs. Clobbering
XXX Cleaning
XXX Empty start methods (:kill, :true)
