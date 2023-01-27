# 제네릭스(Generics)

2022.07.12

한 가지 예시를 들면서 이번 주제를 시작해보겠습니다. 여름이 되면서 요즘 날씨가 되게 덥습니다. 이럴 때 과일 화채 먹으면서 에어컨 앞에 앉아 유튜브 보면 진짜 그것 만큼 기분 좋은 일이 없을텐데 말이죠. 현실은 도서관에서 자바의 정석을 읽고 있네요..ㅎ

제가 화채를 해먹기 위해 마트에 가서 여러 과일들을 산다고 생각해봅시다. 여름에는 복숭아랑 수박이 맛있으니까 이 과일들을 산다고 생각해보죠. 그러면 마트에 이 과일들이 각기 다른 박스에 담겨 진열이 되어 있을 것입니다. 수박 박스랑 복숭아 박스를 한번 만들어보죠.

```java
class waterMelonBox {
	WaterMelon item;
	
	void setItem(WaterMelon item) {
		this.item = item;
	}
	WaterMelon getItem() {return item;}
}
```

```java
class peachBox{
	Peach item;
	
	void setItem(Peach item) {
		this.item = item;
	}
	Peach getItem() {return item;}
}
```

두 클래스의 코드가 굉장히 비슷합니다. 그러면 모든 객체의 조상이 `Object`니까 저장하는 `item`의 타입을 `Object`로 해서 하나의 박스로 만들 수 있지 않을까요?

```java
class Box {
	Object item;

	void setItem(Object item) { this.item = item; }
	Object getItem() { return item; }
}
```

그럴싸 해보입니다. 근데 `Object`는 모든 객체의 조상이기 때문에 어떤 타입이든 `Box`에 저장이 가능합니다. 즉, 수박 박스라고 생각한 박스에 복숭아가 아무 문제 없이 저장될 수 있는거죠.

```java
Peach peach = new Peach();
Box waterMelonBox = new Box();
waterMelonBox.setItem(peach);    // peach도 Object 타입이기 때문에 문제 없이 저장이 된다!
```

이러면 안됩니다. 수박을 담는 박스에는 수박만 담아야하죠. **하나의 박스를 이용하여 여러 과일을 담는 박스를 만들고 싶지만 일단 이 박스가 수박을 담는 박스라고 우리가 정의를 하면 수박만 담게 하고 싶습니다.** 이렇게 할 수 있는 방법이 없을까요? 바로 **지네릭스(Generics)**가 이를 가능하게 해줍니다.

## 지네릭스(Generics)

지네릭스의 정의는 다음과 같습니다.

<aside>
💡 ***다양한 타입의 객체들을 다루는 메서드나 컬렉션 클래스에 컴파일 시의 타입 체크를 해주는 기능***

</aside>

말이 조금 어려운데 쉽게 풀면 다음과 같이 말할 수 있습니다.

<aside>
💡 *여러 타입에서 사용하고 싶은 컬렉션 클래스에 **현재 이 클래스가 어떤 타입을 다루고 있는지**를 체크하는 기능*

</aside>

위의 과일 박스의 예시의 경우 지네릭스를 사용하여 현재 이 박스가 수박을 담고 있는지 복숭아를 담고 있는지를 나타낼 수 있습니다. 이렇게 하면 컴파일러가 ‘이 박스는 수박을 담는구나’ 또는 ‘이 박스는 복숭아를 담는구나’를 알아 **수박 박스에 복숭아가 저장되는 불상사를 막을 수 있습니다.** 이것이 제네릭의 장점, **타입 안정성을 제공한다**입니다. 또, 타입이 표시되기 때문에 **불필요한 타입 체크 및 형변환을 줄일 수 있습니다.**

### 제네릭 클래스의 선언, 사용, 제한

제네릭을 사용하여 `Box`를 다시 구현하면 다음과 같습니다.

```java
class Box<T> {
	T item;
	
	void setItem(T item) { this.item = item; }
	T getItem() { return item; }
}
```

여기서 **`T`는 타입 변수이며 임의의 참조형 타입**을 의미합니다. 여기에 우리가 원하는 타입을 지정해주면 해당 타입의 클래스로 사용이 가능한 것입니다. 다음과 같이 `WaterMelon`과 `Peach`로 타입을 구체화하면, 이제 해당 박스에서는 각각 `WaterMelon`과 `Peach` 타입의 객체만 저장이 가능합니다.

```java
Box<WaterMelon> waterMelonBox = new Box<WaterMelon>();
Box<Peach> peachBox = new Box<Peach>();
```

제네릭에 사용하는 타입이 상속이랑 더해지면 조금 헷갈리기 시작합니다. `WaterMelon`과 `Peach`의 조상인 `Fruit`이 있다고 생각해보죠. **제네릭이 사용된 클래스의 객체를 생성할 때는 T에 넣는 타입이 반드시 같아야합니다.** 따라서 아무리 상속 관계에 있더라고 해도 다음의 코드는 에러가 납니다. 

```java
Box<Fruit> box = new Box<Fruit>();   // OK
Box<Fruit> box = new Box<Peach>();   // 에러
```

하지만 `Fruit`과 `Peach`는 상속 관계이기 때문에 `Box`의 매개변수에 `Peach` 객체를 사용할 수 있습니다.

```java
Box<Fruit> box = new Box<Fruit>();
box.setItem(new Peach());
```

또, 제네릭을 이용하여 클래스의 타입을 하나로 제한할 수는 있지만 그 타입은 여전히 모든 종류의 타입이 가능합니다. 우리는 `Box`를 이용하여 과일 박스만 만들고 싶은데 `Integer` 박스도 만들 수 있고, `Boolean` 박스도 만들 수 있다는 것입니다. 이를 방지하기 위해 제네릭 클래스 선언 시 **`extends`를 이용하여 타입 매개변수 T에 지정할 수 있는 타입의 종류를 제한**할 수 있습니다.

```java
class Box<T extends Fruit> {
	ArrayList<T> list = new ArrayList<T>;
	... 
}
```

상속을 받았을 때처럼 동일하게 `extends`를 작성해주면 T에 들어갈 수 있는 것은 `Fruit`의 자손들 뿐입니다. 따라서 우리는 T에 들어갈 수 있는 타입에 제한을 둘 수 있습니다.

### 와일드 카드

**`static` 메서드에는 타입 변수를 사용할 수 없습니다.** `static` 메서드에서 인스턴스 변수를 사용할 수 없는 것과 같은 이유인데요. `static` 메서드 호출 시에 생성되어 있는 인스턴스가 존재하지 않을 수 있기 때문에 `static` 메서드의 타입을 지정해줄 수 없기 때문입니다. 

그러면 static 메서드를 만들 때 같은 내용이더라도 각기 다른 타입에 대해 정의를 해주어야 합니다. 마트에서 현재 과일 박스에 있는 과일의 정보들을 알고 싶다면 각 과일에 대하여 메서드를 정의해주어야 합니다. 

```java
class Mart {
	static void print(Box<WaterMelon> box) {
		System.out.println(box.item);
	}
	static void print(Box<Peach> box) {
		System.out.println(box.item);
	}
}
```

근데 위와 같이 코드를 작성하면 컴파일 에러가 발생합니다. 바로 두 메서드는 오버로딩 된 것이 아니라 중복되어 정의가 된 것이기 때문이죠. 컴파일러는 컴파일 시에 제네릭 타입을 떼버리기 때문에 결국 두 메서드는 같은 메서드가 되는 것입니다. 이때 **`static` 메서드에서도 제네릭 타입을 이용할 수 있게 해주는 것이 와일드 카드**입니다.

와일드 카드를 사용하면 사용 가능한 타입에 상한과 하한을 정할 수 있습니다. 

- <? extends T> : T의 자손들만 가능. 상한 제한
- <? super T> : T의 조상들만 가능. 하한 제한
- <?> : 모든 타입 가능

와일드 카드를 써서 위의 코드를 올바르게 작성하면 다음과 같습니다.

```java
class Mart {
	static void print(Box<? extends Fruit> box) {
		System.out.println(box.item);
	}
}
```

`extend`와 `super` 모두 상속에서 사용했던 의미와 동일하기 때문에 쉽게 이해할 수 있습니다.

`<? super T>`의 경우 `sort()`를 정의할 때 많이 사용합니다.

```java
static <T> void sort(List<T> list, Comparator<? super T> c);
```

보통 `sort()`를 정의할 때 위와 같은 형태로 많이 정의를 하는데 조상 클래스에 하나의 `Comparator`를 정의해놓으면 자손 클래스가 정렬할 때 해당 `Comparator`를 기준으로 정렬할 수 있기 때문에 코드의 재사용성은 물론이고 **표준화된 정렬 기준**을 제공할 수 있습니다.