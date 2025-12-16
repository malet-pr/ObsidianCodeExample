Kryo is a fast and efficient **binary object graph serialization framework for Java**. It is widely used in big data systems like Apache Spark, Apache Flink, and Apache Hive to provide high speed, a small serialized data size, and an easy-to-use API for persisting and transmitting objects. 

Key Features and Usage

- **Speed and Efficiency**: Kryo is significantly faster and more compact than standard Java serialization (often by 10x or more), making it ideal for high-performance, memory-intensive applications.
- **Simple API**: The core API uses `Kryo` for managing the process, and `Input` and `Output` streams to handle bytes.
- **Automatic Copying/Cloning**: Kryo can perform automatic deep or shallow copies of objects directly, without the object-to-bytes-to-object conversion.
- **No-Arg Constructors**: By default, Kryo requires classes to have a no-argument constructor to create instances during deserialization.
- **Thread Safety**: The `Kryo` instance is not thread-safe; it is recommended to use a `ThreadLocal` or a pool to manage instances in a multi-threaded environment. 

Registration (Optional but Recommended)

For best performance, you should register the classes you plan to serialize. 

- **With Registration**: Classes are assigned a small integer ID (usually 1-2 bytes), which results in a smaller serialized size and faster performance. This requires the registration order to be the same during serialization and deserialization.
- **Without Registration**: Kryo can automatically serialize unregistered classes, but it writes the full class name string into the byte stream, which is less efficient and increases the data size. 

Example Usage (Java)

To use Kryo, add the dependency to your project and follow these basic steps:

```java title:"Kryo Serialization" fold
import com.esotericsoftware.kryo.Kryo;
import com.esotericsoftware.kryo.io.Input;
import com.esotericsoftware.kryo.io.Output;
import java.io.*;

// Make sure your class has a no-arg constructor
public class SomeClass {
    public String value;
    public SomeClass() {}
    public SomeClass(String value) { this.value = value; }
}

// Serialization and Deserialization
Kryo kryo = new Kryo();
kryo.register(SomeClass.class); // Register the class

SomeClass object1 = new SomeClass("Hello Kryo!");
ByteArrayOutputStream stream = new ByteArrayOutputStream();
Output output = new Output(stream);
kryo.writeObject(output, object1);
output.close();
byte[] serializedData = stream.toByteArray();

Input input = new Input(new ByteArrayInputStream(serializedData));
SomeClass object2 = kryo.readObject(input, SomeClass.class);
input.close();
// object2.value will be "Hello Kryo!"
```

Compatibility and Class Evolution

Kryo offers different serializers to handle class evolution (adding/removing fields) over time: 

- **`FieldSerializer`**: The default and fastest option, but it does not support changes to fields without potentially invalidating previously serialized bytes.
- **`CompatibleFieldSerializer`**: Provides forward and backward compatibility by writing a simple schema the first time a class is encountered (at a small performance cost).
- **`TaggedFieldSerializer`**: Only serializes fields with a `@Tag` annotation, allowing fields to be renamed or deprecated without breaking compatibility. 

For more documentation and examples, refer to the [Kryo GitHub repository](https://github.com/EsotericSoftware/kryo).
Other links: 
- Baeldung: https://www.baeldung.com/kryo
- Medium 1: https://medium.com/@prekshaan95/kryo-serialization-and-deserialization-in-java-9e6da25e2a76
- Pitfalls 1: https://blog.lunatech.com/posts/2022-01-03-kryo-pitfalls
- 