🚧 🚧 🚧 GUIDE UNDER CONSTRUCTION 🚧 🚧 🚧

This is the first draft of this document and it currently leans pretty heavily on a existing setup for `reactotron`. Eventually all the information needed should be available within this guide.

# Electron app setup

When setting up an Electron app there are a lot of moving parts and things to consider. To ease the setup process this guide will walk you though some basic setup steps from basic building, setting up an auto updater and using Circle CI for CI/CD

## Building

Electron apps are made up of two somewhat separate parts. The first part is the binary that the operating system actually runs. This will need to be compiled specifically for each platform you want to target. We will call this the `Binary`. The second part is the actual implementation of whatever you want the app to do. This can be written in any webbased technology (`react`, `angular`, `vue`, `jQuery`, etc). This guide will be assuming it will be built in `react` and bundled with `webpack` (https://webpack.js.org/) although these play a minor part in this guide so feel free to switch them out and adapt this guide to whatever technology you like. We will call this secondar part the `App`.

### Building the `App`

The way that electron works is it just uses Chrome to render a website but provides certain APIs to allow for access to certain system APIs (like access to the filesystem). As such the `App` is basically just a website hosted in its own application that a user can run. Typically when working with `webpack` for the web your end result is a javascript file that is your entry point. Much like web that will be our goal for our Electron app too.

The first step to getting a working build is to get your javascript bundle built. I won't go into details around `webpack` configuration as that is a topic covered in all corners of the web ad nauseam. A good exmaple you can follow (which is actually Electron focused) is https://github.com/electron-react-boilerplate/electron-react-boilerplate.

I would recommend having all the source files for `App` in some sort of `src` or `app` directory. The bundle that gets output is configurable so you can have that setup in whatever way you would like.

### Building the `Binary`

Once you have a basic bundle for you application being built you will need to be able to package up binaries for all platforms you would like to target. For this I recommend using `electron-builder` (https://www.electron.build/) but you can do this with a few other methods. The first thing that you will need to setup is a entry html file. This file is the webpage that is actually rendered by Electron. This file should be located alongside your `App` code. Its job will be to do any setup needed in the browser and to also add the `App` bundle as a script tag. There is nothing particularly special about this html file. I would recommend using the one from the `electron-react-boilerplate` as a starting point for this html file. The next thing you will need is a bit of javascript to wire up some things for your application. These things include the menu, loading the actual html file and wiring up any events you might want to watch for on the applications window. I recommend calling this `main.js` or `main.dev.js`. This should typically be located alongside your `App` code. The main file included in `electron-react-boilerplate` is a really good, clean example of a main file. Once you have both of these things you will need to configure `webpack` to build the javascript file and generate a bundle. This is the final bundle you will need to generate for the app.

Now that you have a lot of bundled javascript code you need to setup `electron-builder`. You will start by adding a `build` object to your `package.json`. Within this `build` object you can include a bunch of configuration for how the application will be bundled for each platform. I will cover some of the more common ones here but you can find a lot of other options here: https://www.electron.build/configuration/configuration.

- `productName` - A string containing the name of the application
- `appId` - A string that is app ID. This is needed for when you are creating the signing certificate for macOS
- `files` - An array of files to include in the compiled application
- `directories`
  - `buildResources` - The name of the folder containing resources (like an app icon)
  - `output` - Where to put the binaries
- `publish` - Configuration for where to publish artifacts if you have `electron-builder` handle the release process (this guide does not but is technically an option).

An example configuration from `reactotron`:

```json
"build": {
  "productName": "Reactotron",
  "appId": "com.reactotron.app",
  "files": [
    "src/dist/",
    "src/app.html",
    "src/main.prod.js",
    "src/main.prod.js.map",
    "package.json"
  ],
  "directories": {
    "buildResources": "resources",
    "output": "release"
  },
  "publish": {
    "provider": "github",
    "owner": "infinitered",
    "repo": "reactotron",
    "private": false
  }
}
```

The files must include the output of the `App` build, the html file for entry (named `app.html`), the output of the `main` build and the `package.json`.

With all that configuration setup you should be ready to try and run your first build. Here are some helpful scripts to get you started:

```json
"scripts": {
  "build": "concurrently \"yarn build-main\" \"yarn build-renderer\"",
  "build-main": "cross-env NODE_ENV=production webpack --config ./configs/webpack.config.main.prod.babel.js --colors",
  "build-renderer": "cross-env NODE_ENV=production webpack --config ./configs/webpack.config.renderer.prod.babel.js --colors",
  "package": "yarn build && electron-builder build",
  "package-all": "yarn build && electron-builder build -mwl",
}
```

The `build` commands simply run webpack to build the bundles. The `package` command will take it a setup further and generate a binary for the operating system you are on. Sometimes that isn't enough and you will want to build for all the operating systems. That is where `package-all` comes in which includes a `-mwl` argument. `m` - macOS, `w` - Windows, `l` - Linux.

## CI Setup

Now that we can build and create binaries we need to make it automated since no one wants to be responsible for manually building the application. This guide uses CircleCI as the CI service and makes the assumption that you know basic CircleCI configuration. Lets start with some code and break it down. Here is the CircleCI config for `reactotron`:

```yml
# Javascript Node CircleCI 2.0 configuration file
#
# Check https://circleci.com/docs/2.0/language-javascript/ for more details
#

defaults: &defaults
  macos:
    xcode: "10.1.0"
  working_directory: ~/repo

version: 2
jobs:
  build_and_test:
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
      - run:
          name: Build Apps
          command: yarn build

  releaseVersion:
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
      - run:
          name: Install Wine & RPM
          command: brew install wine && brew install rpm
      - run:
          name: Decode Certificates
          command: base64 -D -o Certificates.p12 <<< $ReactotronCerts
      - run:
          name: Install Gems
          command: bundle install
      - run:
          name: Install Cert
          command: bundle exec fastlane setup
      - run:
          name: Run Release
          command: yarn ci:publish

workflows:
  version: 2
  test_and_release:
    jobs:
      - build_and_test:
          filters:
            branches:
                ignore: master
      - releaseVersion:
          context: ReactotronCerts
          filters:
            branches:
                only: master
```

There are only two jobs setup right now because I was having difficulty with getting the macOS build servers to use the caching mechinism provided by CircleCI. The first job is setup to only build the `App` to ensure that ic can. This job is used by all branches other then master and PRs. The second job is where the real magic happens.

Here is the breakdown of the steps:
- Restore node modules
    - We restore any cached `node_modules` (I am not convinced at this moment that this is actually working)
- Install dependencies
    - Install all `node_modules`
- Save node modules
    - Save the `node_modules` cache (again, not sure this works right now)
- Install Wine & RPM
    - These are preqs for building Windows binaries
- Decode Certificates
    - This is pulling in the certificates used to sign the macOS build. Most documentation suggests using `fastlane match` to manage the certificates but in this case I was not wanting to do that so I found a great article showing how to handle certificates without having to have a repo hosting the certificates. (https://medium.com/@m4rr/circleci-2-0-and-the-ios-code-signing-df434d0086e2)
- Install Gems
    - Reactotron is setup with a few ruby gems. Lets install em.
- Install Cert
    - This runs a fastlane lane that just installs the cert into the build servers keychain. You can find the script below.
- Run Release
    - This runs the `semantic-release` which handles building the entire electron app and publishes it to GitHub.

#### Fastlane Script
```ruby
setup_circle_ci
import_certificate(
    keychain_name: ENV["MATCH_KEYCHAIN_NAME"],
    keychain_password: ENV["MATCH_KEYCHAIN_PASSWORD"],
    certificate_path: 'Certificates.p12',
    certificate_password: ENV["CERTIFICATE_PASSWORD"] || "default",
    log_output: true
)
```
