---
name: create-code-app
description: Scaffold, configure, and deploy a new Power Platform Code App. Use when the user says they want to create a new code app, start a code app project, or set up a Power Apps code-first app.
---

# Create a Power Platform Code App

## When to use
User wants to create/scaffold a new Power Platform code app and wire it to data.

## Prerequisites (verify first)
Run and confirm each returns a version:
    node -v
    npm -v
    git --version
    pac --version
If `pac` is missing, install the Power Platform CLI before continuing.

## Steps

### 1. Scaffold the project
    npx degit github:microsoft/PowerAppsCodeApps/templates/vite my-code-app

### 2. Move into the folder and open the editor
    cd my-code-app
    code . -r

### 3. Authenticate
    pac auth create
    pac env list
    pac env select --environment <environment-id>

### 4. Install dependencies and initialize
    npm install
    pac code init --displayname "My Code App"
    npm install @microsoft/power-apps

### 5. Add data sources (as needed)
Dataverse:
    pac code add-data-source -a shared_dataverse -c <connectionId> -t <table-logical-name>
SharePoint / SQL (common form):
    pac code add-data-source -a <apiName> -c <connectionId> -t <tableId> -d <datasetName>
List available connections / tables first:
    pac connection list
    pac code list-tables -a <apiName> -c <connectionId>

### 6. Run locally
    npm run dev

### 7. Build, then push (push ONLY if build succeeds)
    npm run build
    pac code push --solutionName "YourSolutionName"

Done when: `pac code push` completes with no errors and the app appears in the target environment.