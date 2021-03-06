# Adding custom (de)serializers to Jackson's ObjectMapper

Sometimes the behavior of the Spring Data REST's ObjectMapper, which has been specially configured to use intelligent serializers that can turn domain objects into links and back again, may not handle your domain model correctly. There are so many ways one can structure your data that you may find your own domain model isn't being translated to JSON correctly. It's also sometimes not practical in these cases to try and support a complex domain model in a generic way. Sometimes, depending on the complexity, it's not even possible to offer a generic solution.

So to accommodate the largest percentage of the use cases, Spring Data REST tries very hard to render your object graph correctly. It will try and serialize unmanaged beans as normal POJOs and it will try and create links to managed beans where that's necessary. But if your domain model doesn't easily lend itself to reading or writing plain JSON, you may want to configure Jackson's ObjectMapper with your own custom type mappings and (de)serializers.

### Abstract class registration

One key configuration point you might need to hook into is when you're using an abstract class (or an interface) in your domain model. Jackson won't know by default what implementation to create for an interface. Take the following example:

		@Entity
		public class MyEntity {

			@OneToMany
			private List<MyInterface> interfaces;

		}

In a default configuration, Jackson has no idea what class to instantiate when POSTing new data to the exporter. This is something you'll need to tell Jackson either through an annotation, or, more cleanly, by registering a type mapping using a [Module](http://wiki.fasterxml.com/JacksonFeatureModules).

Any `Module` bean declared within the scope of your `ApplicationContext` will be picked up by the exporter and registered with its `ObjectMapper`. To add this special abstract class type mapping, create a `Module` bean and in the `setupModule` method, add an appropriate `TypeResolver`:

		public class MyCustomModule extends SimpleModule {

			private MyCustomModule() {
				super("MyCustomModule", new Version(1, 0, 0, "SNAPSHOT"));
			}

			@Override public void setupModule(SetupContext context) {
				context.addAbstractTypeResolver(
						new SimpleAbstractTypeResolver().addMapping(MyInterface.class, MyInterfaceImpl.class)
				);
			}

		}

Once you have access to the `SetupContext` object in your `Module`, you can do all sorts of cool things to configure Jackon's JSON mapping. You can read more about how `Module`s work on Jackson's wiki: [http://wiki.fasterxml.com/JacksonFeatureModules](http://wiki.fasterxml.com/JacksonFeatureModules)

### Adding custom serializers for domain types

If you want to (de)serialize a domain type in a special way, you can register your own implementations with Jackson's `ObjectMapper` and the Spring Data REST exporter will transparently handle those domain objects correctly.

To add serializers, from your `setupModule` method implementation, do something like the following:

		@Override public void setupModule(SetupContext context) {
			SimpleSerializers serializers = new SimpleSerializers();
			SimpleDeserializers deserializers = new SimpleDeserializers();

			serializers.addSerializer(MyEntity.class, new MyEntitySerializer());
			deserializers.addDeserializer(MyEntity.class, mew MyEntityDeserializer());

			context.addSerializers(serializers);
			context.addDeserializers(deserializers);
		}

Now Spring Data REST will correctly handle your domain objects in case they are too complex for the 80% generic use case that Spring Data REST tries to cover.