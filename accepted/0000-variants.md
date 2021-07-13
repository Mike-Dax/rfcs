- Start Date: 2021-07-13
- RFC PR: 
- Yarn Issue: 



# Summary

Native module installation is complicated, with package maintainers having various competing options for delivering their native modules. Consumers of these modules have little choice in how they're pulled, often resorting to build scripts.

A proposed solution lets maintainers publish 'meta-packages' that describe where to find a specific implementation for a given set of parameters, defined by a template string. Given matching parameters, the package is replaced (at the same version).

This proposal should include a 'recommended' native module publishing method that produces the best case scenario for the majority of consumers, including those using other package managers.

# Motivation

We have native modules that can't be distributed via WASM, native modules are required. We would like to provide pre-built artifacts instead of requiring compilation on the end user systems.

A unified method of native module publishing with the following high level goals:

1. Native artifacts are stored in the registry, not out of band in GitHub Releases, S3, etc.

2. Package and download only the minimum artifacts for any given operating system / platform.
3. Statically link the native modules.
4. Fetching these artifacts is the responsibility of the package manager.
5. Some level of zero-install-ish support.

### Native artifacts are stored in the registry.

Storage of artifacts in the registry results in fewer points of failure. 

GitHub isn't a particularly good CDN. 

We've had outright failures to install due to GitHub CDN failures in the past when NPM has been functioning fine, and frequently have slow downloads from GitHub's CDN.

### Package the minimum artifacts for any given operating system / platform.

Packaging every combination of supported platforms into the npm release results in a combinatorial explosion of bandwidth required for a singular download. Ideally, only the artifacts required are fetched. 

We had one instance where a Golang native dependency, when compiled for 32bit windows, was incorrectly identified as a Trojan by Windows Defender on 64bit systems.

One of our dependencies tries to pull in 100MB of unnecessary native modules per download as a result of their chosen prebuild solution.

### Statically link the native modules.

The bundler of choice should be able to statically analyse the dependency tree, including the native module resolution. Webpack for example should be able to take the file and treat it like any other asset, giving it a hash, storing it as per its ruleset.

I often see Electron apps that use bundlers marking the entire native module package as `external` , resulting in the distribution of significant unnecessary files, such as the source code, docs, and often due to violating other goals, the native dependencies for other operating systems.

### Fetching responsibility

Running build scripts takes a long time, and is prone to failure. The package manager has all the information it needs to do this job, and in packages with native modules, the native bit is part of the package and should also be fetched.

### Zero install support

It is valuable to be able to copy the cache directory and be sure that later, the application will be able to build and run. 

We pass the yarn cache around on CI to speed up our pipelines, but due to our use of native modules, we run `yarn install` to run our native module fetching plugins.

Zero install support is essentially at odds with static linking, so a compromise will probably need to be found.

## Prior art

[Prebuild](https://github.com/prebuild/prebuild) uploads artifacts out of band, then downloads them with an install script. Only what's needed is pulled. The artifacts aren't statically linked. This solution requires build scripts.

[Prebuildify](https://github.com/prebuild/prebuildify) uploads all artifacts into the same npm release. It's in the registry but the artifacts are very large. The artifacts aren't statically linked. This solution works without build scripts. This solution is supported in zero-install situations.

[napi-rs](https://github.com/napi-rs/package-template) uploads a package per combination of platform and arch supported by their matrix to the registry. When downloading with package managers that support the `os` and `cpu` compatibility guards, only what's necessary is downloaded. On Yarn Berry, the entire matrix is downloaded due to these guards not being implemented. The artifacts aren't statically linked. This solution works without build scripts. This solution is supported in zero-install situations technically, since Yarn Berry downloads the whole matrix.

[esbuild](https://github.com/evanw/esbuild) publishes releases to the registry, but fetches them with a build script (that forks and executes an `npm install`, falling back to a direct HTTP request). Only what's needed is pulled. The artifacts are statically linked. This solution requires build scripts.

[Electron](https://github.com/electron/get) publishes releases as Github artifacts, fetching them with a build script. 

Our related prototypes:

[yarn-redirect-app-builder](https://github.com/electricui/yarn-redirect-app-builder) is a plugin we wrote to handle one of our native dependencies, `app-builder`, an Electron build chain utility. We publish to our registry a package per combination of platform and arch. The plugin redirects the request and statically installs the native dependency. Since it's a yarn plugin, no build scripts are required.

[yarn-prebuilds](https://github.com/electricui/yarn-prebuilds) is a plugin that rewrites dependencies on the [bindings](https://github.com/TooTallNate/node-bindings) package, fetching the native module from the prebuild-compatible repository and statically linking it in place of the bindings package. It works well enough but the config is scope driven, and different packages have different prebuild mirrors with different formats that can't all be encapsulated in a singular pattern. Since it's a yarn plugin, no build scripts are required.

## Our User Story

We have a monorepo that contains various packages, including Electron applications and templates, and a [hardware-in-the-loop testing framework](https://electricui.com/blog/hardware-testing). (You may find the [logging familiar](https://asciinema.org/a/392751), thank you!)

The yarn-prebuilds plugin is used to fetch prebuilds for our native modules. Install scripts aren't allowed in the monorepo (we have other tooling to grab the correct Electron builds). The testing frameworks have manual operations required to pull in the native modules for the correct Node ABI, since the majority of the time we have the ones for our Electron build pulled in.

We'd like a solution that's aware of which 'context' the dependency was required in. The Electron templates should pull in the Electron native builds, the tests should pull in the relevant Node native builds.



The Electron ABIs are non-trivial to determine, it doesn't seem like a good fit to me to include something like [node-abi](https://github.com/lgeiger/node-abi) in Yarn itself, it seems a better fit for that functionality to be in a plugin.

# Detailed Design

Arcanis has a gist of a general variants idea:

https://gist.github.com/arcanis/df6da3fa63e38bf52e4af1136529d9c7



The general idea is that the published package contains a field in the package.json that provides static information about variants of the package that may replace it, given certain parameters being met. These parameters can be set by Yarn plugins, default plugins would provide environment parameters such as `platform`, `arch`, etc.



I think the ideal publishing strategy is similar to the one employed by napi-rs. Every combination of packages is published to the registry with the appropriate `os` and `arch` compatibility guards. 

On 'variant' compatible package managers, the meta-package is _replaced_ by the correct variant. It would statically expose its native module. 

On non-variant compatible package managers, the meta package would perform dynamic linking to point at the correct native module.

If a package manager believes strongly in any of the alternative methods employed in the prior art section, they can do them in their meta-package, then variants can take over if they're supported.

If I as a consumer of packages believes strongly in this variants approach, in some cases I can use packageExtensions to annotate my dependencies to fetch only what I need, assuming their artifacts are published to the registry.



Variants store their artifacts as other registry packages, only download exactly what's required, statically link by replacing the package, and are fetched by the package manager instead of a build install script.

### Variant format

An array of objects with keys of `pattern`, `matrix` and `exclude`, let's call them variant objects. The `pattern` defines the template string to replace parameters within. The `matrix` defines the possibilities of the parameters supported. The `exclude` defines any specific combinations that aren't allowed.

Multiple variant objects can be defined to support different patterns. The `matrix` and `exclude` keys are optional.

A final fallback can be set with an object with only a `pattern` key.

```json
"variants": [
  {
    "pattern": "sled-%platform-%napi",
    "matrix": {
      "platform": {
        "candidates": [
          "darwin",
          "win32",
          "linux"
        ]
      },
      "napi": {
        "candidates": [
          "5",
          "6"
        ]
      }
    },
    "exclude": [
      {
        "platform": "win32",
        "napi": 5
      }
    ]
  },
  {
    "pattern": "sled-%platform",
    "matrix": {
      "platform": {
        "candidates": [
          "wasm"
        ]
      }
    }
  },
  {
    "pattern": "sled-build-sources"
  }
]
```



### Parameters are cascaded (by plugins only?)

A `reduceVariantStartingParameters` hook lets plugins set keys that are propagated per workspace. These would be used to set things based on the Node runtime, for example the platform, arch, etc.

A `reduceVariantParameters` hook lets plugins set keys from a package 'down', inclusive, based on the packages' dependencies. This would be useful for an Electron plugin that sets the runtime to Electron, and the node ABI version appropriately if a package depends on Electron. This kind of thing feels best left as a plugin, since something like `node-abi` would have to be pulled in to fulfil that.

A `reduceVariantParameterComparators` hook lets plugins define compatibility relationships between values of the parameter keys. For example, Node is backwards compatible with NAPI versions, but not ABI versions for non NAPI packages. This hook lets an `napi` key be backwards compatible.



Parameters set by these would cascade to dependents implicitly. I'm not sure whether to allow cascading parameters, set in the workspace package.json file. That would not be required by our use case, given our use cases' requirement for a plugin to provide information.

### Parameters can be set explicitly for certain packages with dependenciesMeta

These parameters do not cascade, and only affect the specified dependency.

```json
"dependenciesMeta": {
  "esbuild": {
    "parameters": {
      "platform": "wasm"
    }
  }
}
```



### Variant information can be set as package extensions

For example, replacing `app-builder-bin` with our variant-compatible packages.

```yaml
packageExtensions:
  "app-builder-bin@*":
    variants: 
    - pattern: "@electricui/app-builder-bin-%platform-%arch"
      matrix: 
        platform:
          candidates: 
            - darwin
            - linux
            - win32
        arch:
          candidates:
            - x64
```



# How We Teach This

Ideally this can transparently happen in the background for most native modules and "just work", without end users needing to change their setups at all. There's a very fractured ecosystem around native modules, especially around Electron and the use of native modules, ideally that can be simplified to the point of not needing to worry about it at all.

For native module package authors, ideally this provides a unified recommended method for package deployment that covers the majority of use cases for all package managers, our specific use case will likely require an additional yarn plugin, which unfortunately won't be portable to other package managers.

This would add additional `packageExtension` and `dependenciesMeta` options for users wanting to 'convert' existing native modules to use the variants system.

There would be several additional plugin hooks, these are mostly documented within the codebase at the moment.



# Drawbacks

This seems like quite a powerful feature that could be (ab)used for other use cases. I'd certainly like to hear comments for other use cases that can abuse this proposal to see if there are other edges.

For example, packages could be published with or without docs, with differing typescript types [depending on the API contract](https://api-extractor.com/) desired.

It's throwing another 'standard' into a sea of standards.



# Alternatives

### Implementing `os` and `arch` compatibility guards

This would allow solutions like napi-rs to work without pulling every dependency. Ideally this would be controllable via plugins. Perhaps this should be done anyway?

It doesn't solve our use case for fetching Electron compiled modules in one workspace and Node compiled modules in another.

### Implementation in plugin space

With additional hooks or manifest information exposed in the `reduceDependency` hook, it might be possible to implement this in plugin-space?

As far as I know it's not possible to fetch the entire dependency graph from the perspective of a plugin, since it only exists transiently within the package resolution stages.

# Unresolved questions

Should users be allowed to cascade parameters from their workspace down with configuration in their workspace package.json?

```
"cascadeVariantParameters": {
  "platform": "wasm"
}
```

I'd like to hear use cases for this specific feature, it feels a little dangerous.

---

Should the variants config have the `candidates` key or go straight from the parameter key to the candidates array? Without the candidates key:

```yaml
// Yaml
packageExtensions:
  "app-builder-bin@*":
    variants: 
    - pattern: "@electricui/app-builder-bin-%platform-%arch"
      matrix: 
        platform:
          - darwin
          - linux
          - win32
        arch:
          - x64
```

```json
// JSON
"variants": {
  "pattern": "@electricui/app-builder-bin-%platform-%arch",
  "matrix": {
    "platform": ["darwin", "linux", "win32"],
    "arch": ["x64"],
  }
}
```

I can see the argument for having the candidate key in JSON as it makes it clearer what it is.

---

Perhaps there should also be an optional version pattern, allowing for arbitrary redirection to say, a `tgz` url?

I think there might be room for an evolution of the yarn-prebuilds plugin where this variants system redirects to an alternative resolver that grabs a prebuild compatible link and statically requires the native module? 

---

I don't know how to deal with the `workspace` resolver, since the version is a folder, and this concept relies on the package being the same version. 

Ideally mono-repos containing native modules can also use this for internal resolution.

---

### Zero Installs

I'd like a feature where yarn can be instructed to pull every variant possible in the matrix(es), the resulting yarn cache is portable, and with an `install` step, can be used on any supported OS without network requirements.

I'm not familiar enough with PnP to know if it's possible to move some of this variant logic into the hook itself? Perhaps that's a goal for a future PR after an initial one?

### PR requests

I'd like to at least prototype the method described in this RFC so I'd love some guidance on some technical details in addition to general comments.

The WIP is available here:

https://github.com/Mike-Dax/berry/pull/1/files



What's the most idiomatic way to 'replace' a package?

Where should this variant information be stored? Does it need to be in the yarn.lock?

Is there an idiomatic way to get the manifest from a package?