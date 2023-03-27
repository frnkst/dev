# Marshalling and unmarshalling

- It's called **marshalling** when you convert a xml object in memory to a serialized format like a string or file
- It's called **unmarshalling** when you convert a xml file or string to an object in memory (for exmpale to a java class)

## With the @XmlRoot annotation

If the class you are trying to marshall has a @XmlRoot annotation it's pretty straight forward to marshall it.

```kotlin
val context: JAXBContext = JAXBContext.newInstance(Person::class.java)
val marshaller = context.createMarshaller()

// output pretty printed
marshaller.setProperty(Marshaller.JAXB_FORMATTED_OUTPUT, true)

val sw = StringWriter()
marshaller.marshal(person, sw)
println(sw.toString())
```

## Without a @XmlRoot annotation

I ran into a case, where I wanted to test a function that has @XmlType annotated java class as an input parameter. The verify function of the Mockk library always returned false, because the nested java object doesn't implement an equals function. Since this java class should be easily serialized I tried to marshall it using JAXB. I got an error saying that the class doesn't have an @XMLRoot annotation, which indeed was true. Since the class is from a third party library I had to find a way to marshall it with only a @XmlType annotation. Turns out this can be done by wrapping the object in a JAXBElement with a QName.

```kotlin
val jaxbElement = JAXBElement<Person>(
    QName("", "person"),
    Person::class.java,
    person
)
```

A full example would look like this. Let's say we have the following java class without a @XMLRoot annotation:

```java
@XmlAccessorType(XMLAccessType.FIELD)
@XmlType(name = "person")
public class Person {
  String firstName;
  String lastName;

  Person(String firstName, String lastName) {
    this.firstName = firstName;
    this.lastName = lastName;
  }
}
```

Now we can marshall it using the following code:

```kotlin
val person = Person("firstName", "lastName")
val context: JAXBContext = JAXBContext.newInstance(Person::class.java)
val marshaller = context.createMarshaller()
val sw = StringWriter()
val jaxbElement = JAXBElement<Person>(QName("", "person"), Person::class.java, person)

marshaller.marshal(jaxbElement, sw)
println(sw.toString())
```
