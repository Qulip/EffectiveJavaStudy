# ITEM 13. **clone 재정의는 주의해서 진행하라.**

`Cloneable`은 복제해도 되는 클래스임을 명시하는 용도의 `mixin interface`지만 의도한 목적을 제대로 이루지 못했다.  
가장 큰 문제는 `clone`메서드가 선언된 곳이 `Cloneable`이 아닌 `Object`이고, `protected`로 선언되어있다.  
이러한 이유로 인해 `Cloneable`을 구현하는 것만으로는 외부 객체에서 `clone` 메서드를 호출할 수 없다.

<br>

## **`Cloneable` 인터페이스의 동작 방식**

메서드 하나 없는 `Cloneable` 인터페이스는 놀랍게도 `Object`의 `protected` 메서드인 `clone`의 동작 방식을 결정한다.  
`Cloneable` 을 구현한 클래스의 인스턴스에서 `clone`메서드를 호출하면, 그 객채의 모든 필드를 각각 복사해 반환하고, 만약 `Cloneable`을 구현하지 않은 클래스의 인스턴스에서 호출하면 `CloneNotSupportedException`을 던진다.  

단 이는 인터페이스를 상당히 이례적으로 사용한 예이니 따라하지 말자.  
인터페이스를 구현하는 것은 일반적으로 해당 클래스가 그 인터페이스에서 정의한 기능을 제공한다고 선언하는 행위이지만, `Cloneable`의 경우는 상위 클래스에서 정의된 `protected` 메서드의 동작 방식을 변경한 것이다.

<br>

## **`clone` 메서드의 일반 규약의 허술함**

```java
//어떤 객체 X에 대해 아래 식은 참이다.
x.clone() != x

//다음식도 참이다.
x.clone().getClass() == x.getClass()

//다음식도 일반적으로 참이지만, 필수는 아니다.
x.clone().equals(x)

//관례상, 이 메서드가 반환하는 객체는 super.clone을 호출해 얻어야 한다.
//이 클래스와 Object를 제외한 모든 상위 클래스가 이 관례를 따르면 다음식은 참이 된다.
x.clone().getClass() == x.getClass()

//관례상, 반환된 객체와 원본 객체는 독립적이어야 한다. 
//이를 만족하려면 super.clone으로 얻은 객체의 필드 중 하나 이상을 반환 전에 수정해야 할 수도 있다.
```
이는 강제성이 없다는 점만 뺴면 생성자 연쇄와 비슷한 메커니즘이다.  
만약 클래스에서 생성자를 호출해 얻은 인스턴스를 반환하여도 컴파일러는 문제를 찾지 못한다.  
하지만, 하위 클래스에서 `super.clone()`메서드를 호출하는 순간 문제가 발생한다.

위 상황의 해결방법은 아래와 같다.

```java
@Override
public PhoneNumber clone() {
    try {
        return (PhoneNumber) super.clone();
    } catch (ClassNotSupportedException e){
        throw new AssertionError();             //일어날 수 없는 일이다.
    }
}
```
`Object`의 `clone` 메서드는 `Object`를 반환하지만 `PhoneNumber`의 `clone` 메서드는 `PhoneNumber`를 반환하게 하였다.  
위와 같이 클라이언트가 형 변환을 하지 않아도 되게 작성하자.

위 코드에서 `try-catch` 블록으로 감싼 이유는 `Object` 의 `clone` 메서드가 `ClassNotSupportedException`을 던지도록 선언 되어있기 때문이다.  
하지만 우리는 `super.clone`이 무조건 성공할 것임을 안다. 이는 `ClassNotSupportedException`이 사실은 비검사 예외여야 한다는 신호이다.(Item. 71)

<br>

## **가변 객체를 참조하는 경우**

위에서 구현된 `clone` 메서드가 가변 객체를 참조하는 순간 문제가 발생한다.

~~~java
public class Stack {
    private Object[] elements;
    private int size = 0;
    private static final int DEFAULT_INITIAL_CAPACITY = 16;

    public Stack() {
        this.elements = new Object[DEFAULT_INITIAL_CAPACITY];
    }

    public void push(Object e) {
        ensureCapacity();
        elements[size++] = e;
    }

    public Object pop() {
        if (size == 0)
            throw new EmptyStackException();
        Object result = elements[--size];
        elements[size] = null; // 다 쓴 참조 해제
        return result;
    }

    //원소를 위한 공간을 적어도 하나 이상 확보한다.
    private void ensureCapacity() {
        if (elements.length == size) {
            elements = Arrays.copyOf(elements, 2 * size + 1);
        }
    }
}
~~~

위와 같이 복제를 할 경우, 반환된 `Stack` 인스턴스의 `size` 필드는 올바른 값을 가지지만, `element`필드는 원본 `Stack` 인스턴스와 똑같은 배열을 참조하게 된다.  
이의 경우 원본이나 복제본 둘중 하나를 수정하여도 둘다 수정되어 불변식을 해치게 된다.  
**`clone` 메서드는 사실상 생성자와 같은 효과를 낸다. 즉 원본 객체에 아무런 해를 끼치지 않으며 동시에 불변식을 보장해야 한다.**  
`Stack`의 `clone`메서드를 제대로 동작하게 하려면 스택 내부 정보를 복사해야하는데, `elements` 배열의 `clone`메서드를 재귀적으로 호출해 주면 된다.

```java
@Override
public Stack clone() {
    try {
        Stack clone = (Stack) super.clone();
        clone.elements = elements.clone(); // 배열을 복제할 때는 배열의 clone 메서드를 사용하길 권장한다.
        return clone;
    } catch(CloneNotSupportedException e) {
        throw new AssertionError();
    }
}
```

재귀적으로 `clone` 메서드를 호출하는 것만으로는 충분하지 않는 경우도 있다.  
이럴떄는 `deepCopy`메서드를 재귀적으로 사용하거나 반복문으로 순회하여 복제한다.  
단 재귀적으로 복제할 경우 스택 오버플로가 발생할 수 있으므로 반복문을 사용하자.  
아래 코드를 보자.

```java
public class HashTable implements Cloneable {
        private Entry[] buckets = ...;
        private static class Entry {
            final Object key;
            Object value;
            Entry next;
            Entry(Object key, Object value, Entry next){
                this.key = key;
                this.value = value;
                this.next = next;
            }
            //자신이 가르키는 연결 리스트를 반복적으로 복사한다.
            Entry deepCopy() {
                Entry result = new Entry(key, value, next);
                for(Entry p = result; p.next != null; p = p.next){
                    p.next = new Entry(p.next.key, p.next.value, p.next.next);
                }
                return result;
            }
            //생략 ..
        }

    @Override 
    public HashTable clone() {
        try {
            HashTable result = (HashTable) super.clone();
            result.buckets = new Entry[buckets.length];
            for (int i = 0; i < buckets.length; i++)
            if (buckets[i] != null)
                result.buckets[i] = buckets[i].deepCopy();
            return result;
        } catch (CloneNotSupportedException e) {
            throw new AssertionError();
        }
    }
}
```

<br>

## **복잡한 가변 객체를 복사하는 방법**

- `super.clone`을 호출하여 얻은 객체의 모든 필드를 초기 상태로 설정 후, 원본 객체의 상태를 다시 생성하는 고수준 메서드 들을 호출한다.
- 고수준 API를 활용하는 경우는 저수준에서 바로 처리하는 경우보다 느리다.

<br>

## **주의사항**

### **고려 사항**
- 생성자에서는 재정의될 수 있는 메서드를 호출하지 않아야 한다. `clone` 메서드도 마찬가지다.  
  만약 `clone`이 하위 클래스에서 재정의한 메서드 호출 시, 하위 클래스는 복제 과정에서 자신의 상태 교정 기회를 잃게 된다.  
  이는 원본과 복제본의 상태가 달라지게 할 가능성이 존재한다.  

### **예외처리**
- `Object`의 `clone` 메서드는 `CloneNotSupportedException`를 던진다고 선언하였지만, 재정의한 메서드는 아니다.  
  `public`인 `clone` 메서드는 `throws`절을 없애야 한다. 그래야 그 메서드를 사용하기 편하기 때문이다.  

### **상속해서 사용하기 위한 클래스**
- 상속해서 쓰기 위한 클래스 설계 방식 두 가지(Item. 19)중 어느 쪽에서든, 상속용 클래스는 `Cloneable`을 구현해서는 안된다.  
  이럴 경우 `Object`의 방식을 모방할 수도 있다.
- 하위 클래스에서 `Cloneble` 구현 여부를 선택하게 한다.
- 다른 방법으로는 `clone` 메서드를 동작하지 않게 구현하고, 하위 클래스에서 재정의를 막는 방법이 있다.  

```java
@Override
protected final Object clone() throws CloneNotSupportedException {
    throw new CloneNotSupportedException();
}
```

### **스레드 안전 클래스**
- `Cloneable`을 구현한 스레드 안전 클래스를 작성할 때는 `clone` 메서드 역시 적절히 동기화가 필요하다.(Item 78)
- `Object`의 `clone` 메서드는 동기화를 신경 쓰지 않았으므로 동기화가 필요하다.

<br>

## **다른 방법**

`Cloneable`을 이미 구현한 클래스라면 `clone`이 잘 작동하도록 구현해야한다. 하지만 그렇지 않은 상황에서는 **복사 생성자와 복사 팩터리**라는 더 좋은 객체 복사 방식을 제공할 수 있다.

- 복사 생성자 : 자신과 같은 클래스의 인스턴스를 인수로 받는 생성자
``` java
public Yam(Yam yam){ ... };
```

- 복사 팩터리 : 복사 생성자를 모방한 정적 팩터리(Item. 1)
``` java
public static Yam newIstance(Yam yam){ ... };
```

이 두 방식은 `Cloneable, clone` 방식보다 더 낫다.
- 언어 모순적이고, 위험천만한 객체 생성 메커니즘을 사용하지 않고, 엉성하게 문서화된 규약에 기대지 않고, 정상적 `final`필드 용법과 충돌하지 않으며, 불필요한 검사 예외를 던지지 않고, 형변환도 필요없다.(**사실 거의 모든 문제를 해결해주는 수준...**)
- 복사 생성자, 복사 팩터리는 해당 클래스가 구현한 '인터페이스' 타입의 인스턴스를 인수로 받을 수 있다.
- 이 둘은 원본의 구현 타입에 얽메이지 않고 복제본의 타입을 직접 선택이 가능하다.
  ex) `HashSet` 객체 s를 `TreeSet`타입으로 쉽게 복제가 가능하다.  
    `new TreeSet<>(s)`로 쉽게 처리가 가능하다.

<br>

## **결론**

`Cloneable`이 가져온 모든 문제를 보면, 새로운 인터페이스를 만들때 절대 확장하면 안되며, 새로운 클래스도 이를 구현하면 안된다.  
`final` 클래스라도 성능 최적화 관점에서 검토 후 문제가 없다면 드물게 허용해야 한다.(Item. 67)  
기본 원칙은 "복제 기능은 생성자와 팩터리를 이용하는 것이 최고."라는 것이다.  
단 배열은 `clone` 메서드 방식이 가장 깔끔한 예외이다.