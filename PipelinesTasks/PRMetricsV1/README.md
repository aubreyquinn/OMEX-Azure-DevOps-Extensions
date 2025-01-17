# PR Metrics

PR Metrics is an Azure Pipelines task for adding size and test coverage
indicators to the start of each Pull Request title.

It can be downloaded from the Visual Studio Marketplace at
<https://marketplace.visualstudio.com/items?itemName=ms-omex.prmetrics>.

For example, a PR with the title "Adding code" could become either:

- XS:heavy_check_mark: :black_small_square: Adding code
- L:warning: :black_small_square: Adding code

The former would indicate an extra small PR with sufficient test coverage,
whereas the latter would indicate a large PR with insufficient test coverage.

This task helps ensure engineers keep PRs to an appropriate size with
appropriate test coverage, while informing reviewers of the expected time
commitment for a thorough review of the code.

The task will also add a comment to the PR with a detailed breakdown of the
metrics:

> **Metrics for iteration 1**
> :heavy_check_mark: Thanks for keeping your pull request small.
>
> :heavy_check_mark: Thanks for adding tests.
>
> |              | Lines   |
> | ------------ | ------: |
> | Product Code |   100   |
> | Test Code    |    50   |
> | **Subtotal** | **150** |
> | Ignored      |     5   |
> | **Total**    | **155** |

It will furthermore add a comment to indicate that a review of the file may not
be necessary.

> :exclamation: **This file may not need to be reviewed.**

If no PR description is provided, the description will be set to:

> :x: **Please add a description.**

The extension will also add properties to the PR, which can be queried by other
extensions as desired:

- `/PRMetrics.Size`: A string representing the size indicator, e.g. `XS`.
- `/PRMetrics.TestCoverage`: A Boolean value indicating whether the test
  coverage is deemed sufficient. If no test coverage is expected (i.e. if the
  test factor parameter is set to `0.0`), this property will not be present.
- `/PRMetrics.ProductCode`: An integer indicating the number of lines of product
  code added via the PR.
- `/PRMetrics.TestCode`: An integer indicating the number of lines of test
  code added via the PR.
- `/PRMetrics.Subtotal`: An integer indicating the number of lines of product
  and test code added via the PR. This is the sum of `/PRMetrics.ProductCode`
  and `/PRMetrics.TestCode`.
- `/PRMetrics.Ignored`: An integer indicating the number of lines of ignored
  code added via the PR. This includes files ignored through the use of the code
  matching patterns parameter, as well as those files whose extensions resulted
  in their being ignored.
- `/PRMetrics.Total`: An integer indicating the total number of lines of code
  added via the PR. This is the sum of `/PRMetrics.Subtotal` and
  `/PRMetrics.Ignored`.

## Deploying

The task can be used with [Azure Pipelines][azurepipelines].

To deploy the task:

1. Acquire administrator access to the server to which you wish to deploy.
1. Install [tfx-cli][tfxcli] using [npm][npm] via the command-line
   `npm install -g tfx-cli`.
1. Sign in to the server using:

   ```Batchfile
   tfx login --service-url https://<account>.visualstudio.com/DefaultCollection --token <PAT>
   ```

   You can generate a PAT by following the instructions [here][tfxpat]. This
   will only need to be performed the first time you use tfx-cli.
1. Delete any existing copy of the task using
   `tfx build tasks delete --task-id 907d3b28-6b37-4ac7-ac75-9631ee53e512`.
1. Install `nuget.exe` and add it to your `PATH` environment variable so that
   NuGet can invoked from any console window. Instructions on the process can be
   located [here][nugetcli].
1. Replace `%MAJOR%`, `%MINOR%`, and `%PATCH%` with any numeric values in
   [vss-extension.json][vssextensionjson] (1 instance of each) and
   [task/task.json][taskjson] (2 instances of each). Ensure you undo these
   changes before creating a PR, as these values will be updated as part of the
   release task.
1. Build the task locally by running `msbuild` from the PRMetrics directory.
   You may need to use a Visual Studio command prompt for this, as `msbuild` is
   not always added to the default path.
1. If the build is successful, upload the task using
   `tfx build tasks upload --task-path  ..\..\Release\PipelinesTasks\PRMetricsV1\task\`.

## Configuring

The task can be added to a pipeline as detailed [here][addingtask]. Note that
only Windows pipelines are supported as the task is written in PowerShell.

The agent running the task must allow access to the OAuth token. If access is
unavailable, the task will generate a warning.

It is recommended to add a custom run condition of
`and(succeeded(), eq(variables['Build.Reason'], 'PullRequest'))`. If the task is
not run via a pull request, it will generate a warning. Adding this condition
avoids the warning.

It is also recommended to run the task as one of the first operations in your
build, after code check out is complete. Running the task early in the build
process allows for the title to be updated quickly, avoiding the need for
engineers to wait a long time for the title update.

### Parameters

The task parameters are:

- **Base Size:** The maximum number of new lines in a small PR. If left blank,
  a default of `250` will be used.
- **Growth Rate:** The growth rate applied to the base size for calculating the
  size of larger PRs. If left blank, a default of `2.0` will be used. With a
  base size of `250` and a growth rate of `2.0`, `500` new lines would
  constitute a medium PR while `1000` new lines would constitute a large PR.
- **Test Factor:** The lines of test code expected for each line of product
  code. If left blank, a default of `1.5` will be used. This can be set to `0.0`
  in order to skip the reporting of whether the test code coverage is
  sufficient.
- **File Matching Patterns:** [Azure DevOps file matching patterns][globs]
  specifying the files and folders to include. Autogenerated files should
  typically be excluded. Excluded files will contain a comment to inform
  reviewers that they are unlikely to need to review those files. If left
  blank, a default of `**/*` (all files) will be used.
- **Code File Extensions:** Extensions for files containing code, so that
  non-code files can be excluded. If left blank, a default set of file
  extensions will be used, which are listed at
  [the end of this file](#default-code-file-extensions).

### YAML

Sample YAML for adding the task is as follows. The default parameter values are
expected to be appropriate for most repos. Therefore, the `inputs` can typically
be excluded.

```YAML
steps:
- task: ms-omex.prmetrics.prmetrics.PRMetrics@1
  displayName: 'PR Metrics'
  inputs:
    BaseSize: 250
    GrowthRate: 2.0
    TestFactor: 1.5
    FileMatchingPatterns: |
      **/*
      !Ignore.cs
    CodeFileExtensions: |
      cs
      ps1
  continueOnError: true
  condition: and(succeeded(), eq(variables['Build.Reason'], 'PullRequest'))
```

## Implementation

This task is written in PowerShell using the [Azure Pipelines Task SDK][sdk].

It works by querying Git for changes using the command
`git diff --numstat origin/<target>...pull/<pull_request_id>/merge`. Files with
`test` in the file or directory name (irrespective of case) are considered test
files. All other files are considered product code files.

Note that this task is designed to give a quick estimate of the size of a change
and its test coverage. It is not an authoritative metric and should not be
treated as such. For example, the task should not be used to replace
comprehensive and thorough code coverage metrics. Instead, the task should
merely be considered a guideline for influencing optimal PR behavior.

## Testing

This task is tested via unit and integration tests constructed using the
[Pester 5][pester] test framework. The code coverage is currently extremely
high, and a high rate of coverage should be maintained for all changes.
Validating this extension on the server is significantly more time consuming
than validating locally. You may need [PowerShell 7 or later][powershell]
installed due to subtle differences in the behavior of different PowerShell
releases.

The task contains no violations of the [PSScriptAnalyzer][psscriptanalyzer]
rules. This compliance with the PSScriptAnalyzer rules should also be maintained
for all new code.

Test validation and code scanning will be automatically performed whenever a
PR is opened against the `main` branch. These validations must succeed for the
PR to be merged.

## Debugging

To gain greater insight into the cause of failures, you can set the
`system.debug` variable to `true` for your build pipeline. This will output
significant additional debugging information, including a full trace of the
methods called, which can be used for posting bug reports, etc.

If a failure occurs during the task, full debugging information will be
outputted by default irrespective of the value of the `system.debug` variable.

## Default Code File Extensions

The default value for the Code File Extensions parameter outlined
[above](#parameters) is:

```Text
ada
adb
ads
asm
bas
bb
bmx
c
cbl
cbp
cc
clj
cls
cob
cpp
cs
cxx
d
dba
e
efs
egt
el
f
f77
f90
for
frm
frx
fth
ftn
ged
gm6
gmd
gmk
gml
go
h
hpp
hs
hxx
i
inc
java
l
lgt
lisp
m
m4
ml
msqr
n
nb
p
pas
php
php3
php4
php5
phps
phtml
piv
pl
pl1
pli
pm
pol
pp
prg
pro
py
r
rb
red
reds
rkt
rktl
s
scala
sce
sci
scm
sd7
skb
skc
skd
skf
skg
ski
skk
skm
sko
skp
skq
sks
skt
skz
spin
stk
swg
tcl
vb
xpl
xq
xsl
y
```

[azurepipelines]: https://azure.microsoft.com/services/devops/pipelines/
[azureserver]: https://azure.microsoft.com/services/devops/server/
[tfxcli]: https://github.com/Microsoft/tfs-cli
[npm]: https://www.npmjs.com/
[tfxpat]: https://docs.microsoft.com/azure/devops/extend/publish/command-line
[nugetcli]: https://docs.microsoft.com/nuget/install-nuget-client-tools#nugetexe-cli
[vssextensionjson]: vss-extension.json
[taskjson]: task/task.json
[addingtask]: https://docs.microsoft.com/azure/devops/pipelines/customize-pipeline
[globs]: https://docs.microsoft.com/azure/devops/pipelines/tasks/file-matching-patterns
[sdk]: https://github.com/microsoft/azure-pipelines-task-lib
[pester]: https://github.com/pester/Pester
[powershell]: https://github.com/powershell/powershell
[psscriptanalyzer]: https://github.com/PowerShell/PSScriptAnalyzer
