# Repro case for eyes-storybook/NPM workspaces

Repo to reproduce eyes-storybook monorepo issues.

## Problem

When run in a monorepo setup, `@applitools/eyes-storybook`'s CLI fails to start and does not show an error (but does exit non-zero)

## Using this repo to reproduce the problem

```sh
# Get the repo
git clone https://github.com/betaorbust/eyes-repro && cd eyes-repro

# Make sure we're on the right version of node/npm and install
nvm use
npm install

# So we don't have to add it each time
export APPLITOOLS_API_KEY="<YOUR API KEY>"

# Repro the problem
npm run repro

# You should see
# > eyes-repro@1.0.0 repro
# > cd ./packages/package1 && npm run visual-testing
#
#
# > package1@1.0.0 visual-testing
# > npx eyes-storybook
#
# Using @applitools/eyes-storybook version 3.27.6.
#
# ⠋ Starting storybook servernpm ERR! Lifecycle script `visual-testing` failed with error:
# npm ERR! Error: command failed
# npm ERR!   in workspace: package1@1.0.0
# npm ERR!   at location: <repo root>/eyes-repro/packages/package1


# Other ways to more directly trigger the problem:
# Using the package directly
npm run visual-testing --workspace=package1
# Changing directly to the package directory and running eyes-storybook directly
cd ./packages/package1 && npx eyes-storybook


# Clean up
unset APPLITOOLS_API_KEY
```

## Explanation

Currently `@applitools/eyes-storybook` doesn't work when run in an NPM workspace/monorepo because NPM hoists the `node_modules/.bin/\*` contents up to the root of the monorepo when possible.  
So if you have a package in your monorepo that uses `@applitools/eyes-storybook` and you attempt to run it, you will get a silent error and an early exit.

Our example case:

```sh
cd ./packages/package1 # go to the package
npx eyes-storybook # try to run eyes-storybook
```

When run, the `cwd` for the package above is `./packages/package1` and the CLI does the following:

1. [uses `process.cwd` as the `packagePath` in a call to `validateAndPopulateConfig`](https://github.com/applitools/eyes.sdk.javascript1/blob/6b46adf6fe0bdc10142ce593f604a5e7364722a2/packages/eyes-storybook/src/cli.js#L36)
2. [passes that package path to `startStorybookServer`](https://github.com/applitools/eyes.sdk.javascript1/blob/6b46adf6fe0bdc10142ce593f604a5e7364722a2/packages/eyes-storybook/src/validateAndPopulateConfig.js#L42), through [a call to `new StorybookConnector`](https://github.com/applitools/eyes.sdk.javascript1/blob/6b46adf6fe0bdc10142ce593f604a5e7364722a2/packages/eyes-storybook/src/validateAndPopulateConfig.js#L42)
3. [spawn call to launch the executable](https://github.com/applitools/eyes.sdk.javascript1/blob/6b46adf6fe0bdc10142ce593f604a5e7364722a2/packages/eyes-storybook/src/storybookConnector.js#L118) at that path.

Unfortunately, NPM has hoisted everything away to the root of the repo so it fails to start the server. This error stack trace [gets swallowed](https://github.com/applitools/eyes.sdk.javascript1/blob/6b46adf6fe0bdc10142ce593f604a5e7364722a2/packages/eyes-storybook/src/storybookConnector.js#L130) and under default circumstances does not produce an error in the console other than a non-zero exit.

## Suggestion

I believe using `npx` at the spawn site, which will include PATH variables that will resolve to the closest `start-storybook` .bin file might work here, but I'm not sure what other constraints the product has.
