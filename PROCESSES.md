# Processes, release procedures etc

This repository includes multiple projects at multiple versions,
some of which depend on each other. That makes things tricky, and
leads to rather more complicated procedures than a simple "single
package" repository. Please read this guide carefully to understand
how releases happen, and why they happen that way.

## The API catalog

Only the source within the `apis` directory is published, and not
all of that. (There are test projects and tools alongside the
production code.) Nothing in the `tools` is published as a package.

Most project files under `apis` are at least partially generated.
The master information is in [apis.json](apis/apis.json) - the API
catalog file. There's an entry for each API, containing:

- The kind of API (grpc, rest, other)
- The version number
- A product title and home page for documentation
- Dependencies
- Additional test project dependencies
- Target framework versions
- Package description for NuGet
- Tags for NuGet (in addition to default ones)

The catalog is used to generate project files and also during the
release process described below. Running the project generator is
very simple: from the root directory, in a bash shell, run

```bash
./generateprojects.sh
```

The CI systems run this before building, to ensure that the project
files are in a stable state.

Generating the project files allows for broad changes (such as
adding Source Link support) to be made very simply, just by changing
the generator. Modifying every project file by hand simply doesn't
scale.

However, sometimes manual editing of project files is required. The
project generator supports this by assuming it "owns":

- The first `PropertyGroup` element (for general properties)
- The first `ItemGroup` element (for dependencies)
- The `ItemGroup` element for Source Link (with a label of "dotnet pack instructions")
- The common import to only attempt to build desktop assemblies on Windows

Any other elements are left as they are - so if you wish to add an
`ItemGroup` such as for file grouping, just add it anywhere after
the generated one, and it should do the right thing.

Additionally, the project generator adds all projects under an API
directory (and project references) to the solution. It will create
project files from scratch as well - so when adding a new package
from autogenerated API sources, the simplest approach is to copy the
source files, modify the API catalog, and run the project generator.
The project files and solution file will be generated and should
immediately be usable.

## Versioning

All releasable packages follow [Semantic Versioning](http://semver.org).
Non-releasable code (tests, tools, analyzers) are not versioned. The
precise meaning of a breaking change (dictating version number
increments) is out of the scope of this document. (Jon is writing a
blog post about this topic and will link to it when it is published.)

The version number in the API catalog (and therefore in project
files) is one of:

- The mostly recently released package version
- The version that's *about* to be released (see below)
- A prerelease with a suffix of 00 to indicate that the *next* release
  should be a first prerelease, typically of a new minor version,
  due to changes that have been merged. This is to avoid accidentally
  including new features in a patch release. For example, having
  released Google.Cloud.Storage.V1 version 2.0.0, new features were
  merged into the `master` branch and the version was immediately
  changed to 2.1.0-alpha00

Typically version numbers should be changed in a commit which does
nothing else, for clarity. Include both the `apis.json` change and
the project file changes that occur when the project generator has
been run, in the same commit.

Dependencies in the API catalog are specified as properties where
the property name is the package name and the value is the version
number. If the version number is left blank, then:

- For GAX or gRPC dependencies, the version number is determined by
  the project generator, to keep these dependencies in sync for all
  packages.
- For projects within the repository, a project reference is used.

If a version number is specified, it's *always* a package reference.
Importantly, project references should only be used between
production packages when both packages will be published together.
This avoids inconsistent builds where a package is built against a
local version which is different from the declared dependency. *This
isn't currently checked, but will be as part of the releasing
process.*

## Releasing

Releasing consists of five steps:

- Updating the version number in GitHub
- Tagging the commit (typically the commit that updated the version
  number)
- Building and testing
- Pushing the package to nuget.org
- Updating the documentation in GitHub (in the `gh-pages` branch)

### Current process

- After the version number is committed, run `tagreleases.sh` from the
  root directory to tag all updated versions.
- Run `buildrelease.sh` manually, specifying one of the API names
  and versions just tagged.
- Go into the `releasebuild/nuget` directory and push
- Make sure you have a clone of the gh-pages branch, copy the
  documentation, commit and push.
  
Issues with this:

- Integration tests are not run automatically
- The build takes a long time as it rebuilds all packages, runs unit
  tests for them all and builds all the documentation
- There's no validation of project dependencies as described above

### Future process

- `tagreleases.sh` will perform validation of project references for
  packages being tagged
- A Google build machine (with appropriate secrets) will listen for
  tags being created
- When a set of tags is created (with some delay to avoid "spotting"
  one tag when more are on the way), a build will be run of just the
  tagged packages
- The integration tests for the tagged packages will be run (as well
  as the regular unit tests)
- The documentation for the tagged packages will be built and pushed, as well as
  the "root" documentation