# CircleCI CD Setup - NPM

This document shows the steps necessary to set up automatic continuous integration testing and automatic NPM deployment upon successfully merging a pull request.

## First Things First

1. Write tests
  - If the project already has tests, great. If not, write some.
  - They better pass! Tests are important because you don't want to be deploying broken code.
  - See [this](https://github.com/infinitered/ignite-webview/blob/master/test.js) for an example of a very simple testing setup

## CircleCI Setup

1. [Log into CircleCI](https://circleci.com/vcs-authorize/) with your Github account
2. Choose `infinitered` from the dropdown in the top left
3. Navigate to `Add Projects` on the left
4. Search for your repo
5. Choose `Set Up Project`
  - If you see `Contact Repo Admin`, you will need to be given admin permissions on CircleCI. See [Jamon](https://github.com/jamonholmgren) about this.
6. Set Up Project
  - Select `Linux` for the operating system and `Node` for the language
7. Commit your code changes and push to github
8. Choose `Start building` to initiate the first CI build
9. Enable builds from forked pull requests. Go to project settings > Advanced Settings, then toggle on `Build forked pull requests`

## Configure Code for CircleCI

1. Create a folder in the project root named `.circleci`.
2. Create a file inside that folder named `config.yml`
3. Use the below template in that file. For simple Node projects, you likely won't have to change it.
4. If needed, see [configuration docs](https://circleci.com/docs/2.0/config-intro/#section=configuration) for additional configuration options.

```yaml
# Javascript Node CircleCI 2.0 configuration file
#
# Check https://circleci.com/docs/2.0/language-javascript/ for more details
#

defaults: &defaults
  docker:
    # Choose the version of Node you want here
    - image: circleci/node:10.11
  working_directory: ~/repo

version: 2
jobs:
  setup:
    <<: *defaults
    steps:
      - checkout
      - restore_cache:
          name: Restore node modules
          keys:
            - v1-dependencies-{{ checksum "package.json" }}
            # fallback to using the latest cache if no exact match is found
            - v1-dependencies-
      - run:
          name: Install dependencies
          command: yarn install
      - save_cache:
          name: Save node modules
          paths:
            - node_modules
          key: v1-dependencies-{{ checksum "package.json" }}


  tests:
    <<: *defaults
    steps:
      - checkout
      - restore_cache:
          name: Restore node modules
          keys:
            - v1-dependencies-{{ checksum "package.json" }}
            # fallback to using the latest cache if no exact match is found
            - v1-dependencies-
      - run:
          name: Run tests
          command: yarn ci:test # this command will be added to/found in your package.json scripts

workflows:
  version: 2
  test_and_release:
    jobs:
      - setup
      - tests:
          requires:
            - setup
```

5. Make sure the test script is added to your `package.json`

```json
  {
    ...
    "scripts": {
      ...
      "ci:test": "<command to run tests>" <<-- if you don't already have this one
    },
    ...
  }
  ```

## Add Semantic Release

1. Install semantic-release and git plugin as dev dependencies
  - `yarn add --dev semantic-release @semantic-release/git`
2. Add a `publish` section to your `.circleci/config.yml`, like so:
  
```
defaults: ...

version: 2
jobs:
setup: ...

tests: ...

publish:
  <<: *defaults
  steps:
    - checkout
    - run: echo "//registry.npmjs.org/:_authToken=$NPM_TOKEN" >> ~/.npmrc
    - restore_cache:
        name: Restore node modules
        keys:
          - v1-dependencies-{{ checksum "package.json" }}
          # fallback to using the latest cache if no exact match is found
          - v1-dependencies-
    # Run semantic-release after all the above is set.
    - run:
        name: Publish to NPM
        command: yarn ci:publish # this will be added to your package.json scripts


workflows:
  version: 2
  test_and_release:
    jobs:
      - setup
      - tests:
          requires:
            - setup
      - publish:
          requires:
            - tests
          filters:
            branches:
              only: master
```

3. Add publish scripts to your `package.json`:

  ```json
  {
    ...
    "scripts": {
      ...
      "ci:test": "<command to run tests>",
      "ci:publish": "yarn semantic-release",
      "semantic-release": "semantic-release"
    },
    ...
  }
  ```

4. Add release configuration to your `package.json`:
  
  ```json
  {
    ...
    "release": {
      "plugins": [
        "@semantic-release/commit-analyzer",
        "@semantic-release/release-notes-generator",
        "@semantic-release/npm",
        "@semantic-release/github",
        [
          "@semantic-release/git",
          {
            "assets": "package.json",
            "message": "chore(release): ${nextRelease.version} [skip ci]\n\n${nextRelease.notes}"
          }
        ]
      ]
    }
    ...
  }
  ```

5. Add npm and github tokens to env vars on CircleCI (https://circleci.com/gh/infinitered/YOURPROJECT/edit#env-vars). You should be able to find these in our team 1password under `CircleCI CI/CD Semantic Release Tokens`. 
6. Add the `Circle CI` Github team (Including the `infinitered-circleci` user) to your repo (https://github.com/infinitered/YOURPROJECT/settings/collaboration)
7. Check git tags `git tag --merged master`. Ensure that what shows up there matches what you expect to see.
  - Sometimes this doesn't match what Github shows. So, you need to do a manual release and be sure to tag it in git with `git tag -a v1.4.0 -m "my version 1.4.0"`. Talk to Carlin or Jamon about this if you're confused.

## Test Automatic Release

1. Create a new empty commit with `git commit --allow-empty -m "fix(ci): Testing CI release"`
2. Push it to `master` (or do a pullrequest and merge it in)
3. CircleCI should deploy a new version to NPM
 - NOTE: If CircleCI does the wrong thing and a version goes up to NPM that you don't want, just run `npm unpublish YOURPROJECT@1.4.1` or whatever version. If you are able to do this within 15 minutes NPM will remove the version. Beyond 15 minutes all you can do is use `npm deprecate` and deploy a new version.

## Commit message format

Format your commit messages like so to get the desired version bump:

| Format | Version bump | Example |
|--------|--------------|---------|
| `fix/refactor(feature): Message` | Patch-level, e.g. `1.0.3` to `1.0.4` | `fix(wkwebview): Fixed issue #12 - crash when navigating` |
| `feat(feature): Message` | Minor/Feature-level, e.g. `1.0.3` to `1.1.0` | `feat(android): Added file upload` |
| `fix/feat/perf(feature): BREAKING CHANGES: Message` | Major/Breaking-level, e.g. `1.0.3` to `2.0.0` | `fix(ios): BREAKING CHANGES: Removed UIWebView` |

Each release will include a changelog:

![image](https://user-images.githubusercontent.com/1479215/48959809-cb13df00-ef1c-11e8-9281-579df168c0e7.png)

Semantic Release will also comment on any issues that are fixed in a particular release.

TIP: If you set up your repo to only allow `Squash & Merge` on pull request, you will get a chance to edit the commit message and release notes and cut a new release.

![image](https://user-images.githubusercontent.com/1479215/48959734-31e4c880-ef1c-11e8-96db-4e854a8edc1d.png)

## Questions? Issues?

Open an issue in this repo and tag @carlinisaacson and @jamonholmgren.

If you have a fix for one of these instructions, just edit this file on Github (:pencil: top right) and commit it directly to `master`, no pull request needed.
