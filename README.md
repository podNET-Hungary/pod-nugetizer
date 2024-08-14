# pod-nugetizer
An opinionated GitHub action to publish to NuGet. This is intended mainly for publishing our own packages under the [podNET-Hungary org](https://github.com/podNET-Hungary) but source is shared for reference.

`v1` of the action "usually" executes the following steps:
```yml
- uses: actions/checkout@v3
- uses: actions/setup-dotnet@v3
- run: dotnet pack -c debug
- run: dotnet test
- run: dotnet pack -p:Version=${{ inputs.version }} -p:RepositoryCommit=${{ inputs.commit }}
- run: dotnet nuget push ./artifacts/package/release/*.nupkg -k ${{ inputs.nuget-push-api-key }} -s https://api.nuget.org/v3/index.json
- uses: actions/upload-artifact@v4
  with: 
    path: ./artifacts/package/release/*.*nupkg
    if-no-files-found: error
- uses: softprops/action-gh-release@v2
  with:
    files: ./artifacts/package/release/*.*nupkg
```

## Usage

> [!IMPORTANT]
> The simplicity of the action itself lends to it being easier to simply just copy and modify the code or fork it to better suit your needs,
> especially if you only want to use the action in one or a handful of repositories. If you want reusable parts, you can implement your own composite or
> custom action similarly to this one or use a [reusable workflow](https://docs.github.com/en/actions/using-workflows/).

Basic usage is most simple:

```yml
on: { push: { tags: ["v[0-9]+.[0-9]+.[0-9]+*"] } }
    
jobs:
  publish:
    runs-on: ubuntu-latest
    steps:
      - uses: podNET-Hungary/pod-nugetizer@v1
        with:
          nuget-push-api-key: ${{ secrets.NUGET_API_KEY }}
```

> [!NOTE]
> All of the steps are optional as they are conditioned on their respective input parameters, but by default, all of them will run **except for the NuGet push** command if you supply no parameters, as the `dotnet nuget push` command requires the `nuget-push-api-key` to be set (it isn't set by default, you have to supply the key by yourself; [always set up your secrets properly](https://docs.github.com/en/actions/security-for-github-actions/security-guides/using-secrets-in-github-actions)).

Inputs as of v1:
```yml
  skip-auto-checkout: { }
  skip-auto-setup-dotnet: { }
  skip-prepack-debug: { }
  skip-test: { }
  skip-pack: { }
  version: { default: "$(echo ${{ github.ref_name }} | sed 's/^v//')" }
  commit: { default: ${{ github.sha }} }
  nuget-push-api-key: { }
  skip-store-artifact: { }
  skip-store-release: { }
```

So for example, calling the action like so:
```yml
- uses: podNET-Hungary/pod-nugetizer@v1
  with: { skip-prepack-debug: true }
```
will result in the `dotnet pack -c debug` step being skipped **and** the `dotnet nuget push` command being skipped because of the API key is not being supplied.

For an up-to-date list of all inputs, [take a look at the source](action.yml).

## Workflow

There is also a workflow ([source](.github/workflows/my-workflow.yml)) that mirrors the action's behavior with a few differences:
- The dependant workflow can *inherit* the secrets of the caller if the caller sets secrets: inherit in their workflow.
- The workflow defines the full job, not just the steps. This includes it defining the working environment (`runs-on`).
- The workflow's job is just a job like any other with individually defined steps. Contrast this with the `composite` action, which seems like a single step in the dependant's runs. So using the workflow will list all steps individually, while using the action will only list one large step that does everything.

Usage is similar. From a dependant repo you can define your GitHub Actions workflow (.github/workflows/my-workflow.yml):

```yml
on: { push: { tags: ["v[0-9]+.[0-9]+.[0-9]+*"] } }
    
jobs:
  publish:
    uses: podNET-Hungary/pod-nugetizer/.github/workflows/default.yml@v1
    secrets: inherit
```

This being so well-formed, it's now possible to simply just:
```yml
on: { push: { tags: ["v[0-9]+.[0-9]+.[0-9]+*"] } }
jobs: { publish: { uses: podNET-Hungary/pod-nugetizer/.github/workflows/default.yml@v1, secrets: inherit }}
```

Or similarly to the action, passing of the parameters:
```yml
on: { push: { tags: ["v[0-9]+.[0-9]+.[0-9]+*"] } }
    
jobs:
  publish:
    uses: podNET-Hungary/pod-nugetizer/.github/workflows/default.yml@v1
    with:
      skip-prepack-debug: true
    secrets:
      NUGET_API_KEY: ${{ secrets.MY_NUGET_KEY }}
```
