# Working with the Kubernetes UI
This document explains how to work with the Kubernetes UI. For information on how to access and use it, see [docs/user-guide/ui.md](../docs/user-guide/ui.md).

## Installing dependencies
There are two kinds of dependencies in the UI project: tools and frameworks. The tools help
us manage and test the application. They are not part of the application. The frameworks, on the other hand, become part of the application, as described below.

* We get the tools via `npm`, the [node package manager](https://www.npmjs.com/). 
* We get the frameworks via `bower`, a [client-side package manager](http://bower.io/).

Before you build the application for the first time, run this command from the `www/master` directory:

```
npm install
```

It creates a new directory, `www/master/node_modules`, which contains the tool dependencies.

## Building and serving the app

### Building the app for development
To build the application for development, run this command from the `www/master` directory:

```
npm start
```

It runs `bower install` to install and/or update the framework dependencies, and then `gulp`, a [JavaScript build system](http://gulpjs.com/), to generate a development version of the application.

Bower creates a new directory, `third_party/ui/bower_components`, which contains the framework dependencies. Each of them should be referenced in one of the `vendor.json` files below:

* `www/master/vendor.base.json` - 3rd party vendor javascript files required to start the app. All of the dependencies referenced by this file are compiled into `base.js` and loaded before `app.js`.
* `www/master/vendor.json` - 3rd party vendor js or css files required to make the app work, usually by lazy loading. All of the dependencies referenced by this file are compiled into `www/app/vendor`. (Note: some framework dependencies have been hand edited and checked into source control under `www/master/shared/vendor`.)

The default `gulp` target builds the application for development (e.g., without uglification of js files or minification of css files), and then starts a file watcher that rebuilds the generated files every time the source files are updated. (Note: the file watcher does not support adding or deleting files. It must be stopped and restarted to pick up additions or deletions).

The `www/app` directory and its contents are generated by the build. All of the other files under `www` are source or project files, such as tests, scripts, documentation and package manifests. (Note: the build output checked into source control is the production version, built with uglification and minification, as described below, so expect the build output to change if you build for development.)

### Serving the app during development

For development you can serve the files locally by installing a web server as follows:

```
sudo npm install -g http-server
```

The server can then be launched from the `www/app` directory as follows:

```
cd www/app
http-server -a localhost -p 8001
```

`http-server` is convenient, since we're already using `npm`, but any web server hosting the `www/app` directory should work.

Note that you'll need to tell the application where to find the api server by setting the value of the `k8sApiServer` configuration parameter in `www/master/shared/config/development.json` and then rebuilding the application. For example, for a cluster running locally at `localhost:8080`, as described [here](../docs/getting-started-guides/locally.md), you'll want to set it as follows:

```
"k8sApiServer": "http://localhost:8080/api/v1"
```

### Building the app for production
To build the application for production, run this command from the `www/master` directory:

```
npm run build
```

Like `npm start`, it runs `bower install` to install and/or update the framework dependencies, but then it runs `gulp build` to generate a production version of the application. The `build` target builds the application for production (e.g., with uglification of js files and minification of css files), and does not run a file watcher, so that it can be used in automated build environments.

To make the production code available to the Kubernetes api server, run this command from the top level directory:

```
hack/build-ui.sh dashboard
```

It runs the `go-bindata` tool to package the generated `app` directory into `pkg/ui/data/dashboard/datafile.go`. It can also be used to package other user interface content, such as the Swagger documentation. Note: go-bindata can be installed with `go get github.com/jteeuwen/go-bindata/...`.

Then, run `make kube-ui` in the `cluster/addons/kube-ui/image` directory to build a new `kube-ui` binary that includes the updated `datafile.go`. When the updated UI is ready for release, increment the version tag in `cluster/addons/kube-ui/image/Makefile` and run `make push` in the same directory to build & push the new kube-ui docker image.

### Serving the app in production
The app is served in production by the `kube-ui` binary at:

```
https://<kubernetes-master>/ui/
```

which redirects to:

```
https://<kubernetes-master>/api/v1/proxy/namespaces/kube-system/services/kube-ui/
```

## Configuration
### Configuration settings
A json file can be used by `gulp` to automatically create angular constants. This is useful for setting per environment variables such as api endpoints.

`www/master/shared/config/development.json` and `www/master/shared/config/production.json` are used for application wide  configuration in development and production, respectively.

* `www/master/shared/config/production.json` is kept under source control with default values for production.
* `www/master/shared/config/development.json` is not kept under source control. Each developer can create a local version of the file by copy, paste and rename from `www/master/shared/config/development.example.json`, which is kept under source control with default values for development.

The configuration files for the current build environment are compiled into the intermediary `www/master/shared/config/generated-config.js`, which is then compiled into `app.js`.

* Component configuration added to `www/master/components/<component name>/config/<environment>.json` is combined with the application wide configuration during the build.

The generated angular constant is named `ENV`. The shared configuration and component configurations each generate a nested object within it. For example:

```
www/master
├── shared/config/development.json
└── components
    ├── dashboard/config/development.json
    └── my_component/config/development.json
```
generates the following in `www/master/shared/config/generated-config.js`:

```
angular.module('kubernetesApp.config', [])
.constant('ENV', {
  '/': <www/master/shared/config/development.json>,
  'dashboard': <www/master/components/dashboard/config/development.json>,
  'my_component': <www/master/components/my_component/config/development.json>
});
```

### Kubernetes server configuration
**RECOMMENDED**: The Kubernetes api server does not enable CORS by default, so `kube-apiserver` must be started with `--cors-allowed-origins=http://<your
  host here>` or `--cors-allowed-origins=.*`.

**NOT RECOMMENDED**: If you don't want to/cannot restart the Kubernetes api server, you can start your browser with web security disabled. For example, you can [launch Chrome](http://www.chromium.org/developers/how-tos/run-chromium-with-flags) with flag `--disable-web-security`. Be careful not to visit untrusted web sites when running your browser in this mode.

## Building a new visualizer or component
See [master/components/README.md](master/components/README.md).

## Testing
Currently, the UI project includes both unit-testing with [Karma](http://karma-runner.github.io/0.12/index.html) and end-to-end testing with [Protractor](http://angular.github.io/protractor/#/).

### Unit testing with Karma
To run the existing Karma tests:

* Install the Karma CLI (Note: it needs to be installed globally, so the `sudo` below may be needed. The other Karma packages, such as `karma`, `karma-jasmine`, and `karma-chrome-launcher,` should be installed automatically by the build). 
 
```
sudo npm install -g karma-cli
```

* Edit the Karma configuration in `www/master/karma.config.js`, if necessary.
* Run the tests. The console should show the test results.

```
cd www/master
karma start karma.conf.js
```

To run new Karma tests for a component, put new `*.spec.js` files under the appropriate `www/master/components/**/test/modules/*` directories.

To test the chrome, put new `*.spec.js` files under the appropriate `www/master/test/modules/*` directories.

### End-to-end testing with Protractor
To run the existing Protractor tests:

* Install the CLIs.

```
sudo npm install -g protractor
```

* Edit the test configuration in `www/master/protractor/conf.js`, if necessary.
* Start the webdriver server.

```
sudo webdriver-manager start
```

* Start the application (see instructions above), running at port 8000.
* Run the tests. The console should show the test results.

```
cd www/master/protractor
protractor conf.js
```

To run new protractor tests for a component, put new `*.spec.js` files in the appropriate `www/master/components/**/protractor/*` directories.

To test the chrome, put new `*.spec.js` files under the `www/master/protractor/chrome` directory.

[![Analytics](https://kubernetes-site.appspot.com/UA-36037335-10/GitHub/www/README.md?pixel)]()