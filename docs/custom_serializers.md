# Writing your own serializers

The most simple and straightforward way to create serializer is to write annotation `@Serializable`
directly on your class:

```kotlin
@Serializable
class MyData(val s: String)
```

In this case, compiler plugin will generate for you:

* `.serializer()` method on companion object to obtain serializer. If your class is
a generic class, this method will have arguments `KSerializer<T1>, KSerializer<T2>`..., where `T1, T2` - your generic type parameters.
* Special nested object in your class, which implements `KSerializer<MyData>`
* Methods `save`, `load` and `update` of interfaces `KSerialSaver` and `KSerialLoader`
* Descriptor property `serialClassDesc` of `KSerializer`

## Customizing

If you want to customize representation of the class, in most cases, you need to write your own `save`
and `load` methods. `update` method have default implementation of `throw UpdateNotSupportedException(serialClassDesc.name)`. Serial descriptor property typically used in generated version of those methods,
so you likely don't need it. 

You can write methods directly on companion object, annotate it with `@Serializer(forClass = ...)`, and serialization plugin will respect it as default serializer.
(note you still have to apply `@Serializable` annotation, because we need to define `serialClassDesc` even if we don't use it)

```kotlin
@Serializable
class MyData(val s: String) {
    @Serializer(forClass = MyData::class)
    companion object /*: KSerializer<MyData> can be omitted or not, if you like*/{
        override fun save(output: KOutput, obj: MyData) {
            output.writeStringValue(HexConverter.printHexBinary(obj.s.toByteArray()))
        }

        override fun load(input: KInput): MyData {
            return MyData(String(HexConverter.parseHexBinary(input.readStringValue())))
        }
    }
}
```

*Note:* this approach is not working yet for generic classes.

## External serializers for library classes

Approach above will not work, if you can't modify source code of the class - e.g. it is a Kotlin/Java library class.
If it is Kotlin class, you can just let the plugin know you want to create serializer from object:

```kotlin
// imagine that MyData is a library class
// if you have generic class, you should use `class` instead of `object`
@Serializer(forClass = MyData::class)
object DataSerializer {}
```

This is called external serialization and imposes certain restrictions -
class should have only primary constructor's vals/vars and class body `var` properties (you can learn more in [docs](examples.md))

As in the first example, you can customize the process by overriding `save` and `load` methods.

If it is Java class, things getting more complicated: because Java has no concept of primary constructor,
plugin don't know which properties it can took. For Java classes, you always should override `save`/`load` methods.
You still can use `@Serializer(forClass = ...)` to generate empty serial descriptor.
For example, let's write serializer for `java.util.Date`:

```kotlin
@Serializer(forClass = Date::class)
object DateSerializer: KSerializer<Date> {
    private val df: DateFormat = SimpleDateFormat("dd/MM/yyyy HH:mm:ss.SSS")

    override fun save(output: KOutput, obj: Date) {
        output.writeStringValue(df.format(obj))
    }

    override fun load(input: KInput): Date {
        return df.parse(input.readStringValue())
    }
}
```

If your class has generic type arguments, it shouldn't be an object.
It must be a class with visible primary constructor, where its arguments are `KSerializer<T0>, KSerializer<T1>, etc..` - one for each type argument of your class.

### JS note

Due to an [issue](https://youtrack.jetbrains.com/issue/KT-11586), it's impossible now to write
annotations of a kind `@Serializer(forClass=XXX::class)` in JavaScript now. Unfortunately, you'll have to
implement all methods by yourself:

```kotlin
/**
* MyData serializer for JavaScript
*/
object MyDataSerializer: KSerializer<MyData> {
    override fun save(output: KOutput, obj: MyData) {
        output.writeStringValue(HexConverter.printHexBinary(obj.s.toUtf8Bytes()))
    }

    override fun load(input: KInput): MyData {
        return MyData(stringFromUtf8Bytes(HexConverter.parseHexBinary(input.readStringValue())))
    }

    override fun update(input: KInput, old: MyData): MyData {
        throw UpdateNotSupportedException(serialClassDesc.name)
    }
    
    override val serialClassDesc: KSerialClassDesc = SerialClassDescImpl("com.mypackage.MyData")
}
```

## Using custom serializers

Recommended way of using custom serializers is to give a clue to plugin about which serializer use for specified property,
using annotation in form `@Serializable(with = SomeKSerializer::class)`:

```kotlin
@Serializable
data class MyWrapper(
    val id: Int,
    @Serializable(with=MyExternalSerializer::class) val data: MyData
)
``` 

This will affect generating of `save`/`load` methods only for this dedicated class, and allows plugin to resolve serializer at compile-time to reduce runtime overhead.

## Registering and context

By default, all serializers are resolved by plugin statically when compiling serializable class.
This gives us type-safety, performance and eliminates reflection usage to minimum. However, if there is no
`@Serializable` annotation of class and no `@Serializable(with=...)` on property, in general, it is impossible to know at compile time which serializer to
use - user can define more than one external serializer, or define them in other module, or even it's a class from
library which doesn't know anything about serialization.

To support such cases, a concept of `SerialContext` was introduced. Roughly speaking, it's a map where
runtime part of framework is looking for serializers if they weren't resolved at compile time by plugin.
In some sense, it's similar to jackson modules.

If you want your external serializers to be used, you must register them in some (maybe new) context.
Then, you have to pass the context to the framework. Contexts are flexible and can be inherited; moreover,
you can pass different contexts to different instances of output formats. Let's see it in example:

```kotlin
// Imagine we have classes A and B with external serializers
fun foo(b: B): Pair<String, ByteArray> {
    val aContext = SerialContext().apply { registerSerializer(A::class, ASerializer) }
    
    val strContext = SerialContext(aContext) 
    strContext.registerSerializer(B::class, BStringSerializer)
    val json = JSON(context = strContext) // use string serializer for B in JSON
    
    val binContext = SerialContext(aContext)
    binContext.registerSerializer(B::class, BBinarySerializer)
    val cbor = CBOR(context = binContext) // user binarySerializer for B in CBOR
    
    return json.stringify(b) to cbor.dump(b)
} 
```
