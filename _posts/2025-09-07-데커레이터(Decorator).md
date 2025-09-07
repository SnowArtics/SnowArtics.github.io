---
title: "데커레이터(Decorator)"
author: seolwon
date: 2025-09-07 17:30:00 +0900
categories: [C++, Design Pattern]
tags: [C++, Design Pattern, Decorator]
---

# 서문
동료가 작성한 클래스를 기반으로 어떤 기능을 확장해야 한다고 생각해보자. 원본 코드를 수정하지 않으려면 어떻게 해야 하는가? 가장 쉽게 생각나는 방법은 상속을 이용하는 것이다. 동료의 클래스를 부모로 하는 자식 클래스를 만들어 거기에 새로운 기능들을 추가한다. 어쩌면 몇몇 멤버 함수들을 오버라이딩 해야 할 수도 있을 것이다. 이런 식으로 원본 코드에 수정을 가하지 않고 작업을 진행할 수 있다.

하지만 항상 이렇게 할 수 있는 것은 아니다. 상속은 그다지 선호되는 옵션이 아니다. 상속을 사용할 수 없는 여러가지 사정이 있을 수 있다. 예를 들어 std::vector의 경우 버추얼 소멸자가 없다는 제한이 있다. 무엇보다도 상속을 활용하기 어려운 가장 큰 이유는, 수정하는 이유가 여러가지인 경우 단일 책임 원칙(SRP)에 다라 그 수정 사항들을 각각 완전히 분리하는 것이 바람직하기 때문이다.

데커레이터  패턴은 이미 존재하는 타입에 새로운 기능을 추가하면서도 원래 타입의 코드에 수정을 피할 수 있게 해준다.(열림-닫힘 원칙(OCP)이 준수된다.) 여기에 더해서 파생해야 할 타입의 개수가 과도하게 늘어나는 것도 막을 수 있다.

# 시나리오

여러 가지 개선 작업이 필요한 상황을 생각해보자. 도형을 나타내는 클래스 Shape가 기존에 존재하고 있었고 이를 상속받아 색상이 있는 도형(ColoredShape)과 투명한 도형(TransparentShape)을 추가했다고 하자. 그리고 나중에 두 가지 특성을 모드 필요로 하는 경우가 발생하여 추가로 ColoredTransparentShape를 만들었다고 하자. 결과적으로 두 가지 기능을 추가하기 위해 클래스를 3개 만들었다. 이런 식이면 기능이 하나 더 추가될 경우 7개의 클래스를 만들어야 할 수도 있다. 설상가상으로 Square, Circle 등과 같은 도형의 파생 클래스들까지 적용해야 할 수도 있다. 세 가지의 추가 기능을 두 가지의 도형에 적용하려면 상속받아 만들어야 할 클래스가 14개까지 늘어난ㄷ. 이러한 상황은 감당할 수 없다. 자동 코드 생성툴을 이용하고 있더라도 쉬운 상황이 아니다.

제대로 상황을 이해하기 위해 실제 코드를 작성해보자. 추상 클래스 Shape이 다음과 같이 정의되어 있다고 하자.

```cpp
class Shape
{
public:
	virtual string str() const = 0; // 특정 도형의 상태를 문자열로 나타내기
};
```

이 클래스에서 str()은 버추얼 함수이고 특정 도형의 상태를 텍스트로 나타낸다.

이 클래스를 베이스로 하여 원, 사각형 같은 도형을 정의하고 str() 인터페이스를 구현할 수 있다.

```cpp
class Circle : public Shape
{
public:
	float radius;
	explicit Circle(const float radius_)
		:radius(radius_) {}

	void resize(float factor_) { radius *= factor_; }

	string str() const override
	{
		ostringstream oss;
		oss << "A circle of radius" << radius;
		return oss.str();
	}
};

class Square : public Shape
{
public:
	float side;
	explicit Square(const float side_)
		:side(side_) {}

	void resize(float factor_) { side *= factor_; }

	string str() const override
	{
		ostringstream oss;
		oss << "A side of Square" << side;
		return oss.str();
	}
};
```

평범한 상속으로는 효울적으로 새로운 기능을 도형에 추가할 수가 없다는 것을 알 수 있다. 따라서 접근 방법을 달리해 컴포지션을 활용한다. 콤포지션은 데커레이터 패턴에서 객체들에 새로운 기능을 확장할 때 활용되는 매커니즘이다. 이 접근 방식은 다시 두 가지 서로 다른 방식으로 나누어진다. 앞으로 이야기될 패턴들까지 고려하면 몇 가지 더 늘어난다.

- 동적 컴포지션 : 참조를 주고받으면서 런타임에 동적으로 무언가를 합성할 수 있게 한다. 이 방식은 최대한의 유연성을 제공한다. 예를 들어 사용자 입력에 따라 런타임에 반응하여 컴포지션을 만들 수 있다.
- 정적 컴포지션 : 템플릿을 이용하여 컴파일 시점에 추가기능이 합성되게 한다. 이것은 코드 작성 시점에 객체에 대한 정확한 추가 기능 조합이 결정되어야만 한다는 것을 암시한다. 즉, 나중에 수정될 수 없다.

이와 같은 설명이 이해하기 어렵더라도 걱정할 필요는 없다. 동적 컴포지션과 정적 컴포지션 두 경우 모드 데커레이터 패턴으로 구현해 볼 것이다. 코드를 보면 쉽게 이해된다.

# 동적 데커레이터

도형에 색을 입히려 한다고 가정해보자. 상속 대신 컴포지션으로 ColoredShape를 만들 수 있따. 이미 생성된 Shape 객체의 참조를 가지고 새로운 기능을 추가한다.

```cpp

class ColoredShape : public Shape
{
public:
	Shape& shape;
	string color;
	
	explicit ColoredShape(Shape& shape_, const string& color_)
		: shape{ shape_ }, color{ color_ } {}

	string str() const override
	{
		ostringstream oss;
		oss << shape.str() << " has the color " << color;
		return oss.str();
	}
};
```

코드에서 볼 수 있듯이 ColoredShape는 그 자체로도 Shape이다. ColoredShape은 다음과 같이 이용될 수 있다.

```cpp
Circle circle(0.5f);
ColoredShape redCircle(circle, "red");
cout << redCircle.str();
// 출력 결과 : "A circle of radius 0.5 has the color red"
```

만약 여기에 더하여 도형이 투명도를 가지게 하고 싶다면 마찬가지 방법으로 다음과 같이 쉽게 구현할 수 있다.

```cpp
class TransparentShape : public Shape
{
public:
	Shape& shape;
	uint8_t transparency;

	explicit TransparentShape(Shape& shape_, const uint8_t transparency_)
		: shape{ shape_ }, transparency{ transparency_ } {}

	string str() const override
	{
		ostringstream oss;
		oss << shape.str() << " has "
			<< static_cast<float>(transparency) / 255.f * 100.f
			<< "% Transparency";
		return oss.str();
	}
};
```

이제 0~255 범위의 투명도를 지정하면 그것을 퍼센티지로 출력해주는 새로운 기능이 추가되었다. 이러한 추가 기능은 그 자체만으로는 사용할 수 없고 적용할 도형 인스턴스가 있어야만 한다.

```cpp
Square square(3);
TransparentShape demiSquare(square, 85);
cout << demiSquare.str();
// 결과 출력 : A side of Square3 has 33.3333% Transparency
```

하지만 편리하게도 ColoredShape와 TransparentShape를 합성하여 색상과 투명도 두 기능 모두 도형에 적용되도록 할 수 있다.

```cpp
Circle circle(0.5f);
ColoredShape redCircle(circle, "red");
TransparentShape myCircle(redCircle, 64);
```

위 코드에서 볼 수 있듯이 모두 즉석에서 생성하고 있다. 대단히 멋지다. 하지만 문제도 있다. 비 상식적인 합성도 가능하다. 같은 데커레이터를 중복해서 적용해버릴 수도 있다. 예를 들어 ColoredShape(ColoredShape())와 같은 합성은 비상식적이지만 동작한다. 이렇게 중복된 합성은 “빨간색이면서 노란색이다.”와 같이 모순된 상황을 야기한다. OOP의 테크닉들을 활용하면 예로든 중복 합성이 방지되도록 할 수 있을 것이다. 하지만 아래와 같은 경우는 어떻게 될까?

```cpp
ColoredShape(TransparentShape(ColoredShape()))
```

이런 경우는 탐지해내기가 훨씬 어렵다. 만약 가능하다고 하더라도 투자 대비 효과를 따져보아야 할 것이다. 물론 보통의 경우 프로그래머가 상식적으로 사용할 것으로 가정해도 별 문제 없을 수도 있다.

하지만 지금은 학습임으로 정확한 해결 방법은 밑의 부록에 추가로 남긴다.

# 정적 데커레이터

예제 시나리오에서 Circle에는 resize() 멤버 함수가 있다. 이 함수는 Shape 인터페이스와 관계가 없다. 당연하게도 인위적으로 상황을 만들기 위해 그렇게 둔 것이다. 이미 짐작했듯이 resize()는 Shape 인터페이스에 없기 때문에 데커레이터에서 호출할 수가 없다. 즉, 아래의 코드는 컴파일이 안 된다.

```cpp
Circle circle(3);
ColoredShape redCircle(circle, "red");
redCircle.resize(2);
```

런타임에 객체를 합성할 수 있는지 없는지는 별로 상관하지 않지만 데커레이션된 객체의 멤버 함수와 필드에는 모두 접근할 수 있어야 한다면 어떻게 해야 할까? 그러한 데커레이터를 만드는 것이 가능할까?

다소 난해한 테크닉을 사용해야 하기는 하지만 가능하다. 템플릿과 상속을 활용하되 여기에서의 상속은 가능한 조합의 수가 폭발적으로 증가하지 않는다. 보통의 상속 대신 믹스인(MixIn) 상속이라 불리는 방식을 이용한다. 믹스인 상속은 템플릿 인자로 받은 클래스를 부모 클래스로 지정하는 방식을 말한다.

기본 아이디어는 다음과 같다. 새로운 클래스 ColoredShape를 만들고 템플릿 인자로 받은 클래스를 상속받게 한다. 이때 템플릿 파리미터를 제약할 방법은 없다. 어떤 타입이든 올 수 있다. 따라서 static_assert를 이용해 Shape이외의 타입이 지정되는 것을 막는다.

```cpp
template<typename T>
class ColoredShape : public T
{
public:
	static_assert(is_base_of<Shape, T>::value, "Template argument must be a Shape");

	string color;

	ColoredShape(string color_)
		: color(color_)
	{}

	ColoredShape() {}

	string str() const override
	{
		ostringstream oss;
		oss << T::str() << " has the color " << color;
		return oss.str();
	}
};

template<typename T>
class TransparentShape : public T
{
public:
	static_assert(is_base_of<Shape, T>::value, "Template argument must be a Shape");

	uint8_t transparency;

	TransparentShape(uint8_t transparency_)
		: transparency(transparency_)
	{}

	TransparentShape() {}

	string str() const override
	{
		ostringstream oss;
		oss << T::str() << " has "
			<< static_cast<float>(transparency) / 255.f * 100.f
			<< "% Transparency";
		return oss.str();
	}
};
```

ColoredShape<T>와 TransparentShape<T>의 구현을 기반으로 하여 색상이 있는 투명한 도형을 아래와 같이 합성할 수 있다.

```cpp
ColoredShape<TransparentShape<Square>> square("blue");
square.side = 2;
square.transparency = 0.5;
cout << square.str();
```

훌륭하다! 하지만 완벽하지는 않다. 모든 생성자를 한번에 편리하게 호출하던 부분을 잃었다. 가장 바깥의 클래스는 생성자로 초기화할 수 있지만 도형의 크기, 색상, 투명도까지 한번에 설정할 수는 없다.

데커레이션을 완성하기 위해 ColoredShape와 TransparentShape에 생성자를 전달한다. 이 생성자들은 두 종류의 인자를 받는다. 첫 번째 인자들은 현재의 템플릿 클래스에 적용되는 것들이고 두 번째 인자들은 부모 클래스에 전달될 제너릭 파라미터 팩이다. 즉, 아래와 같이 구현한다.

```cpp
template<typename T>
class ColoredShape : public T
{
public:
	static_assert(is_base_of<Shape, T>::value, "Template argument must be a Shape");

	string color;

	template<typename... Args>
	ColoredShape(const string color_, Args ...args)
		: T(std::forward<Args>(args)...)
		, color(color_)
	{}

	string str() const override
	{
		ostringstream oss;
		oss << T::str() << " has the color " << color;
		return oss.str();
	}
};

template<typename T>
class TransparentShape : public T
{
public:
	static_assert(is_base_of<Shape, T>::value, "Template argument must be a Shape");

	uint8_t transparency;

	template<typename... Args>
	TransparentShape(const uint8_t transparency_, Args ...args)
		: T(std::forward<Args>(args)...)
		, transparency(transparency_)
	{}

	string str() const override
	{
		ostringstream oss;
		oss << T::str() << " has "
			<< static_cast<float>(transparency) / 255.f * 100.f
			<< "% Transparency";
		return oss.str();
	}
};
```

다시 한번 언급하지만, 위의 생성자는 임의의 개수의 인자를 받을 수 있다. 앞쪽 인자는 투명도 값을 초기화하는데 이용되고 나머지 인자들은 그 인자가 어떻게 구성되었느냐와 관계없이 단순히 상위 클래스에 전달된다.

생성자들에 전달되는 인자의 타입과 개수, 순서가 맞지 않으면 컴파일 에러가 발생하기 때문에 올바르게 맞춰질 수 밖에 없다. 클래스 하나에 디폴트 생성자를 추가하면 파라미터의 설정에 훨씬 더 융통성이 생긴다. 하지만 인자 배분에 혼란과 모호성이 발생할 수 있다는 점을 염두에 두어야 한다.

이러한 생성자들에 explicit 지정자를 부여하지 않도록 주의해야 한다. 만약 그렇게 할 경우, 복수의 데커레이터를 합성할 때 C++의 복제 리스트 초기화 규칙 위반 에러를 만나게 된다. 이제 이러한 구현들이 어떻게 이용될 수 있는지 보자.

```cpp
ColoredShape<TransparentShape<Square>> square("blue", 51, 5);
cout << square.str();
// 출력 결과 : A side of Square 5 has 20% Transparency has the color blue
```

너무나 아름답다. 이것이 정확히 우리가 원하던 형태이다. 이것으로 정적 데커레이터의 구현을 모두 다루었다. 다시 말하지만, ColoredShape<ColoredShape<…>> 또는 ColoredShape<TransparentShape<ColoredShape<…>>>와 같이 동일한 데커레이터가 연속 중복 또는 순환 중복되는 것을 막도록 구현할 수 있다. 템플릿의 마법을 이용하여 충분히 실현할 수 있다. 하지만 여기에서 거기까지 다루기에는 주제에서 벗어난다.

# 함수형 데커레이터

데커레이터 패턴은 클래스를 적용 대상으로 하는 것이 보통이지만 함수에도 동등하게 적용될 수 있다. 예를 들어 코드에 문제를 일으키는 특정 동작이 있다고 하자. 그 동작이 수행될 때마다 모든 상황을 로그로 남겨 엑셀에 옮겨다가 상태를 분석하고 싶다고 하자. 로그를 남기기 위해 보통 해당 동작의 앞뒤로 아래와 같이 로그를 남긴다.

```cpp
cout << "Entering function\n";
// 작업 수행 ...
cout << "Exiting function\n";
```

이러한 방식은 익숙하고 잘 동작한다. 하지만 이해 관계를 분리한다는 디자인 철학(separation of concerns) 관점에서는 바람직하지 않다. 로깅 기능을 분리하여 다른 부분과 얽히지 않고 재사용과 개선을 마음 편하게 할 수 있다면 훨씬 더 효과적이다.

이렇게 하는 데에는 여러 접근 방법이 있을 수 있다. 한 가지 방법은 문제의 동작과 관련된 전체 코드 단위를 통째로 어떤 로깅 컴포넌트에서 람다로서 넘기는 것이다. 아래는 이러한 방법을 위한 로깅 컴포넌트의 정의이다.

```cpp
class Logger
{
	function<void()> func;
	string name;

	Logger(const function<void()>& func_, const string& name_)
		: func(func_)
		, name(name_)
	{}

	void operator()() const
	{
		cout << "Entering " << name << endl;
		func();
		cout << "Exiting " << name << endl;
	}
};
```

이 로깅 컴포넌트는 다음과 같이 활용할 수 있다.

```cpp
Logger ([]() {cout << "Hello" << endl; }, "HelloFunction")();
// 출력 결과 :
// Entering HelloFunction
// Hello
// Exiting HelloFunction
```

코드 블록을 std::function으로서 전달하지 않고 템플릿 인자로 전달할 수도 있다. 이 경우 로깅 클래스가 다음과 같이 조금 달라진다.

```cpp
class Logger2
{
public:
	Func func;
	string name;

	Logger2(const Func& func_, const string& name_)
		: func(func_)
		, name(name_)
	{}

	void operator()() const
	{
		cout << "Entering " << name << endl;
		func();
		cout << "Exiting " << name << endl;
	}
};
```

이 로깅 컴포넌트의 사용 방법은 완전히 같다. 로깅 인스턴스를 생성하기 위해 다음과 같이 편의 함수를 만든다.

```cpp
template <typename Func>
auto make_logger2(Func func_, const string& name_)
{
	return Logger2<Func>{func_, name_}; // () = call now
}
```

그리고 다음과 같이 활용한다.

```cpp
auto call = make_logger2([]() {cout << "Hello!" << endl; }, "HelloFuncton");
call();
```

그래서 정확히 뭐가 좋아지는가? 데커레이터를 생성할 수 있는 기반이 마련되었다는데 의미가 있다. 즉, 임의의 코드 블록을 데커레이션 할 수 있고 데커레이션된 코드 블록을 필요할 때 호출할 수도 있다.

이제 좀 더 어려운 경우를 보자. 로그를 남기고 싶은 함수의 리턴 값을 넘겨야 한다면 어떻게 해야 할까? 예를 들어 아래와 같이 함수 add()가 있다고 하자.

```cpp
double add(double a_, double b_)
{
	cout << a_ << "+" << b_ << "=" << (a_ + b_) << endl;
	return a_ + b_;
}
```

이 함수에 로그를 남기면서 동시에 리턴값도 넘겨야 한다면 어떻게 해야 할까? 그냥 Logger에서 값을 리턴하면 되는 거 아닌가 싶지만 실제로는 그렇게 되지 않는다. 하지만 불가능하지는 않다. Logger를 다음과 같이 바꾸어 보자.

```cpp
template<typename R, typename... Args>
class Logger3
{
public:
	function<R(Args...)> func;
	string name;

	Logger3(function<R(Args...)> func_, const string& name_)
		: func(func_)
		, name(name_)
	{}

	R operator() (Args... args_)
	{
		cout << "Entering " << name << endl;
		R result = func(args_...);
		cout << "Exiting " << name << endl;
		return result;
	}	
};
```

위 코드에서 템플릿 인자 R은 리턴 값의 타입을 의미한다. 그리고 Args는 이미 여러 번 보았던 파라미터 팩이다. 이 데커레이터는 앞서와 마찬가지로 함수를 가지고 있다가 필요할 때 호출할 수 있게 해준다. 유일하게 다른 점은 oprator()가 R 타입의 리턴 값을 가진다는 점이다. 즉, 리턴 값을 잃지 않고 받아낼 수 있다.

데커레이터 생성 편의 함수를 이에 맞게 수정하면 아래와 같이 된다.

```cpp
template <typename R, typename... Args>
auto make_logger3(R(*func_)(Args...), const string& name_)
{
	return Logger3<R, Args...>(
		std::function<R(Args...)>(func_),
		name_
	);
}
```

이 편의 함수는 한 가지 달라진 점이 있다. std::function 대신에 일반 함수 포인터를 첫 번째 인자로 받고 있다. 이 편의 함수를 이용하면 다음과 같이 add()를 데커레이션 하고 호출할 수 있다.

```cpp
auto logged_add = make_logger3(add, "Add");
auto result = logged_add(2, 3);
// 출력 결과
// Entering Add
// 2+3=5
// Exiting Add
```

당연하게도 make_logger3를 종속성 주입으로 대체할 수도 있다. 종속성 주입 방식을 이용하면 다음과 같은 장점이 생긴다.

- 실제 로깅 컴포넌트 대신 Null 객체를 전달하여 로깅 작업을 동적으로 끄거나 켤 수 있다.
- 로깅할 코드의 실제 실행을 막을 수도 있다. 이 부분도 로깅 컴포넌트를 다르게 제공함으로써 구현할 수 있다.

이러한 기능은 개발자에게 유용한 도구가 되어 준다.

# 요약

데커레이터 패턴은 열림-닫힘 원칙(OCP)를 준수하면서도 클래스에 새로운 기능을 추가할 수 있게 해준다. 데커레이터의 핵심적은 특징은 데커레이터들을 합성할 수 있다는 것이다. 객체 하나에 복수의 데커레이터들을 순서와 무관하게 적용할 수 있다. 이 절에서는 다음과 같은 데커레이터 패턴의 종류를 살펴보았다.

- 동적 데커레이터 : 데커레이션할 객체의 참조를 저장하고(객체의 전체 값을 저장할 수도 있다.) 런타임에 동적으로 합성할 수 있다. 대신 원본 객체가 가진 맴버들에 접근할 수 없다.
- 정적 데커레이터 : 믹스인 상속(템플릿 파라미터를 통한 상속)을 이용해 컴파일 시점에 데커레이터를 합성한다. 이 방법은 런타임 융통성은 가질 수 없다. 즉, 런타임에 객체를 다시 합성할 수가 없다. 하지만 원본 객체의 멤버들에 접근할 수 있는 장점이 있다. 그리고 생성자 포워딩을 통해 객체를 완전하게 초기화할 수 있다.
- 함수형 데커레이터 : 코드 블록이나 특정 함수에 다른 동작을 덧씌워 합성할 수 있다.

다중 상속이 지원되지 않는 프로그래밍 언어에서는 데커레이터 패턴을 이용해 여러 객체를 연관시켜 다형성을 흉내낸다. 즉, 연간시킨 여러 객체의 인터페이스를 데커레이터로 합성하여 단일한 인터페이스 집합을 제공한다.