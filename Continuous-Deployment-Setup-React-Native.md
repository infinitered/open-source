🚧 🚧 🚧 GUIDE UNDER CONSTRUCTION 🚧 🚧 🚧

This is untested as we're waiting on CircleCI to resolve an issue on their end where macOS builds are getting stuck.

# CircleCI CD Setup - React Native

This document shows the steps necessary to set up automatic continuous integration testing and automatic Fastlane beta builds upon successfully merging a pull request.

## First Things First

1. Write tests
  - If the project already has tests, great. If not, write some.
  - They better pass! Tests are important because you don't want to be deploying broken code.
  - See [this](https://github.com/infinitered/ChainReactApp2018) for an example of how we typically setup tests for a React Native app.

## CircleCI Setup

1. [Log into CircleCI](https://circleci.com/vcs-authorize/) with your Github account
2. Choose `infinitered` from the dropdown in the top left
3. Navigate to `Add Projects` on the left
4. Search for your repo
5. Choose `Set Up Project`
  - If you see `Contact Repo Admin`, you will need to be given admin permissions on CircleCI. See [Jamon](https://github.com/jamonholmgren) about this.
6. Set Up Project
  - Select `macOS` for the operating system and `Other` for the language
7. Copy the basic config.yml to `.circleci/config.yml`, commit your code changes and push to github `master`
8. Choose `Start building` to initiate the first CI build. This build will fail. That's ok. We will update the config in the next step.
9. Enable builds from forked pull requests. Go to project settings > Advanced Settings, then toggle on `Build forked pull requests`

## Configure Code for CircleCI

1. Create a folder in the project root named `.circleci`.
2. Create a file inside that folder named `config.yml`
3. Use the below template in that file. For simple Node projects, you likely won't have to change it.
4. If needed, see [configuration docs](https://circleci.com/docs/2.0/config-intro/#section=configuration) for additional configuration options.
_(Here is a complete [config.yml](https://github.com/infinitered/open-source/blob/master/config.example.yml) with CI and CD steps completed)_

```yaml
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
          keys:
            - v1-dependencies-node-{{ checksum "package.json" }}
            # fallback to using the latest cache if no exact match is found
            - v1-dependencies-node-
      - run:
          name: Install dependencies
          command: yarn install
      - save_cache:
          name: Save node modules
          paths:
            - node_modules
          key: v1-dependencies-node-{{ checksum "package.json" }}

  tests:
    <<: *defaults
    steps:
      - checkout
      - restore_cache:
          keys:
            - v1-dependencies-node-{{ checksum "package.json" }}
            # fallback to using the latest cache if no exact match is found
            - v1-dependencies-node-
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

## Add Fastlane

1. Before you can add continuous deployment, you'll need to setup Fastlane and Match to sign and deploy your app. You can follow these blog posts
to get setup!

[Releasing on iOS with Fastlane](https://shift.infinite.red/simple-react-native-ios-releases-4c28bb53a97b)
[Releasing on Android with Fastlane](https://shift.infinite.red/simple-react-native-android-releases-319dc5e29605)

2. Once you're setup with Fastlane (you should be able to successfully release a beta (or alpha) build to TestFlight or the Google Play store using `fastlane ios beta` and `fastlane android beta`), the first thing will be to make sure CircleCI has all the credentials to run your fastlane scripts:

2a. Go into the Settings screen for your project on CircleCI
2b. Under "Build Settings", click on "Environment Variables"
2c. Click "Add Variable"
2d. Set `FASTLANE_USER` to the email address of your your Apple App Store Connect / Dev Portal user. For Infinite Red projects, this is probably: `admin@infinite.red`
2e. Do this for all of the variables listed [here](https://github.com/fastlane/docs/blob/950c6f42231d86b5187d2cfdcab2a6c81d0f61dc/docs/best-practices/continuous-integration.md#environment-variables-to-set)

You can find more info from the [Fastlane Docs](https://github.com/fastlane/docs/blob/950c6f42231d86b5187d2cfdcab2a6c81d0f61dc/docs/best-practices/continuous-integration.md)

3. Now you can add `deploy` jobs to your CircleCI `config.yml`:

```yaml
defaults: ...

mac: &mac
  macos:
    xcode: "10.1.0"
  working_directory: ~/repo

version: 2
jobs:
  setup: ...

  tests: ...

  deploy_ios:
    <<: *mac
    steps:
      - checkout
      - restore_cache:
          keys:
            - v1-dependencies-mac-{{ checksum "package.json" }}
            # fallback to using the latest cache if no exact match is found
            - v1-dependencies-mac-
      - run:
          name: Install dependencies
          command: yarn install
      - save_cache:
          name: Save node modules
          paths:
            - node_modules
          key: v1-dependencies-mac-{{ checksum "package.json" }}
      - run:
          name: Install Gems
          command: cd ios && bundle install
      - run:
          name: Fastlane
          command: cd ios && bundle exec fastlane ios beta

  deploy_android:
    <<: *mac
    steps:
      - checkout
      - restore_cache:
          keys:
            - v1-dependencies-mac-{{ checksum "package.json" }}
            # fallback to using the latest cache if no exact match is found
            - v1-dependencies-mac-
      - run:
          name: Install dependencies
          command: yarn install
      - save_cache:
          name: Save node modules
          paths:
            - node_modules
          key: v1-dependencies-mac-{{ checksum "package.json" }}
      - run:
          name: Install Gems
          command: cd android && bundle install
      - run:
          name: Fastlane
          command: cd android && bundle exec fastlane android beta

workflows:
  version: 2
  test_and_release:
    jobs:
      - setup
      - tests:
          requires:
            - setup
      - deploy_ios:
          requires:
            - tests
          filters:
            branches: master
      - deploy_android:
          requires:
            - tests
          filters:
            branches: master
```

If you have CocoaPods dependencies, make sure you also add the following steps to your `deploy_ios` job before you execute Fastlane:

```yaml
      - run:
          name: Fetch CocoaPods Specs
          command: |
            curl https://cocoapods-specs.circleci.com/fetch-cocoapods-repo-from-s3.sh | bash -s cf
      - run:
          name: Install CocoaPods
          command: cd ios && pod install --verbose
```

NOTE: The macOS boxes currently come with Node 11.0, with no apparent way to change the version. This shouldn't be a huge problem. One known issue is with `upath`, which is a deep dependency of react-native. If you encounter errors related to `upath` requiring a lower version of Node, just make sure it is at `1.1.0`, and not `1.0.4` in your `yarn.lock`. See https://github.com/airbnb/enzyme/issues/1637#issuecomment-397327562.

----------------- WIP --------------------------