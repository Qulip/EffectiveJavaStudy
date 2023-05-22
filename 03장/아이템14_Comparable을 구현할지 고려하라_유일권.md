# ITEM 14. **Comparable을 구현할지 고려하라.**

## **Comparable 인터페이스의 compareTo**

- compareTo()는 Object의 equals()와 다르게 단순 동치성 비교 뿐만 아니라 순서 비교도 가능하며 제네릭하다.
- Comparable의 존재은 인스턴스들이 자연적인 순서를 가진다는 뜻이며, 손쉽게 정렬이 가능하고, 컬렉션 검색/극단값 계산/자동 정렬도 쉽게 가능하다.
- 비교를 활용하는 정렬된 컬렉션인 TreeSet과 TreeMap이 있고, 검색과 정렬 알고리즘을 활용하는 유틸리티 클래스인 Collections와 Arrays가 있음

<br>

## **규약**

순서를 비교할 때 이 객체가 주어진 객체보다 작으면 음의정수(-1), 같으면 (0), 크면 양의정수(+1) 반환

비교할 수 없는 타입의 객체가 주어지면 `ClassCastException` 을 던짐

1) x.compareTo(y) == -y.compareTo(x)

    - 두 객체 참조의 순서를 바꿔 비교해도 예상한 결과가 나와야함  
<br>
2) 추이성 (x.compareTo(y) > 0 && y.compareTo(z) > 0) == x.compareTo(z) > 0
    - 두 번째 규약은 첫번째가 두 번째보다 크고 두 번째가 세 번째보다 크다면, 첫번째는 세번째보다 커야한다.  
<br>
3) x.compareTo(y) == 0 == (x.compareTo(z) == y.compareTo(z))
    - 크기가 같은 객체들끼리는 어떤 객체와 비교하더라도 항상 같아야 한다는 뜻
<br>
1) (x.compareTo(y) == 0) == x.eqauls(y)
    - 이 권고는 필수가 아니지만, 꼭 지키는것이 좋다. (지키지 못하면 명시)
        지키지 않는다면 컬렉션에 넣으면 해당 컬렉션이 구현한 인터페이스에 정의한 동작과 엇박자를 낼 것이다.  
        (ex. Collections, Set, 혹은 Map)
<br>

## **compareTo 작성 요령**

- `equals` 메서드 작성 요령과 비슷하지만, 몇가지 차이점이 있다.
- `Comparable`은 타입을 인수로 받는 제네릭 인터페이스이므로 `compareTo()`의 인수 타입은 컴파일타임에 정해지게 된다.
- `compareTo()`는 각 필드가 동치인지를 비교하는것이 아니라 순서를 비교하는 것이다.
- `Comparable`을 구현하지 않은 필드나 표준이 아닌 순서로 비교해야한다면 `Comparator`(비교자)를 대신 사용 (비교자는 직접 작성 or 자바에서 제공되는 것 사용)
- `compareTo()`에서는 관계연산자 <,>를 사용하는 이전 방식은 거추장스럽고 오류를 유발하므로 박싱 기본 타입 클래스들의 정적 메서드 compare()을 사용하자.
- 핵심 필드부터 비교해나가서 순서가 먼저 결정되면 곧장 반환하자.

<br>

## **비교자 생성 메서드를 이용한 비교자**

자바8에서 Comparator 인터페이스가 비교자 생성 메서드를 이용하여 비교자를 생성할 수 있게 되었다.

객체 비교가 복잡해지는 경우에 Comparator 비교자를 활용할 수 있고, 체이닝하여 순차적으로 필드를 비교할 수 있다.

```java
private static final Comparator<PhoneNumber> COMPARATOR =
            comparingInt((PhoneNumber pn) -> pn.areaCode)
                    .thenComparingInt(pn -> pn.prefix)
                    .thenComparingInt(pn -> pn.lineNum);

    @Override
    public int compareTo(PhoneNumber pn) {
        return COMPARATOR.compare(this, pn);
    }
```

<br>

## 주의사항

```java
//추이성을 위배함
static Comparator<Object> hashCodeOrder = new Comparator<>() {
    public int compare(Object o1, Object o2) {
        return o1.hashCode() - o2.hashCode();
    }
};
```

값의 차를 기준으로 비교하는 코드는 정수 오버플로우나 부동 소수점 계산 방식에 따른 오류를 낼 수도 있다.  
대신 아래의 방식 중 하나를 사용하자.

```java
static Comparator<Object> hashCodeOrder = new Comparator<>() {
    public int compare(Object o1, Object o2) {
        return Integer.compare(o1.hashCode(), o2.hashCode());
    }
};
```

```java
static Comparator<Object> hashCodeOrder = Comparator.comparingInt(o -> o.hashCode());
```

<br>

## **결론**

순서를 고려해야하는 값 클래스를 작성하게 된다면 꼭 `Comparable` 인터페이스를 구현하여  
인스턴스들을 쉽게 정렬/검색하고 비교 기능을 제공하는 컬렉션과 어우러져야 한다.

`compareTo()`에서는 필드 값 비교시에 <,>연산자 말고 박싱된 기본 타입 클래스가 제공하는 정적 메서드인  
`compare()`이나 `Comparator` 인터페이스가 제공하는 비교자 생성 메서드를 사용하도록 한다.