# Azure DevOps multi-stage pipeline template

This repository hosts a reusable Azure DevOps pipeline definition that validates pull requests and manages continuous integration deployments for .NET solutions. The pipeline is designed to be copied into any repository that contains one or more `*.csproj` projects and provides strict quality gates before code can be merged.

## Pipeline capabilities

### Pull request validation stage

When a pull request targets the `development`, `staging`, or `main` branches, the **PR_Validation** stage runs automatically and enforces the following checks:

1. **Project discovery and compilation** – every `*.csproj` file in the repository is restored and built to ensure the solution is compilable.
2. **Targeted unit test execution** – only the test projects that were modified in the pull request are executed, and the run is narrowed to the test classes that changed. If no tests were updated, the pipeline falls back to running the entire affected test projects.
3. **Code coverage on changed lines** – Cobertura coverage is produced from the targeted test run. The pipeline fails if any non-test C# line added or modified by the pull request is missing from the coverage report or has zero hits.
4. **Reporting** – detailed test results and coverage reports are published to the pipeline summary.

These checks ensure that new code is accompanied by unit tests and that the affected tests are reported before reviewers approve the pull request.

### Continuous integration stage

For direct pushes (after a pull request is completed) to `development`, `staging`, or `main`, the **Continuous_Integration** stage runs:

1. **Full solution build** – restores and builds every project in `Release` mode.
2. **Full test pass with coverage** – executes the entire unit-test suite, producing Cobertura coverage and TRX test logs.
3. **Selective Linux deployments** – only projects affected by the latest commit are published. ASP.NET Core projects (SDK `Microsoft.NET.Sdk.Web`) are zipped and deployed to Azure App Service on Linux, while Azure Functions projects (SDK `Microsoft.NET.Sdk.Functions`) are packaged and deployed to Azure Functions on Linux using the Azure CLI.
4. **Reporting** – publishes aggregate test results and coverage artifacts for the run.

## Repository layout

```
azure-pipelines.yml     # Multi-stage pipeline definition
deployment-map.json     # Maps project files to Azure resources for deployment
README.md               # Usage and configuration guide (this file)
```

Copy the files into the root of any Azure DevOps Git repository that hosts .NET applications to immediately adopt the workflow.

## Prerequisites

To use this pipeline you need:

- An Azure DevOps organization with the **Azure Pipelines** service enabled.
- A project with a Git repository containing one or more `.csproj` files.
- Unit-test projects that either include `<IsTestProject>true</IsTestProject>` in the project file **or** contain `Test`/`Tests` in the project name.
- The .NET SDK version configured by the pipeline (`8.0.x` by default). Update the `dotnetVersion` variable if your solution targets another LTS.
- An **Azure Resource Manager service connection** with contributor access to the resource groups that host your App Service and Function App instances. Configure its name in the `azureServiceConnection` pipeline variable.
- Existing **Linux-based Azure App Service** and/or **Azure Functions** resources. The pipeline deploys zipped packages to these resources.
- Pull requests must be created against `development`, `staging`, or `main` branches to trigger PR validation. Update the `trigger` / `pr` sections if your branching model differs.

## Azure DevOps setup instructions

1. **Copy the pipeline files** – add `azure-pipelines.yml`, `deployment-map.json`, and this README to the root of the target repository and commit the files.
2. **Create the pipeline**
   - In Azure DevOps, open **Pipelines ➜ New pipeline**.
   - Choose **Azure Repos Git** and select your repository.
   - When prompted, pick the existing YAML file and browse to `azure-pipelines.yml`.
   - Save the pipeline; it will automatically run for the configured branches.
3. **Grant permissions (if required)** – the YAML uses checkout with `persistCredentials` so the build agent can fetch branches for diffs. Make sure the project collection build service has read access to your repo.
4. **Configure Azure access** – set the `azureServiceConnection` variable in the pipeline to the name of your Azure Resource Manager service connection. Store any sensitive values (for example, slot names) as secret variables or variable-group entries.
5. **Review policies** – consider enabling branch policies that require the PR validation pipeline to succeed before merging into `development`, `staging`, and `main`.

## Solution requirements and conventions

- **Project detection** – the scripts enumerate every `*.csproj`. Ensure all deployable code is represented by project files in source control.
- **Test detection** – the pipeline treats a project as a test project when either:
  - the project file contains `<IsTestProject>true</IsTestProject>`, or
  - the project file name includes `Test`/`Tests`.
  Adapt the detection logic by editing the PowerShell queries if your naming differs.
- **Targeted PR tests** – only test classes present in changed files are executed. If no classes are detected, the whole affected test projects run. This protects against false negatives while still keeping PR validation fast.
- **Coverage enforcement** – the PR stage fails if any modified non-test C# line lacks coverage. Update the coverage verification script if you need a different tolerance (for example, allowing uncovered lines in generated files).
- **Deployment mapping** – maintain `deployment-map.json` so each deployable project points to its Azure resource group, resource name, and optional slot. The pipeline auto-detects ASP.NET Core web apps and Azure Functions and deploys them to Linux-based App Service and Function App plans respectively. Projects missing from the map cause the deployment stage to fail so you explicitly configure every deployable workload.

## Customization guide

- **Change the branch filters** – edit the `trigger` and `pr` sections at the top of `azure-pipelines.yml` to match your branching strategy.
- **Adjust the build configuration** – change the `buildConfiguration` variable if you prefer `Debug` or another configuration.
- **Support additional SDKs** – add extra `UseDotNet@2` tasks if your solution spans multiple target frameworks.
- **Adjust deployment packaging** – update the publish step if you need self-contained builds, specific runtimes, or artifact retention. The generated zip files are consumed directly by the Azure CLI tasks.
- **Add new resource types** – extend the PowerShell deployment planner and Azure CLI step if you need to target services beyond App Service or Functions (for example, container apps or static web apps).
- **Parallelism** – convert jobs to run in parallel (e.g., separate build/test/deploy jobs) if you need to scale for large solutions.

## Troubleshooting

| Symptom | Possible cause | Resolution |
| --- | --- | --- |
| PR pipeline fails with `No .csproj files were found` | Repository does not contain project files or the pipeline runs in the wrong directory | Confirm the pipeline file resides in the repository root and projects are checked in. |
| Coverage verification fails because a file is missing | New production code lacks tests | Add or update unit tests so the changed lines are executed. |
| Deployment stage fails with missing mapping | `deployment-map.json` does not include the changed project | Add the project entry (including resource group, app name, and type) so the pipeline knows where to deploy. |
| Deployment step skipped | `azureServiceConnection` variable was not set | Edit the pipeline variables and supply the name of the Azure service connection. |

## Extending the template

Feel free to fork this repository and tailor the pipeline logic for other ecosystems (for example, adding linting, packaging, or integration-test stages). Contributions that enhance test discovery, coverage diffing, or deployment strategies are welcome.

## Configuring deployments

The `deployment-map.json` file controls how changed projects are deployed. Each entry must provide the relative path to the project file, the deployment `type`, and Azure resource information:

```json
{
  "projects": [
    {
      "path": "src/Contoso.Api/Contoso.Api.csproj",
      "type": "appService",
      "resourceGroup": "rg-contoso-shared-linux",
      "appName": "contoso-api",
      "slot": "production"
    },
    {
      "path": "src/Contoso.Functions/Contoso.Functions.csproj",
      "type": "functionApp",
      "resourceGroup": "rg-contoso-serverless",
      "appName": "contoso-func"
    }
  ]
}
```

- Set `type` to `appService` for ASP.NET Core web/API projects that use the `Microsoft.NET.Sdk.Web` SDK.
- Set `type` to `functionApp` for Azure Functions projects that use the `Microsoft.NET.Sdk.Functions` SDK.
- `slot` is optional; omit it to deploy to the production slot.

When the CI stage runs, the pipeline:

1. Identifies changed projects by comparing the latest commit to the previous commit.
2. Validates each project against the deployment map and infers whether it is an App Service or Function App.
3. Publishes the project in `Release` mode for Linux hosting and zips the output.
4. Uses the configured service connection to run `az webapp deploy` (App Service) or `az functionapp deployment source config-zip` (Function App) with the generated package.

If you add new deployable projects in the future, append their entries to `deployment-map.json` and update the Azure resources as needed.

