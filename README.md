[![Deploy Docusaurus to GitHub Pages](https://github.com/colibriproject-dev/colibri-sdk-go-docs/actions/workflows/pages.yml/badge.svg)](https://github.com/colibriproject-dev/colibri-sdk-go-docs/actions/workflows/pages.yml)

# Website

This website is built using [Docusaurus](https://docusaurus.io/), a modern static website generator.

### Installation

```
$ yarn
```

### Local Development

```
$ yarn start
```

This command starts a local development server and opens up a browser window. Most changes are reflected live without having to restart the server.

### Build

```
$ yarn build
```

This command generates static content into the `build` directory and can be served using any static contents hosting service.

___


## Documentation Versioning

Use Docusaurus versioning to snapshot the current docs when you release a new SDK version.

When to create a new version
- After releasing a new SDK version that changes public APIs or documentation.
- Prepare and update the content in docs/ first; versioning will snapshot that content.

Create a new version
1. Make sure your working tree is clean and you are on the main branch.
2. Update docs/ for the upcoming release.
3. Run the versioning command (replace 0.1.5 with your new version tag):

```sh
yarn docusaurus docs:version 0.1.5
```

What this command does
- Creates a snapshot of the current docs at versioned_docs/version-0.1.5
- Generates a matching sidebar file at versioned_sidebars/version-0.1.5-sidebars.json
- Adds "0.1.5" to versions.json as the latest released version

Where to edit after versioning
- Ongoing work for the next release: edit docs/ (this is the "current" version).
- Hotfixes to an already released version: edit files under versioned_docs/version-<x>.

Version dropdown and navigation
- The Docs version dropdown should automatically show released versions based on versions.json.
- If your navbar item pins versions explicitly in docusaurus.config.js (e.g., versions: ['current', '0.1.4']), also add your new version to that array.

Local testing
```sh
yarn start
```
- Open the site locally and use the version dropdown to switch to the new version.

Deploying
```sh
yarn build
yarn deploy
```
- Ensure your GitHub Pages settings and permissions are configured (deployment branch: gh-pages).

Removing or fixing a version
If you need to remove an accidentally created version or roll back:
- Remove the entry from versions.json
- Delete the directory versioned_docs/version-<x>
- Delete the file versioned_sidebars/version-<x>-sidebars.json
- If you manually list versions in docusaurus.config.js, update that list as well.

References
- Docusaurus docs versioning guide: https://docusaurus.io/docs/versioning
