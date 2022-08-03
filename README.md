# The Guild library config reference

This repository is a collection of configurations, tools and examples, that demonstrate how The Guild is using their libs.

## Available Features

### Automated Release Flow

Tools involved:
- Changesets
- NPM 
- GitHub Actions

To setup automated release flow for your package, using `changesets`:

1. Create a monorepo, either by using `yarn` (v1) or `pnpm`.
2. Install and initialize the Changesets config by following these instructions: https://github.com/changesets/changesets/blob/main/docs/intro-to-using-changesets.md 

Make sure to adjust you Changesets config file, based on your repo setup:

```json
{
  "$schema": "https://unpkg.com/@changesets/config@2.1.0/schema.json",
  "changelog": [
    "@changesets/changelog-github", // this will make nice output for changesets, with "thank you..." notes, and liks to the commits + references in PRs!
    { "repo": "guild-member/project-repo" } // Set the repo name here
  ],
  "commit": false,
  "linked": [],
  "access": "public",
  "baseBranch": "master", // change if needed
  "updateInternalDependencies": "patch", 
  "ignore": ["website"] // change if needed
}
```

3. Configure your monorepo packages correctly, you should make sure to have the following in your `package.json`:

```json
  "publishConfig": {
    "directory": "dist",
    "access": "public"
  },
```

> If you are not using a bundler/build flow, make sure to change the `directory` value if needed.

4. Configure Changesets scripts for the release/PR flows. You should have a script called `release`, that basically prepares the package for publishing, and then call `changesets` CLI to do the actual publishing:

```json
{
  "scripts": {
    "release": "yarn build && changeset publish"
  }
}
```

5. Configure your NPM publishing secrets. You can create an NPM publishing token by using `npm token create`. 

After creating your token, make sure to add it as part of your GitHub Actions Secrets (under repo Settings). Name it `NPM_TOKEN`.

```
NPM_TOKEN="..."
```

You should also make sure to setup `.npmrc` correctly with your action:

```yaml
      - name: Setup NPM credentials
        run: echo "//registry.npmjs.org/:_authToken=$NPM_TOKEN" >> ~/.npmrc
        env:
          NPM_TOKEN: ${{ secrets.NPM_TOKEN }}
```

6. Configure GitHub Actions permissions: Go to repo Settings > Actions > General and make sure to configure the following:

  - `Actions permissions` should be set to `Allow all actions and reusable workflows`
  - `Workflow permissions` should be set to `Read and write permissions`, and make sure the `Allow GitHub Actions to create and approve pull requests` option is active. 

7. Create a GitHub Actions workflow with your needs, and eventually make sure to add a Step for running your `changesets` script:

```yaml
      - name: Create Release Pull Request or Publish to npm
        id: changesets
        uses: changesets/action@master
        with:
          publish: yarn release # This points to your script
          commit: 'chore(release): update monorepo packages versions'  # You can customize the actual release commit name 
          title: 'Upcoming Release Changes' # You can customize the title of the Release PR if you want
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          NPM_TOKEN: ${{ secrets.NPM_TOKEN }}
```

> Note: Make sure to pass `fetch-depth: 0` to the checkout action, because Changesets is using Git history in order to check for changesets updates.

If you wish to use **Aggregated Releases** feature (to create a single, unified GitHub Release, instead of many), you can change the configuration to use this:

```yaml
      - name: Create Release Pull Request or Publish to npm
        uses: dotansimha/changesets-action@262e957f99be29087d2b13afa32a5d579bf1d080 # temporary, we are trying to get this merged
        with: # you can still use all other flags from before 
          publish: yarn release
          createGithubReleases: aggregate 
          githubReleaseName: "Release ${{ steps.vars.outputs.sha_short }} (from ${{ steps.vars.outputs.branch }})" # how to name the GitHub Release
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          NPM_TOKEN: ${{ secrets.NPM_TOKEN }}
```

8. Install the Changesets bot in your repository, in order to get automatic comments and warnings on missed/planned changesets: https://github.com/apps/changeset-bot 

### Automated Canary Release Flow 

Tools involved:
- Changesets
- NPM 
- GitHub Actions

To setup automated release flow for your package, using `changesets`, based on PR changes, use the following setup:

1. Follow all the instructions from `Automated Release Flow`. Once you get stable releases working, you can now set the canary releases (Note: Make sure to have at least oen stable release first) 
2. To setup snapshot/canary releases, start by updating your changesets `config.json` to use the following:

```json
{
  "$schema": "https://unpkg.com/@changesets/config@2.1.0/schema.json",
  // ... other stuff ...
  "snapshot": {
    "useCalculatedVersion": true,
    "prereleaseTemplate": "{tag}-{commit}.{datetime}"
  }
}
```

> You can customize the canary release template, see: https://github.com/changesets/changesets/blob/main/docs/config-file-options.md#prereleasetemplate-optional-string

3. Create a new script for running Changesets release in snapshot mode. This should match your `release` script requirements:

```json
{
  "scripts": {
    "release": "yarn build && changeset publish",
    "release:snapshot": "yarn build && changeset publish --no-git-tag --snapshot alpha"
  }
}
```

> You can choose the NPM tag of the release, by changing the value of the snapshot CLI flag. We prefer using `alpha` or `canary` for PR-based releases. 

4. Create a new GitHub Action pipeline, similar to the one you have for releases, that does the following:

```yaml
      - name: Release Snapshot # This will run your actual release script and will make sure to capture the output
        id: canary
        uses: 'kamilkisiela/release-canary@master'
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          NPM_TOKEN: ${{ secrets.NPM_TOKEN }}
        with:
          npm-token: ${{ secrets.NPM_TOKEN }}
          npm-script: 'yarn release:snapshot'
          changesets: true

      - name: Publish a message
        if: steps.canary.outputs.released == 'true'
        uses: 'kamilkisiela/pr-comment@master'
        with:
          commentKey: canary
          message: |
            The latest changes of this PR are available as canary in npm (based on the declared `changesets`):
            ```
            ${{ steps.canary.outputs.changesetsPublishedPackages}}
            ```
          bot-token: ${{ secrets.GITHUB_TOKEN }}
          bot: 'github-actions[bot]'
          github-token: ${{ secrets.GITHUB_TOKEN }}

      - name: Publish a empty message
        if: steps.canary.outputs.released == 'false'
        uses: 'kamilkisiela/pr-comment@master'
        with:
          commentKey: canary
          message: |
            The latest changes of this PR are not available as canary, since there are no linked `changesets` for this PR.
          bot-token: ${{ secrets.GITHUB_TOKEN }}
          bot: 'github-actions[bot]'
          github-token: ${{ secrets.GITHUB_TOKEN }}
```

Also, make sure you pipeline is configured for the following triggers:

```yaml
on:
  pull_request:
    paths:
      - '.changeset/**/*.md'
```

5. You can now create Pull Requests and add `changeset` to your PRs, and you should get autoamtic snapshot releases for your package 🎉


### Automated Dependencies Updates 

Tools involved:
- Renovate
- Changesets
- GitHub Actions

