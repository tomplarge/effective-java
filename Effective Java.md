## 1. Creating and Destroying Objects
### Item 1: Consider providing static factory methods instead of constructors
```java
public static Boolean valueOf(boolean b) {
  return (b ? Boolean.TRUE : Boolean.FALSE);
 }
```
Advantages: 
- clear names
- not required to create a new object each time they're invoked
	- allows immutable classes to use preconstructed instances 
- can return an object of any subtype of their return type
	- return non-public classes
	
Disadvantages:
- classes without public or protected constructors cannot be subclassed
- not readily distinguishable from other static methods
	- valueOf: returns instance that has same value as parameters
	- getInstance: returns an instance described by parameters but cannot be said to have same value

### Item 2: Consider a builder when faced with many constructor parameters
TODO

### Item 3: Enforce the singleton property with a private constructor
Method 1:
```java
public class Elvis {
	public static final Elvis INSTANCE = new Elvis();
	private Elvis() {
		...
	}
}
```
Main advantage: clear that class is singleton 
Method 2:
```java
public class Elvis {
	private static final Elvis INSTANCE = new Elvis();
	private Elvis() {
		...
	}
	public static Elvis getInstance() {
		return INSTANCE;
	}
}
```
Main advantage: flexibility to change mind about whether class should be singleton or not

Serializing: need special readResolve method:
```java
private Object readResolve() throw ObjectStreamException {
	return INSTANCE;
}
```
### Item 4: Enforce noninstantiability with a private constructor
For a class that just groups static methods (utility classes), use private constructor (that does nothing). Compiler gives default public constructor when none are given, even for abstract classes. However, this class cannot be subclassed.
### Item 5: Prefer dependency injection to hardwiring resources
TODO

### Item 6: Avoid creating duplicate objects
- DON'T DO THIS:  ` String s = new String("silly");`
- Using static factory methods can eliminate duplicate object creation e.g. `Boolean.valueOf(String)` vs `Boolean(String)`
- For classes that will never change, only instantiate once:
```java
class Person {
	private static final Date BOOM_START;
	private static final DATE BOOM_END;
	static {
		// Instantiate BOOM_START and BOOM_END
	}
	public boolean isBabyBoomer() {
		// compare to BOOM_START and BOOM_END
	}
}
```
### Item 7: Eliminate obsolete object references
- If an object reference is unintentionally retained, this object and objects references held by this object, etc.
- Stack implementation that manages its own memory:
```java
public Object pop() {
	if (size == 0)
		throw new EmptyStackException();
	Object result = elements[--size];
	elements[size] = null; // Eliminate obsolete reference
	return result;
}
```
- Usually, when managing own memory, need to null out object references
- Caching is another common location for memory leaks
	- solution 1: if cache entry is relevant only as long as there are references to it, use `WeakHashMap`
	- solution 2: clean cache with timer activated background thread or side effect of insert

### Item 8: Avoid finalizers
- DO NOT use finalizers for anything time-sensitive or anything critical state related
- `try-finally` is usually used to reclaim resources
- Provide explicit termination method instead, usually used with `try-finally`

### Item 9: Prefer try-with-resources to `try-finally`
TODO

## 2. Methods Common to All Objects
### Item 10: Obey the general contract when overriding `equals`
- By default, objects are only equal to themselves
- Do not override equals if any of the following:
	- Each instance of the class is inherently unique
	- You don't care whether the class provides a logical equality test
	- A superclass has already overriden `equals`, and the behavior inherited from the superclass is appropriate for this class
	- The class is private or package-private, and you are certain that its `equals` method will never be invoked
- Generally, override `equals` with value classes (e.g. `Integer` or `Date`)
- `equals` contract: reflexive, symmetric, transitive, consistent (`x.equals(y)` is the same after multiple invocations), and `x.equals(null)` returns `false`
- Violating symmetry example: overriding `equals` for `caseInsensitiveString` to interoperate with `String` (`String` doesn't know about `caseInsensitiveString`)
- Violating transitivity example: `Point` class, subclassed by `ColorPoint` with `color` field, where `ColorPoint` does color-blind comparison when given a `Point`, but then this violates transitivity:
```java
ColorPoint p1 = new ColorPoint(1,2,Color.RED);
Point p2 = new Point(1,2);
ColorPoint p3 = new ColorPoint(1,2,Color.BLUE);
```
- There is simply no way to extend an instantiable class and add an aspect while preserving `equals` contract
- Solution: don't have `ColorPoint` extend `Point` and add point view method
```java
public class ColorPoint {
	private Point point;
	private Color color;
	public ColorPoint(int x, int y, Color color) {
		point = new Point(x,y);
		this.color = color;
	}
	// Returns point-view of this color point
	public Point asPoint() {
		return point;
	}
	public boolean equals(Object o) {
		if (!(o instanceof ColorPoint))
			return false;
		ColorPoint cp = (ColorPoint) o;
		return cp.point.equals(point) && cp.color.equals(color);
	}
}
```
Guidelines:
1. Use `==` operator to check if argument is a reference to this object
2. Use `instanceof` to check if argument is of correct type
3. Cast the argument to correct type
4. For each "significant" field in class, check if field matches corresponding field. For floats, `Float.floatToIntBits` and compare with `==`. For doubles, use `Double.doubleToLongBits` and compare with `==`. Necessary for `-0.0f` and `Float.NaN`. For references, compare with `(field == null ? o.field == null : field.equals(o.field))`
5. Check for symmetry, transitivity, and consistency
6. Always override `hashCode` as well

### Item 11: Always override `hashCode` when you override `equals`
- Equal objects must have equal hash codes

Guidelines:
1. Store some constant nonzero value in an `int result`
2. For each field used in `equals`, compute a hashcode `c`:
	1. `boolean` &rarr; `(f ? 0 : 1)`
	2. `byte`, `char`, `short`, `int` &rarr; `(int) f`
	3. `long` &rarr; `(int) (f^(f >>> 32))`
	4. `float` &rarr; `Float.floatToIntBits(f)`
	5. `double` &rarr; `Double.doubleToLongBits(f)` and then 3. 
	6. `Object` &rarr; compute `hashCode`
	7. `array` &rarr; treat each element as a separate field
3. Combine `c` and `result` with `result = 37*result + c`
4. Return `result`
5. Check that equal instances have equal hash codes

### Item 12: Always override `toString`
- Providing a good `toString` implementation will make your class much more pleasant to use (`toString` is called for `println` or `+` string concatenation)
- When practical, should return all of the interesting information
- It can be useful to specify the format and provide a `String` constructor (or static factory). However, once you've specified the format, there's no going back
- Allow programmatic access to all the information contained in value returned by `toString`

### Item 13: Override `clone` judiciously
- `Cloneable` makes an object's `clone` method return a field-by-field copy of the object, otherwise it throws `CloneNotSupportedException`
- If you override `clone` method in a non-final class, you should return an object obtained by invoking `super.clone`. This will chain up to the `Object.clone` method, and return an object of the right type (i.e. don't use a constructor)
- A class that implements `Cloneable` is expected to provide a properly functioning `clone`, which requires all superclasses provide one as well
- If all fields are primitive or refer to immutable Objects, then the following suffices:
```java
public Object clone() {
	try {
		return super.clone();
	} catch(CloneNotSupportedException) {
		// Can't happen
		throw new Error("Assertion failure");
	}
}
```

- If a field refers to mutable object, then call `clone` on this field (usually):
```java
public Object clone() throws CloneNotSupportedException {
	Stack result = (Stack) super.clone();
	result.elements = (Object[]) elements.clone();
	return result;
}
```
- the clone architecture is incompatible with normal use of final fields referring to mutable objects (since you can't assign a new value to that field from the result of `super.clone()`)
- Sometimes just calling `clone()` on the field isn't sufficient (hash table with buckets array of linked lists). Just cloning the buckets will create a new list that references the old linked lists.
	- Solution 1: Iteratively (or recursively) "deep-copy" each of the linked lists
	- Solution 2: Call `super.clone`, set all fields in resulting object to virgin state, and then call higher-level methods to regenerate the state of the object (e.g. empty buckets list and use `put(key, value)` for each entry to be cloned) 
- Don't invoke any non-final methods on the clone under construction since the state of the original and clone could be intermixed
- A far better alternative is to provide a copy constructor: `public Foo (Foo foo) { ... }`

### Item 14: Consider implementing `Comparable`
- `compareTo()` method is the sole method of `Comparable`
- Contract:
1. `sign(x.compareTo(y)) == -sign(y.compareTo(x))`
2. transitivity
3. `x.compareTo(y) == 0` implies that `sign(x.compareTo(z)) == sign(y.compareTo(z))` for all `z`
4. `(x.compareTo(y) == 0) == x.equals(y)`
- Same issue applies as with `equals`: adding an aspect to a subclass will cause an issue, so use the same solution as before
- Example: 
```java
public int compareTo(Object o) {
	PhoneNumber pn = (PhoneNumber) o;
	// compare area codes
	int areaCodeDiff = areaCode - pn.areaCode;
	if (areaCodeDiff != 0) {
		return areaCodeDiff;
	}
	// compare exchanges
	int exchangeCodeDiff = exchangeCode - pn.exchangeCode;
	if (exchangeCodeDiff != 0) {
		return exchangeCodeDiff;
	}
	// compare extensions
	return extension - pn.extension;
}
```

## 3. Classes and Interfaces
### Item 15: Minimize the accessibility of classes and members
- Make each class or member as inaccessible as possible
- Access level modifiers:
	- `private`: the member is accessible only inside the top-level class where it is declared
	- `package-private` (default): the member is accessible from any class in the package where it is declared
	- `protected`: the member is accessible from subclasses where it is declared and from any class in the package where it is declared
	- `public`: member is accessible from anywhere
- private should be the reflex - too much package-private suggests changes should be made
- big jump from package-private to protected - becomes part of exported API. protected should be rare
- public classes should rarely have public fields, except for static final constants containing primitive values or references to immutable objects
- almost always wrong to have a public static final array - arrays are  always mutable. Alternative:
```java
private static final Type[] PRIVATE_VALUES = { ... };
public static final List VALUES = Collections.unmodifiableList(Arrays.asList(PRIVATE_VALUES));
```

### Item 16: In public classes, use accessor methods, not public fields
TODO

### Item 17: Favor Immutability
- Follow these rules to make class immutable:
	1. Don't provide any methods that modify the object
	2. Ensure that no methods may be overridden (make the class final, or alternatives below)
	3. Make all fields final
	4. Make all fields private
	5. Ensure exclusive access to any mutable components. If your class has any fields that refer to mutable objects, ensure clients cannot obtain these references. Make defensive copies in constructors, accessors, and `readObject` methods
- Example where the arithmetic creates new object
```java
public final class Complex {
	private final float re;
	private final float im;
	public Complex(float re, float im) {
		this.re = re;
		this.im = im;
	}
	public float realPart() { return re; }
	public float imaginaryPart() { return im; }
	public Complex add(Complex c) {
		return new Complex(re + c.re, im + c.im);
	}
	// Other operations omitted
}
```
- Advantages:
	- simplicity; only one state
	- inherently thread-safe; require no synchronization
	- can be shared freely. take advantage of this by providing public static final constants: `public static final Complex ZERO = new Complex(0,0);`
		- can even provide static factory that cache frequently requested instances and avoid creating new instances when a preexisting instance is requested
	- should not provide `clone` method
	- sharable internals (`BigInteger` uses sign-magnitude and shares the magnitude array with the `negate` method)
- Only disadvantage is cost of creating a new object for each distinct value
	- Solution 1: guess which multistep operations will be common and provide them as primitives (single functions)
	- Solution 2: provide public mutable companion class (`String` class has mutable companion `StringBuffer`)
- Making class immutable: instead of making class itself final you can make all constructors private or package-private and add public static factories. This allows the use of multiple package-private implementation classes. Externally, it is effectively final since it can't be extended and lacks public/protected constructor
```java
public class Complex {
	private final float re;
	private final float im;
	Complex (float re; float im) {
		this.re = re;
		this.im = im;
	}
	public static Complex valueOf(float re, float im) {
		return new Complex(re, im);
	}
}
```
- This is also effective because it allows for other types of constructors (e.g. `Complex` from polar coordinates, which would be messy since it would also take the same arguments)
- As a relaxation, one can add non-final redundant fields for caching expensive computations, which is returned if the request is made for that value in the future
```java
private volatile Foo cachedFooVal = UNLIKELY_FOO_VALUE;
public Foo foo() {
	Foo result = cachedFooVal;
	if (result == UNLIKELY_FOO_VALUE)
		result = cachedFooVal = fooVal();
	return result;
}
private Foo fooVal() { ... }
```
- Be careful with `Serializable` - provide explicit `readObject` and `readResolve` methods
- If a class can't be made mutable, limit mutability as much as possible (limit amount of states the object can be in). Constructors should create fully initialized objects with all of their invariants established

### Item 18: Favor composition over inheritance
 - Inheritance is fine when subclass and superclass are under the control of the same programmers and when classes are specifically designed for extension. Inheritance across package boundaries is dangerous (superclass can change from release to release)
 - Composition solves this problem: instead of extending, add the superclass as a private field of the new class. Methods will have the same name but just forward to the "superclass"
```java
// Wrapper class - uses composition instead of inheritance
public class InstrumentedSet implements Set {
	private final Set s;
	pricvate int addCount = 0;
	public InstrumentedSet(Set s) {
		this.s = s;
	}
	public boolean add(Object o) {
		addCount++;
		return s.add(o);
	}
	public boolean addAll(Collection c) {
		addCount += c.size();
		return s.addAll(c);
	}
	public int getAddCount() { return addCount; }
	// Forwarding methods
	public void clear() { s.clear(); }
	public boolean contains(Object o) { return s.contains(o); }
	...
}
``` 
- Use inheritance only when a "is-a" relationship exists: "Is every B really an A?" Ultimately, inheritance violates encapsulation

### Item 19: Design and document for inheritance or else prohibit it
- Class must document precisely the effects of overriding any method
	- document self-use of overridable (non-final and public/protected) methods i.e. when it's used, in what sequence, how results affect subsequent processing
- Class may have to provide hooks into its internal workings in the form of judiciously chosen protected methods
	- Example: `protected removeRange(fromIndex, toIndex)` in `AbstractList` to speed up `clear`. Unnecessary for end-users, but still provided for overriding
- Provide as few protected methods/fields as possible because each one represents a commitment to an implementation detail, but enough for usable inheritance
- Constructors must not invoke overridable methods, directly or indirectly. This is because superclass constructor runs before subclass constructor, and if overriding method depends on initialization, then failure may occur. Example:
```java
public class Super {
	public Super() {
		m();
	}
	public void m() {
	}
}

final class Sub extends Super {
	private final Date date;
	Sub() {
		date = new Date();
	}
	public void m() {
		System.out.prinln(date);
	}
	public static void main(String[] args) {
		Sub s = new Sub();
		s.m();
	}
}
``` 
- This program prints `null` and then a date, thus a final field is observed in 2 states. 
- Also not advised to implement `Cloneable` or `Serializable` as they present large burdens on the inherited class
- It is best to prevent subclassing in classes not designed or documented to be safely subclassed
	- Method 1: declare class `final`
	- Method 2: make all constructors private/package-private and replace with static factories
- One way to leave a class open to subclassing is to ensure no self use (no invoking overridable methods) by moving the body of each overridable method to a private "helper method" and have each overridable method invoke its private helper method.

### Item 20: Prefer interfaces to abstract classes
- Existing classes can be easily retrofitted to implement a new interface, which is not generally true for an abstract class
- Interfaces are ideal for defining mixins (a type that a class can implement in addition to its "primary type"), which is not true for abstract types
- Interfaces allow the construction of nonhierarchical type frameworks.
```java
public interface Singer {
	AutoClip sing(Song s);
}
public interface Songwriter {
	Song compose(boolean hit);
}
public interface SingerSongwriter extends Singer, Songwriter {
	AudioClip() strum();
	void actSensitive(); // lol wtf
}
```
- Interfaces enable safe, powerful functionality enhancements via the wrapper class idiom
- You can combine interfaces and abstract classes by providing an abstract skeletal implementation class to go with each nontrivial interface you export (e.g. `AbstractMap`, `Abstract Set`, etc.)
- Example of using a abstract skeletal implementation, demonstrating how simple it is to provide all the utility of a `List` with such a design:
```java
// List adapter for int array
static List intArrayAsList(final int[] a) {
	if (a == null) throw new NullPointerException();
	
	return new AbstractList() {
		public Object get(int i) {
			return new Integer(a[i]);
		}
		public int size() {
			return a.length
		}
		public Object set(int i, Object o) {
			int oldVal = a[i];
			a[i] = ((Integer) o).intValue();
			return new Integer(oldVal);
		}
	};
}
```
- Designing a skeletal implementation:
	- study interface and decide which methods are the primitives in terms of which the others can be implemented. These will be abstract methods
	- Then provide concrete implementations of all other methods in the interface
- Example:
```java
public abstract class AbstractMapEntry implements Map.Entry {
	// primitives
	public abstract Object getKey();
	public abstract Object getValue();
	
	// Entries in modifiable maps must override this method
	public Object setValue(Object value) {
		throw new UnsupportedOperationException();
	}
	
	public boolean equals(Object o) {
		if (o == this) 
			return true;
		if (! (o instanceof Map.Entry))
			return false;
		Map.Entry arg = (Map.Entry) o;
		return eq(getKey(), arg.getKey()) && 
			   eq(getValue(), arg.getValue());
	}
	private static boolean eq(Object o1, Object o2) {
		return (o1 == null ? o2 == null : o1.equals(o2));
	}
	public int hashCode() {
		return 
			(getKey() == null ? 0 : getKey().hashCode()) ^ 
			(getValue() == null ? 0 : getValue().hashCode());
	}
}
```
- Note that the documentation guidelines in Item 19 should be followed for skeletal implementations
- Abstract classes have one advantage: it is far easier to evolve an abstract class than an interface (since adding additional method for interfaces in new release breaks things)

### Item 21: Define interfaces for posterity
TODO

### Item 22: Use interfaces only to define types
- Interfaces tell clients what an instance of a class can do that implement the interface
- The constant interface pattern is a poor use of interfaces:
```java
public interface PhysicalConstants {
	static final double AVOGADROS_NUMBER = ...
	static final double BOLTZMANN_CONSTANT = ...
	...
}
```
- This says nothing about what a client can do with a class and may even confuse them. 
- To export constants:
	- if the constants are strongly tied to an existing class/interface, add them
	- perhaps a `typesafe enum`
	- non-instantiable utility class (Item 4):
```java
public class PhysicalConstants() {
	private PhysicalConstants() {}
	public static final double AVOGADROS_NUMBER = ...
	...
```

### Item 23: Prefer class hierarchies over tagged classes
TODO

### Item 24: Favor static member classes over non-static
- Nested classes (classes defined within another class) should exist only to serve to enclosing class
- 4 kinds of nested classes: static member classes, non-static member classes, anonymous classes, and local classes
- static member class
	- ordinary class declared inside another class, having access to all enclosing class members (even private), obeys same rules as other static members
	- use whenever defining a member class that does not require access to an enclosing instance
	- Example: `Map` has internal private static `Entry` object for each key-value pair in the map. Methods on an `Entry` (`getKey`, `getValue`, etc.) don't require access to the map.
- non-static member class: 
	- same as static, just not declared static
	- each instance is implicitly associated with an enclosing instance of its containing class. 
	- can invoke methods of enclosing class and get reference to it - impossible to create an instance of a non-static member class without an enclosing instance
	- one common use is to define an Adapter to allow outer class to be viewed as instance of unrelated class (e.g. iterators)
```java
public class MySet extends AbstractSet {
	public Iterator iterator() {
		return new MyIteerator();
	}
	private class MyIterator implements Iterator {
		...
	}
}
```
- anonymous class
	- rather than being declared with other instances, it is simultaneously declared and instantiated. 
	- one common use is a function object: 
```java
Arrays.sort(args, new Comparator() {
	public int compare(Object o1, Object o2) {
		return ((String) o1).length() - ((String) o2).length();
	}
});
```
```java
// Typical use of a public static member class
public class Calculator {
	public static abstract class Operation {
		private final String name;
		Operation(String name) { this.name = name; }
		public String toString() { return this.name; }
		abstract double eval (double x, double y);
		
		// Doubly nested anonymous classes
		public static final Operation PLUS = new Operation("+") {
			double eval (double x, double y) { return x + y; }
		};
		public static final Operation MINUS = new Operation("-") {
			double eval (double x, double y) { return x - y; }
		};
		...
	}
	public double calculate(double x, Operation op, double y) {
		return op.eval(x, y);
	}
}
```
### Item 25: Limit source files to a single top-level class
TODO


## 5. Generics
### Item 26: Don't use raw types
- Raw type corresponding to `List<E>` is `List`
```java
// Raw collection type - don't do this!
private final Collection stamps = ...;

// Raw iterator type - don't do this!
for (Iterator i = stamps.iterator(); i.hasNext(); ) 
	Stamp stamp = (Stamp) i.next(); // Throws ClassCastException if i.next() is a coin
```
Do this instead:
```java
// Parameterized collection type - typesafe
private final Collection<Stamp> stamps = ...;
```
- Cannot pass `List<String>` as a parameter of type `List<Object>`
```java

```
## 6. Enums and Annotations

### Item 34: Use enums instead of `int` constants
- C `enum` is not bad
- `typesafe enum` pattern: Define class representing single element of enumerated type, and don't provide any public constructors, but public static final fields instead, one for each constant in `enum`.
```java
public class Suit {
	private final String name;
	private Suit(String name) { this.name = name; }
	public String toString() { return name; }
	public static final Suit CLUBS = new Suit("clubs");
	public static final Suit DIAMONDS = new Suit("diamonds");
	public static final Suit HEARTS = new Suit("hearts");
	public static final Suit SPADES = new Suit("spades");
}
```
- Class is not extendable and can only exist in the 4 different provided types
- To extend this to allow for sorting:
```java
// Ordinal-based typesafe enum
public class Suit implements Comparable {
	private final String name;
	// ordinal of next suit to be created
	private static int nextOrdinal = 0;
	// assign an ordinal to this suit
	private final int ordinal = nextOrdinal++;
	public int compareTo(Object o) {
		return ordinal - ((Suit) o).ordinal;
	}
	// rest the same
	...
}
```
- Item 24 presents a method for providing enum constants that provide their own implementation of methods. A declaration of `abstract double eval(double x, double y);` should also be included.
- Also suggested to override `equals` and `hashCode` with `Object` invocations:
```java
public final boolean equals(Object that) {
	return super.equals(that);
}
public final int hashCode() {
	return super.hashCode();
}
```
- To make this pattern extensible, just provide a protected constructor. Note that the comparable variant does not work with the extensible variant.
```java
// Serializable, extensible typesafe enum
public abstract class Operation implements Serializable {
	private final transient String name;
	protected Operation(String name) { this.name = name; }
	public static final Operation PLUS = new Operation("+") {
		protected double eval(double x, double y) { return x + y; }
	}
	...
	protected abstract double eval(double x, double y);
	public String toString() { return this.name; }
	public final boolean equals(Object that) {
		return super.equals(that);
	}
	public final int hashCode() {
		return super.hashCode();
	}
	// for serialization
	private static nextOrdinal = 0;
	private final int ordinal = nextOrdinal++;
	private static final Operation[] VALUES = 
		{PLUS, MINUS, TIMES, DIVIDE}
	Object readResolve() throws ObjectStreamException {
		return VALUES[ordinal];
	}
}
```
- Extension of this enum to add log:
```java
abstract class ExtendedOperation extends Operation {
	ExtendedOperation(String name) { super(name); }
	public static Operation LOG = new ExtendedOperation("log") {
		protected double eval(double x, double y) {
			return Math.log(y) / Math.log(x);
		}
	};
	// for serialization
	private static nextOrdinal = 0;
	private final int ordinal = nextOrdinal++;
	private static final Operation[] VALUES = 
		{LOG}
	Object readResolve() throws ObjectStreamException {
		return VALUES[ordinal];
	}
```

## 8. Methods
### Item 49: Check parameters for validity
- Check parameter validity at the beginning, and document these restrictions
- For public methods, use Javadoc `@throws` tag to document exception that is thrown with invalid parameter (e.g. `IllegalArgumentException`, `IndexOutOfBoundsException`, `NullPointerException`)
```java
/*
 * [Description]
 * @param m the modulus, which must be positive
 * @return this mod m
 * @throws ArithmeticException if m <= 0
 */
public BigInteger mod(BigInteger m) {
	if (m.signum() <= 0) {
		throw new ArithmeticException("Modulus not positive");
	}
	...
} 
```
- For non-public methods, check parameters with assertions
- If validity check is expensive, or will be implicitly done in calculations, you can exclude or delay it
- 
<!--stackedit_data:
eyJoaXN0b3J5IjpbMTAxNDMyMzQ2OCwxMjg0NjgwNTIyLDEyNT
Y1OTQ0MTUsLTExMjg2NjYzLDM1MzU5Mjg1NiwxNDU3ODI5MDkz
LC0xMDg4NDYzODQ4LC0xMjg5MTU1NDM1LDg1NTM4NjYxOCwtMj
AxMzgwNTY5MiwxNTA4NDc5NzI5LC0xMTUzNjMyMTY5LDE2MDk0
NTgzNzAsMjAyNTY5NDUwNyw0MzcyNDY2NzEsOTQ3NTIwMjcyXX
0=
-->