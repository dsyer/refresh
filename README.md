## EnvironmentManager and EnvironmentManagerEndpoint

The `EnvironmentManager` adds a `PropertySource` with high priority to the Spring `Environment` (so
properties set there will take precedence). If you have the Actuator and are in a webapp then the `EnvironmentManagerEndpoint` 
will expose "/env" with POST for updating the `EnvironmentManager` remotely. E.g. run the app locally:

```
@Configuration
@EnableAutoConfiguration
public class Application {
	public static void main(String[] args) {
		SpringApplication.run(Application.class, args);
	}
}
```

then interact with the `Environment`:

```
$ curl localhost:8080/env/message
Hello
$ curl localhost:8080/env -d message=Foo
$ curl localhost:8080/env/message
Foo
```

Doing this (or the equivalent operation over JMX) will trigger an event
that leads to rebinding of `@ConfigurationProperties` (see below). 

## ConfigurationPropertiesRebinder

Spring beans that are `@ConfigurationProperties` can be rebound if the `Environment` changes. Example:

```
	@ConfigurationProperties
	protected static class ServiceProperties {
		private String message;
		public String getMessage() {
			return message;
		}
		public void setMessage(String message) {
			this.message = message;
		}
	}
```

then inject `ServiceProperties` into an application component using `@Autowired` and use it to govern the behaviour.
When the `Environment` changes do this

```
    @Autowired
    private ConfigurationPropertiesRebinder rebinder;
    
    ...
    
		rebinder.rebind();

```

The `ConfigurationPropertiesRebinder` is also a `@ManagedResource` so you can ping the `rebind()` operation remotely.

## RefreshScope

Spring beans in scope="refresh" are re-initialized on the next method call after a refresh. Example:

```
	@Configuration
	@EnableConfigurationProperties(TestProperties.class)
	@EnableAutoConfiguration
	protected static class TestConfiguration {
		
		@Autowired
		private TestProperties properties;
		
		@Bean
		@RefreshScope
		public ExampleService service() {
			ExampleService service = new ExampleService();
			service.setMessage(properties.getMessage());
			service.setDelay(properties.getDelay());
			return service;
		}
		
	}
```

then update the `TestProperties` (e.g. via JMX or something), and

```
	@Autowired
	private RefreshScope scope;

...

		// ...and then refresh, so the bean is re-initialized:
		scope.refresh("service");
```