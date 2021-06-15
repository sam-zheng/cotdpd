# Karaf and Felix
Quote from Karaf doc: "Karaf is the modulith runtime, supporting a wide range of frameworks and technologies."

### A bit of Karaf internals
Internal implementation: a feature roughly corresponds to a `Subsystem` with each being a `Resource` which has `Requirement`s and `Capability`s.
When Karaf starts, it reads config from etc/org.apache.karaf.features.cfg, `featuresRepositories` and `featuresBoot` are extracted, repositories are read and processed recursively (through &lt;features&gt;/&lt;repository&gt;) to get all available features defined in all the repositories.

    BootFeaturesInstaller.installBootFeatures() -> FeaturesServiceImpl.installFeatures()
    
where boot features are passed in. in `FeaturesServiceImpl.doProvision`, `Deployer.deply()` gets called to do the deployment, where:

1. `SubsystemResolver.prepare` constructs a root `Subsystem` and adds all the boot features to the root `Subsystem` as its requirements, then call `root.build()` to build it, which, for each boot feature, creates a `Subsystem` as a child of the root `Subsystem`; then for the dependency features of the boot feature,  either creates a `Subsystem` in a hierachical tree structure or as a direct child of the root `Subsystem` depending on whether the feature was defined to accept dependencies(default false). after build, all dependency features starting from the boot features are populated as `Subsystem`s. here each `Subsystem` constructed with a `Feature` will have a requirement on the `Feature` passed in.
2. `SubsystemResolver.resolve` does the resolution, i.e. finds capabilities for requirements and connects resources (wiring):

`root.downloadBundles` downloads bundles defined in the features (Subsystems) and those bundles will be added as dependencies either of the defining `Subsystem` or of the root `Subsystem` depending on it's `dependency` attribute: 
    
    <bundle dependency="false" > will make the bundle (resource) depend on the Subsystem where it's defined
    <bundle dependency="true" > will make the bundle (resource) depend on the region's Subsystem, e.g. root

A `Resource` is created for each bundle downloaded.
A `Resource` is created for the feature of the Subsystem and a `Requirement` on the Subsystem is created and added to the `FeatureResource`.

bundle `Resource`s and the feature `Resource` of the `Subsystem` are added to `org.apache.karaf.features.internal.region.Subsystem.installable` (a set of `Resoruce`s), which will then be used in resolution.

3. `SubsystemResolveContext` is created with the root Subsystem and there the distances of root to all the `installable`s are computed, which will be used for finding capability provider later during resolution.
4. finally `Resolver.resolve(context)' is called with the context created above, 

### A bit of OSGi(Apache Felix implementation)
Core concepts:
1. A `Resource` has requirements and capabilities. during resolution of a resource, the requirements of the resource need to be statisfied by capatilities of other resources, if no qualified capabilities are provided by other resources, then the resolution fails.
2. A bundle defines requirements and capabilities, kind of like a resource (however, a `Bundle` represents a runtime dynamic object with lifecycle), the resolution of a bundle treats it as a `Resource`.
3. `Resolver` resolves the `Resource`s in the given `ResoveContext`, it starts with some root `Resource`s(e.g. `ResolveContext.getMandatoryResources`), recursively finds capabilities for the requirements defined by the resources and establishes the connections(`Wire`s) between the resources. see `org.apache.felix.resolver.ResolverImpl.doResolve`(returning a Map&lt;Resource, List&lt;Wire&gt;&gt;, i.e. all the resources and their corresponding required dependencies with each wire representing a resolved requirement of the resource, each resource in the map is required directly or indirectly starting from the root resource.), `org.apache.felix.resolver.ResolverImpl.getInitialCandidates`, `org.apache.felix.resolver.Candidates.populate`.

A bit of `Felix`:
1. `FrameworkFactory` creates a `Felix` with configuration properties.
2. `Felix.init()` initializes the framework, e.g. start the event dispatching thread(`EventDispatcher`), create `BundleCache` (s. property `org.osgi.framework.storage`), etc.
3. `Felix.start()` starts the framework, this will, among other things, start the start level thread which handles start level changes(i.e. start/stop bundles according to requested start level). after the framework is started, bundles can be installed with `Felix.getBundleContext()`, and there the ball starts rolling.

Some important bundles:
1. org.apache.felix.fileinstall - automatically install bundles & config files(`ConfigInstaller` using `ConfigurationAdmin`).
2. org.apache.felix.configadmin - implements auto configuration.
3. org.apache.felix.scr - implements declarative services (see OSGi component spec).

### Conclusion
A Karaf feature is a group of bundles constituting an application/module, Karaf eases the configuration/management of OSGi applications - my understanding.

