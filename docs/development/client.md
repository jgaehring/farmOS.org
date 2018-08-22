# farmOS Client / Native app

Work has begun on a simplified farmOS app that works offline in both browser
and native app form. It will be installable on Android and iOS devices. The
goal is to create a fast and focused client app for day-to-day and in-the-field
record keeping that stores data locally for offline use, and syncs back to a
farmOS server when internet access is available.

The application is divided into two projects:

1. farmOS Client: [https://github.com/farmOS/farmOS-client]
2. farmOS Native: [https://github.com/farmOS/farmOS-native]

The "client" is where all of the UI and features are built. The "native"
project wraps that client inside a native app, which can be installed on
Android/iOS.

**This documentation is not complete. It is intended only for informational
purposes at this time.**

## Architecture
The client essentially represents a UI library; the native repository consumes 
that library, while also providing the client with the dependencies it needs to 
be able to run on native devices, via Cordova. Those dependencies are the data 
and login plugins, which perform data and authentication operations specific to 
the native environment. The client library is separated from the native 
implementation in this way is so that the client library can also be consumed 
by other, browser-based implementations, such as Drupal modules, or standalone 
single page applications.

### farmos-native
The native repo currently performs two main functions:

- housing the data and login plugins
- controlling the Cordova build process

The plugins are already quite self-contained. It seems like a forgone conclusion 
that at some point we'll want to make them into their own independent libraries. 
They're actually Vue plugins, which is all that's preventing all Vue-related 
dependencies (and devDependencies) from being removed from the native repo, so 
that time might need come sooner rather than later. Anticipating that fact, the 
data and login plugins are outlined separately below.

Putting the plugins aside for now then, the native repo really becomes quite a 
spare repository, basically comprised of the following:

- the configuration files and build scripts for npm, Webpack and Cordova
- a mostly empty `index.html`, which just serves as a target for the build 
scripts
- a `main.js` file, which only calls the client library and passes it the 
plugins as arguments

Being concerned solely with the final build process, it really doesn't contain 
much essential code of its own, only serving to bring all the necessary 
dependencies and scripts together in one place. Then all it has to do is call 
the client library's load function and pass in the dependencies it needs. It's 
conceivable that the whole thing could be replaced with a Makefile.

### data plugin
The data plugin supplies the client with methods for storing farmOS data on 
local disk with WebSQL, and for sending that data to a farmOS server via AJAX 
and the farmOS REST API.

Specifically, the data plugin is a [Vue plugin], which implements

- Vuex subscribe methods, which 
    - listen for UI actions and mutations, then 
    - dispatch actions to the db and http modules;
- the db Vuex module, consisting of Vuex actions, which implement
    - WebSQL transactions, then
    - dispatches actions to the UI's store (eg, to update `logs`' "cached" 
    status);
- the http Vuex module, consisting of Vuex actions, which implement
    - AJAX requests, then
    - dispatches actions to UI's store (eg, to update `logs`' "synced" status).
  
### login plugin
I need to finish the login plugin's documentation once I've gone through and 
figured out the authentication process better, and fixed some issues with the 
way it currently uses Vue mixins, but structurally it's pretty similar to the 
data plugin. 

The one crucial difference is that it also registers a Vue component, 
`Login.vue`, on the main Vue instance when it's installed by the client. There's 
a lot that's not ideal about the login plugin, but how it registers the Vue 
component could actually be a good model for how other components could be added 
dynamically to the client's core library, if we wanted to break up the UI itself 
into separate modules.

### farmos-client
Compared to the native repo, the client repo has a lot more going on. The 
primary organizing principle for all this, currently, is the Vue framework 
itself.

The load function, found in `src/app.js`, is the main entry point. It is 
basically a thin wrapper for instantiating the main Vue object, and installing 
any Vue plugins that were passed in as dependencies. It's only a few lines, so 
perhaps it's easiest to illustrate how it works by including it here in its 
entirety:

```js
export default (data, login) => {
  Vue.use(data, {store, router})
  if (typeof login !== 'undefined') {
    Vue.use(login, {router, store})
  }
  return new Vue({
    el: '#app',
    store,
    router,
    components: {App},
    template: '<App/>'
  })
};
```

Note that the login plugin is optional. Crucially, when the plugins are 
installed, they must be passed the Vuex store and Vue router. Also, the DOM 
selector `#app` is hardcoded here as the mount point for the Vue application. 
In the future, this selector should certainly be parameterized to allow more 
flexibility. 

Beyond the load function, the client can be summarized as comprising:

- Vue components, which implement
    - Rendering algorithms, based on current state of the Vuex store
    - Component methods, which dispatch actions/mutations to the store when DOM 
    events are triggered
- The Vuex store, which implements 
    - a state tree, representing the entire UI state (eg, an array of log 
      objects)
    - mutations, which transform the UI state (synchronously)
    - actions, which 
        - handle asynchronous requests from the data plugin and components, and 
        - 'commits' mutations to the store at different stages of those requests

## A Path Towards an Improved Architecture
A sensible course for the future might be to decouple Vue from the client architecture entirely, so that neither the main load function nor its parameterized plugins depend on it. This would free future developers to use the framework of their chosing, or forgo frameworks entirely.

Beyond merely allowing for greater flexibility with the frontend stack, generalizing the client architecture could encourage greater overall modularity within the project. Instead of one monolithic UI, the individual pages or components could be pulled in selectively when an implementation, such as farmos-native, decides to compile them with their corresponding plugins. If these UI modules corresponded to specific farmOS modules, they could even be bundled into proper Drupal modules themselves, either to extend the existing modules (eg, the offline module) or to provide an alternative interface (eg, quick forms).

Another great advantage of modularizing the client in this way is to optimize the bundle size required for each pageload. Modern [code-splitting](https://webpack.js.org/guides/code-splitting/) techniques are impossible with the current architecture, because the client's Webpack configuration outputs the library as one big chunk, `dist/index.js`, bundling together all the vendor libraries, UI components and even the CSS as one file. For the time being, while our focus is restricted to the Observations module and the native implementation, this is not much of a problem, since the bundle size is still relatively small, and ultimately it will only need to be downloaded once from the Apple or Play stores. However, as the project grows, and browswer-based implementations are developed, this strategy will likely become untenable.

Before splitting everything up into a bunch of smaller libraries, I think we should seriously consider moving everything back into one "monorepository". There are [some compelling arguments](https://medium.com/@maoberlehner/monorepos-in-the-wild-33c6eb246cb9) for using monorepos in cases much like our own. The maintainers of the Babel JS compiler have actually extracted the tool they use for managing their own monorepo into its own library, called [Lerna](https://lernajs.io/), which has become quite popular. The Babel compiler is well known for its highly extensible nature, [being composed almost entirely of plugins](https://github.com/babel/babel/tree/master/packages#woah-whats-going-on-here). If we are in fact looking to go a similar route, Lerna seems like a well-suited and battle-tested tool for achieving that kind of extensibility. It would just make development and code sharing a lot easier, obviating the need to rely on npm-link to work on dependent libraries.

### Characteristics of the client architecture
- a run or load function that takes the shell, drivers and modules as parameters
- an app shell or main function, which provides a basic UI (eg, what gets mounted by the DOM driver)
- drivers, which provide an interface to various hardware and browser API's, including the DOM
- modules or plugins that add features to the shell; each provides a UI for a subset of farmOS data
- a component library of smaller UI elements which can be shared between modules
- an installer (client-side) and module delivery service (server-side) for transporting additional modules and drivers over a network (like the proposed native modules)

### Characteristics of the client module delivery system
- a manifest of client modules, which is generated, stored and updated by the server whenever corresponding farmOS modules are added
- a means of querying the server's manifest for info about the installed modules and their static assets
- a means of retrieving and caching those assets, which each module requires (specifically html, css and js files), whenever the server updates the manifest
- the ability to render the static html and css and evaluate the JavaScript files securely from their cached locations, as seamlessly as any other app component 

My personal wishlist for, or some happy byproducts of, the new architecture:
- easily decoupled from Vue, Bootstrap and any other frameworks
- static assets and vendor libraries can be more efficiently bundled and shared between modules
- quick forms: provide interfaces for subsets of the farmOS data that don't strictly correlate to one farmOS module or another

### Libraries contained in a client monorepo
Below is one potential way of organizing a monorepository's directories and libraries:

- Core
    - Run function
    - App shell (ie, `main`)
    - Installer (for additional modules)
- Drivers
    - DOM
    - HTTP (via `fetch`, not `$.http`)
    - WebSQL
    - IDB
    - Cordova Camera
    - Browser Camera
    - OAuth
- Modules
    - Observations
    - Grazing
    - Plantings
    - Seedlings
    - etc.
- Utils
    - logFactory
    - time & date utils (to replace Moment)
- Assets
    - logos, icons & shared images
    - main CSS, swatches, etc
- Component library (maybe included in Assets?)
    - Dropdown menus
    - Forms
    - etc
- Scripts
    - Integration Tests
    - Shared Scripts (by core, drivers, modules, etc)
        - common scripts & config's
        - Node/npm scripts
        - Webpack config files
    - Builds (for specific implementations)
        - Node/npm/Webpack scripts & config
        - shell scripts & Makefiles
        - build scripts for Drupal modules?

#### Core
The core library would primarily consist of an expanded load function ...

#### Drivers

The drivers, as I'm calling them, are more generalized versions of the Vue plugins we're currently using, such as the data plugin. I'm stealing this term and some of its conceptual underpinnings from [Cycle.js drivers](https://cycle.js.org/drivers.html), although there are some differences and I'm not proposing we use actual Cycle drivers. Fundamentally, the idea is to isolate side effects so that the main business logic of one's application can remain agnostic to the specific implementation details*. For our purposes, these side effects almost invariably represent some kind of I/O operation that is specific to the hardware or platform of the client implementation.

Ideally, most drivers would handle just one external API, such as WebSQL or XMLHttpRequest, both of which are currently handled by the data plugin, but should probably be handled by separate plugins/drivers. The WebSQL driver, thus abstracted, would not have to be limited to the Cordova implementation, but could also be used for a browser-based implementation for local persisteance, if it was so desired.

Other API's that could have their own drivers:

- localstorage
- IndexedDB
- Service Worker
- navigator.camera (for Cordova)
- navigator.MediaDevice (for browser-based camera access)
- navigator.geolocation (same API for Cordova & browsers)

Possibly, these drivers could also abstract specific aspects of the farmOS RESTful API, such as logs and assets, or even more specific endpoints like observations and plantings. However, I'm not sure if that's going too far with the abstraction.

\* = In the case of Cycle.js, this also means reducing the business logic to a single, reactive, pure function that takes a observable stream of input data and returns a stream of output data to and from the drivers, eliminating *all* side effects, including DOM manipulation, but I don't see any reason for us to be so absolutist.

#### Components/Modules
...

#### Clients
These would be platform specific implementations, such as the native client. While they wouldn't entail a lot of original code themselves, they would be the only consumers of the other libraries. Their main responsibility, as is the case for the native repo now, would be calling the load function from the core library, passing it any drivers it needed, and providing any build scripts and devDependencies necessary for the target platform, such as Cordova.

#### Scripts
This might not be necessary, but it could prove useful to provide some shared scripts that automate the development environments and build processes, since it's likely that many of the driver and component libraries will share similar build configurations. It's also possible that these scripts could supplant the need for having separate libraries for the client implementations themselves, since those implementations are mostly just concerned with scripting the build process.


## Development environments
Currently, a local clone of the farmOS-native repository is required to run a 
development environment for both the client and native repos; however, you do 
not need to clone the client repo separately if you're only planning to work on 
the native repo. So, follow the steps below to set up the native repo's dev 
environment first, regardless of which you're planning to work on, then proceed 
to setting up the client environment only if you wish to work on the client.

The first thing of course is to clone the native repo from GitHub, and install 
the npm dependencies:

```bash
$ git clone https://github.com/farmOS/farmOS-native.git
$ cd farmOS-native
$ npm install
```

You should now have all the dependencies you need to run the native app, either 
in the browser or via one of the platform SDK's. If you're main objective is to 
develop on the client, you'll probably want to do so in the browser. If you only 
wish to work on the native repo, it will have already downloaded its own copy of 
the client repo into its `node_modules` directory, along with all its other 
dependencies, when you ran `npm install`.

### Browser
The browser-based development environment can be started by running the 
following command from the project root:

```bash
$ npm start
```

This will start the Webpack devServer, which will compile a development build 
and serve it from [http://localhost:8080/]. Changes saved to the files in your 
local repo will be hot reloaded automatically while the devServer is running.

#### Proxying a farmOS Docker container
By default, when the Webpack devServer starts, it will set up a proxy service, 
which will route all AJAX requests and responses through `http://localhost:80`, 
in order to prevent CORS errors. This is assumed to be the address of a local 
farmOS development server running in a Docker container. This is probably the 
easiest way to get a development backend up and running, so if you don't already 
have a sever you can use for testing, see the [full instructions for setting up 
a farmOS Docker container].

If you wish to proxy an address other than `http://localhost:80`, you'll need to 
change the proxy settings in `farmOS-native/config/index.js`. Set 
`dev.proxy.target` to the address and port of your farmOS testing server. 
Restart the devServer and it should now be proxying your new address. It may 
also be necessary to add new endpoints to the `dev.proxy.context` array as new 
features are developed and new endpoints on the farmOS server need to be 
reached. For more information, see the [Webpack documentation on configuring the 
proxy middleware].

You will also have to install the Drupal [CORS module] on your farmOS server in 
order to handle the way the client does authentication (hopefully this will no 
longer be necessary once we implement OAuth). Once it's installed, go to the 
[CORS configuration page] and add the following line to the Domains field:

```
*|http://localhost:8080||Content-Type,Authorization,X-Requested-With|true
```

Finally, when logging in to the client from the browser, simply leave the URL 
field blank. The devServer will then interpret all requests as relative links 
and proxy them accordingly. For some reason login can sometimes fail on the 
first attempt, but should succeed on all subsequent attempts.

### Platform SDK's

* [Cordova docs on running the emulator and debugger in Android Studio]
* [Cordova docs on running the emulator and debugger in XCode]

[//]: <> (TODO: Add a few more details on this once I know more)

### Linking the farmOS-client repository
If you wish to work on the client repo in development, the first thing you'll 
want to do is clone it from GitHub and install its dependencies:

```bash
$ git clone https://github.com/farmOS/farmOS-native.git
$ cd farmOS-native
$ npm install
```

Next, you'll need to direct the native repo to use this local copy of the client 
library, instead of the copy it downloaded into its own `node_modules` 
directory. Fortunately, npm provides a helpful little cli tool for just this 
sort of occasion, [npm-link]. It basically just creates a symbolic link in place 
of the npm package in the consuming repository's `node_modules`. To use it, 
first navigate to client library to create a reference to your local copy of 
the client repo:

```bash
$ cd path/to/farmOS-client 
$ npm link
```

Now, navigate to your local copy of the native repo, to point it to your local 
copy of the client, and start the dev server there, if you haven't already:

```bash
$ cd path/to/farmOS-native
$ npm link farmos-client
$ npm start
```

Finally, you can navigate back again to the client repo, and run the dev script:

```bash
$ cd path/to/farmOS-client 
$ npm run dev
```

This will set a watcher on the `src/` directory, so that whenever you save a 
change to the client repo, Webpack will automatically rebuild the `dist/` 
folder, which will in turn trigger the native repo's dev server to hot reload, 
so you can see the changes in the browser.

## Native build process

The process for building the native applications for iOS and Android can be
broken down into 2 main steps: first, Webpack bundles all the HTML, CSS and
JavaScript modules into their final form that will run in [WebView]; secondly,
Cordova, with some help from the Android and iOS SDK's (Android Studio & XCode,
respectively), builds the full-blown native packages (.apk and .ipa files,
respectively).

To build from source, you'll need to clone the farmOS-native repository from
GitHub:

```bash
$ git clone https://github.com/farmOS/farmOS-native.git
```

Note that the farmOS-client is a dependency which will be automatically
installed as an npm package in the following step, so you do not need to clone
that repo.

Bundling the web assets with Webpack is fairly straight-forward. [Node]
(v.6.0.0 or higher) and npm (v.3.0.0 or higher) are the only system
requirements. Once Node is installed, you can install the necessary JavaScript
dependencies with npm, which is automatically included with Node. The npm
script `build-mobile` can then run Webpack to bundle the assets. All this can
be done by running the following two commands from the project's root
directory:

```bash
$ npm install
$ npm run build-mobile
```

This will generate all the necessary files within the `www` directory, which
Webpack will create if it doesn't already exist. Do not alter these files
directly. They are optimized for WebView, which is basically a browser that
Cordova installs and runs inside the native application itself to render all
the HTML, CSS and JavaScript source files. The same files are used by both iOS
and Android implementations

All the web assets are now ready for building the final native packages. Both
platforms will require installing Cordova globally, via npm:

```bash
$ npm install -g cordova
```

This, however, is where the process diverges into separate iOS and Android
builds. This next stage will probably comprise the most system configuration,
too, since it requires installing each platform's SDK (Software Development
Kit), if they aren't installed already. Of course, if you only intend to build
for one platform, you only need to install that platform's SDK and
corresponding system requirements; building for both platforms will require
installing both SDK's.

### Android Build

To configure your system to build the Android app, follow Cordova's
[Android Platform Guide]" and the [Android Studio installation guide]. This
will differ depending on your development machine's operating system, but Mac,
Windows and Linux are all supported.

**System Requirements:**

- Java 8
- Android Studio
- Gradle

Note that Android Studio recommends the official Oracle JDK for Java 8; using
OpenJDK may cause errors. Also make sure to follow Cordova's instructions for
setting up your system's [environment variables] and installing the
[SDK packages] for the Android versions the app will target. As of May 2018,
the latest version of Android that Cordova seems to support is 7.1.1, at API
Level 25.

[//]: <> (TODO: Determine what versions of Android the app should target and list them here.)

Once Android Studio is installed and configured, make sure Android has been
added to Cordova's list of platforms, then you're ready to run the final build
command:

```bash
$ cordova platform add android
$ cordova build android
```

By default, the `build` command will produce a debugging APK (equivalent to
running `cordova build android --debug`). If you want to build a final release
version, you'll need to add the `--release` flag, but you'll also need to use
your keys for signing the APK (see "[Signing an App]" in the Cordova docs for
more details). Both the debug and release APK's can then be found at
`path/to/farmos-native/platforms/android/app/build/outputs/apk` after building.

[//]: <> (TODO: Figure out signing the app for the Play Store and document here.)

### iOS Build

Only available on Apple OS X. Windows and Linux are not supported. For some
workarounds, see "[Developing an iOS App on Linux]"

Cordova's [iOS Platform Guide]

[https://github.com/farmOS/farmOS-client]: https://github.com/farmOS/farmOS-client
[https://github.com/farmOS/farmOS-native]: https://github.com/farmOS/farmOS-native
[Vue plugin]: https://vuejs.org/v2/guide/plugins.html
[http://localhost:8080/]: http://localhost:8080/
[full instructions for setting up a farmOS Docker container]: /development/docker/
[Webpack documentation on configuring the proxy middleware]: https://webpack.js.org/configuration/dev-server/#devserver-proxy
[CORS module]: https://www.drupal.org/project/cors
[CORS configuration page]: http://localhost/admin/config/services/cors
[Cordova docs on running the emulator and debugger in Android Studio]: https://cordova.apache.org/docs/en/latest/guide/platforms/android/index.html#debugging
[Cordova docs on running the emulator and debugger in XCode]: https://cordova.apache.org/docs/en/latest/guide/platforms/ios/index.html#debugging
[npm-link]: https://docs.npmjs.com/cli/link
[WebView]: https://cordova.apache.org/docs/en/latest/guide/hybrid/webviews/
[Node]: https://nodejs.org
[Android Platform Guide]: https://cordova.apache.org/docs/en/latest/guide/platforms/android/index.html
[Android Studio installation guide]: https://developer.android.com/studio/install
[environment variables]: https://cordova.apache.org/docs/en/latest/guide/platforms/android/index.html#setting-environment-variables
[SDK packages]: https://cordova.apache.org/docs/en/latest/guide/platforms/android/index.html#adding-sdk-packages
[Signing an App]: https://cordova.apache.org/docs/en/latest/guide/platforms/android/index.html#signing-an-app
[Developing an iOS App on Linux]: https://andrewmichaelsmith.com/2017/02/developing-an-ios-app-on-linux-in-2017/
[iOS Platform Guide]: https://cordova.apache.org/docs/en/latest/guide/platforms/ios/index.html

