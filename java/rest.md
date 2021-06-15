# Java RESTful API with OSGi
1. osgi whiteboard extender(pax-web-extender-whiteboard) tracks servlets & resources and registers with HttpService:
WebApplication org.ops4j.pax.web.extender.whiteboard.internal.ExtenderContext.getWebApplication(Bundle bundle, String httpContextId, Boolean sharedHttpContext)
HttpServiceTracker
ServletTracker
ResourceTracer
ServletContextHelperTracker
pax-web-runtime provides HttpService implementation

2. jaxrs osgi connector:
com.eclipsesource.jaxrs.publisher.internal.ServletContainerBridge publishes resources according to configuration (RootApplication) to the Jersey REST implementation which then extracts resources annotated (e.g. with Path.class) and registers them with the application configuration (ApplicationHandler, org.glassfish.jersey.server.ServerRuntime). com.eclipsesource.jaxrs.publisher.internal.ServletContainerBridge registers itself with the OSGi framework, ServletTracker of 1. registers it with the HttpService and there makes it available to the web server (e.g. Jetty).
ServletContainerBridge.service() -> (Jersey)ServletContainer.service() -> (Jersey) method dispatching -> method of the class (e.g. annotated with Path.class) 

3. Jersey:
ServletContainer
org.glassfish.jersey.server.model.AnnotatedMethod
Object AbstractJavaResourceMethodDispatcher.invoke(ContainerRequest containerRequest, Object resource, Object... args) throws ProcessingException

PathMatchingRouter
MethodSelector MethodSelectingRouter.selectMethod(List<AcceptableMediaType> acceptableMediaTypes, List<ConsumesProducesAcceptor> satisfyingAcceptors, MediaType effectiveContentType, boolean singleInvokableMethod)