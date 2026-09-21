---
title: الـ Generics
weight: 200
---

# الـ Generics

**[يمكنك العثور على كل كود هذا الفصل هنا](https://github.com/quii/learn-go-with-tests/tree/main/generics)**

سيقدّم لك هذا الفصل مقدمة عن الـ generics، ويُزيل أي تحفظات قد تكون لديك تجاهها، ويعطيك فكرة عن كيفية تبسيط بعض كودك في المستقبل. وبعد قراءته ستعرف كيف تكتب:

- دالة تأخذ وسائط من نوع generic
- نوع بيانات generic


## دوال المساعدة في اختباراتنا (`AssertEqual`, `AssertNotEqual`)

لاستكشاف الـ generics سنكتب بعض دوال المساعدة (test helpers).

### التحقق من الأعداد الصحيحة

لنبدأ بشيء بسيط ثم نتقدم خطوة بخطوة نحو هدفنا

```go
import "testing"

func TestAssertFunctions(t *testing.T) {
	t.Run("asserting on integers", func(t *testing.T) {
		AssertEqual(t, 1, 1)
		AssertNotEqual(t, 1, 2)
	})
}

func AssertEqual(t *testing.T, got, want int) {
	t.Helper()
	if got != want {
		t.Errorf("got %d, want %d", got, want)
	}
}

func AssertNotEqual(t *testing.T, got, want int) {
	t.Helper()
	if got == want {
		t.Errorf("didn't want %d", got)
	}
}
```


### التحقق من النصوص

القدرة على التحقق من تساوي الأعداد الصحيحة أمر رائع، لكن ماذا لو أردنا التحقق على `string` ؟

```go
t.Run("asserting on strings", func(t *testing.T) {
	AssertEqual(t, "hello", "hello")
	AssertNotEqual(t, "hello", "Grace")
})
```

ستحصل على خطأ

```
# github.com/quii/learn-go-with-tests/generics [github.com/quii/learn-go-with-tests/generics.test]
./generics_test.go:12:18: cannot use "hello" (untyped string constant) as int value in argument to AssertEqual
./generics_test.go:13:21: cannot use "hello" (untyped string constant) as int value in argument to AssertNotEqual
./generics_test.go:13:30: cannot use "Grace" (untyped string constant) as int value in argument to AssertNotEqual
```

إذا تمهّلت في قراءة الخطأ، فسترى أن المترجم يشتكي من أننا نحاول تمرير `string` إلى دالة تتوقع `integer`.

#### مراجعة سريعة لأمان الأنواع (type-safety)

إذا كنت قد قرأت الفصول السابقة من هذا الكتاب، أو لديك خبرة مع اللغات ذات الأنواع الثابتة، فلا ينبغي أن يفاجئك هذا. فمترجم Go يتوقع منك أن تكتب دوالك وstructs وغيرها بوصف الأنواع التي تريد العمل بها.

لا يمكنك تمرير `string` إلى دالة تتوقع `integer`.

وقد يبدو هذا إجراءً شكليًا، لكنه مفيد للغاية. فبوصفك هذه القيود أنت:

- تجعل تنفيذ الدالة أبسط. فبوصفك للمترجم الأنواع التي تعمل بها، **تقيّد عدد التنفيذات الصحيحة الممكنة**. فلا يمكنك "جمع" `Person` مع `BankAccount`، ولا يمكنك تحويل `integer` إلى حروف كبيرة. وفي البرمجيات، تكون القيود مفيدة للغاية في كثير من الأحيان.
- تمنع نفسك من تمرير بيانات إلى دالة عن غير قصد.

وتوفر لك Go طريقة لتكون أكثر تجريدًا بأنواعك عبر [واجهات (interfaces)](structs-methods-and-interfaces.md)، لتتمكن من تصميم دوال لا تأخذ أنواعًا ملموسة، بل أنواعًا تقدّم السلوك الذي تحتاجه. ويمنحك ذلك مرونة مع الحفاظ على أمان الأنواع.

### دالة تأخذ string أو integer؟ (بل وحتى أشياء أخرى)

خيار آخر تملكه Go لجعل دوالك أكثر مرونة هو إعلان نوع الوسيط لديك بـ `interface{}`، وهو يعني "أي شيء".

جرّب تغيير التواقيع لتستخدم هذا النوع بدلًا من ذلك.

```go
func AssertEqual(got, want interface{})

func AssertNotEqual(got, want interface{})

```

ينبغي أن تترجم الاختبارات الآن وتنجح. وإذا حاولت إفشالها فسترى أن المخرجات مضطربة قليلًا لأننا نستخدم صيغة العدد الصحيح `%d` في طباعة رسائلنا، فغيّرها إلى الصيغة العامة `%+v` للحصول على مخرجات أفضل لأي نوع من القيم.

### المشكلة مع `interface{}`

دوال `AssertX` لدينا ساذجة بعض الشيء، لكنها من حيث المفهوم ليست بعيدة كثيرًا عن طريقة [المكتبات الشائعة في تقديم هذه الوظيفة](https://github.com/matryer/is/blob/master/is.go#L160)

```go
func (is *I) Equal(a, b interface{})
```

فما المشكلة؟

باستخدام `interface{}` لا يستطيع المترجم مساعدتنا عند كتابة كودنا، لأننا لا نخبره بأي شيء مفيد عن أنواع الأشياء التي تمرّر إلى الدالة. جرّب مقارنة نوعين مختلفين.

```go
AssertEqual(1, "1")
```

في هذه الحالة ننجو من الأمر؛ فالاختبار يُترجم ويفشل كما نأمل، مع أن رسالة الخطأ `got 1, want 1` غير واضحة؛ لكن هل نريد فعلًا أن نتمكن من مقارنة النصوص بأعداد صحيحة؟ وماذا عن مقارنة `Person` بـ `Airport`؟

كتابة دوال تأخذ `interface{}` قد تكون صعبة للغاية وعرضة للأخطاء، لأننا _فقدنا_ قيودنا، وليس لدينا أي معلومة وقت الترجمة عن أنواع البيانات التي نتعامل معها.

وهذا يعني **أن المترجم لا يستطيع مساعدتنا**، وبدلًا من ذلك ترتفع احتمالية حدوث **أخطاء وقت التشغيل (runtime errors)** قد تؤثر على مستخدمينا، أو تسبب انقطاعات، أو ما هو أسوأ.

وغالبًا ما يضطر المطورون إلى استخدام الـ reflection لتنفيذ هذه الدوال الـ generic *همم*، وهو ما قد يصير معقّدًا في القراءة والكتابة، وقد يضرّ بأداء برنامجك.

## دوال المساعدة في اختباراتنا باستخدام الـ generics

من الأفضل ألا نضطر إلى إنشاء دوال `AssertX` مخصصة لكل نوع نتعامل معه. نريد أن تكون لدينا دالة `AssertEqual` _واحدة_ تعمل مع _أي_ نوع، لكنها لا تسمح لك بمقارنة [التفاح بالبرتقال](https://en.wikipedia.org/wiki/Apples_and_oranges).

توفّر لنا الـ generics طريقة لعمل تجريدات (مثل الـ interfaces) بأن تتيح لنا **وصف قيودنا**. فهي تسمح لنا بكتابة دوال تمتلك مستوى مرونة مشابهًا لما يقدمه `interface{}`، لكن مع الحفاظ على أمان الأنواع وتقديم تجربة أفضل للمطورين الذين يستدعونها.

```go
func TestAssertFunctions(t *testing.T) {
	t.Run("asserting on integers", func(t *testing.T) {
		AssertEqual(t, 1, 1)
		AssertNotEqual(t, 1, 2)
	})

	t.Run("asserting on strings", func(t *testing.T) {
		AssertEqual(t, "hello", "hello")
		AssertNotEqual(t, "hello", "Grace")
	})

	// AssertEqual(t, 1, "1") // uncomment to see the error
}

func AssertEqual[T comparable](t *testing.T, got, want T) {
	t.Helper()
	if got != want {
		t.Errorf("got %v, want %v", got, want)
	}
}

func AssertNotEqual[T comparable](t *testing.T, got, want T) {
	t.Helper()
	if got == want {
		t.Errorf("didn't want %v", got)
	}
}
```

لكتابة دوال generic في Go، تحتاج إلى تقديم "معاملات النوع" (type parameters)، وهي مجرد طريقة أنيقة للقول "صِف نوعك الـ generic وأعطه اسمًا".

في حالتنا، نوع معامل النوع لدينا هو `comparable` وقد أعطيناه اسم `T`. وهذا الاسم يتيح لنا بعد ذلك وصف أنواع الوسائط في دالتنا (`got, want T`).

نستخدم `comparable` لأننا نريد أن نوضح للمترجم أننا نرغب في استخدام المعاملين `==` و`!=` على الأشياء من نوع `T` داخل دالتنا، فنحن نريد المقارنة! وإذا جرّبت تغيير النوع إلى `any`،

```go
func AssertNotEqual[T any](got, want T)
```

ستحصل على الخطأ التالي:

```
prog.go2:15:5: cannot compare got != want (operator != not defined for T)
```

وهذا منطقي تمامًا، لأنك لا تستطيع استخدام هذين المعاملين مع كل نوع (أو `any`).

### هل الدالة الـ generic مع [`T any`](https://go.googlesource.com/proposal/+/refs/heads/master/design/go2draft-type-parameters.md#the-constraint) هي نفسها `interface{}` ؟

تأمّل دالتين

```go
func GenericFoo[T any](x, y T)
```

```go
func InterfaceyFoo(x, y interface{})
```

ما الفائدة من الـ generics هنا؟ ألا يصف `any` ... أي شيء؟

من ناحية القيود، فإن `any` تعني فعلًا "أي شيء"، وكذلك `interface{}`. بل إن `any` أُضيفت في الإصدار 1.18 وهي _مجرد اسم مستعار لـ `interface{}`_.

الفرق في النسخة الـ generic هو _أنك ما زلت تصف نوعًا محددًا_، ومعنى ذلك أننا ما زلنا نقيّد هذه الدالة لتعمل مع نوع _واحد_ فقط.

ويعني هذا أنك تستطيع استدعاء `InterfaceyFoo` بأي توليفة من الأنواع (مثل `InterfaceyFoo(apple, orange)`). أما `GenericFoo` فما زالت تقدّم بعض القيود لأننا قلنا إنها تعمل مع نوع _واحد_ فقط هو `T`.

صالح:

- `GenericFoo(apple1, apple2)`
- `GenericFoo(orange1, orange2)`
- `GenericFoo(1, 2)`
- `GenericFoo("one", "two")`

غير صالح (يفشل في الترجمة):

- `GenericFoo(apple1, orange1)`
- `GenericFoo("1", 1)`

إذا كانت دالتك تُرجع النوع الـ generic، فيستطيع المستدعي أيضًا استخدام النوع كما هو، بدلًا من الحاجة إلى تأكيد النوع (type assertion)، لأن المترجم لا يستطيع تقديم أي ضمانات بشأن النوع عندما تُرجع الدالة `interface{}`.

## التالي: أنواع البيانات الـ generic

سننشئ نوع بيانات [مكدس (stack)](https://en.wikipedia.org/wiki/Stack_(abstract_data_type)). وينبغي أن تكون المكدسات سهلة الفهم إلى حد كبير من ناحية المتطلبات. فهي مجموعة من العناصر تستطيع `Push` عناصر إلى "أعلاها"، وللحصول على العناصر مرة أخرى تستخدم `Pop` لإخراجها من الأعلى (LIFO - last in, first out).

وللاختصار، حذفت عملية TDD التي أوصلتني إلى الكود التالي لمكدس من `int`، ومكدس من `string`.

```go
type StackOfInts struct {
	values []int
}

func (s *StackOfInts) Push(value int) {
	s.values = append(s.values, value)
}

func (s *StackOfInts) IsEmpty() bool {
	return len(s.values) == 0
}

func (s *StackOfInts) Pop() (int, bool) {
	if s.IsEmpty() {
		return 0, false
	}

	index := len(s.values) - 1
	el := s.values[index]
	s.values = s.values[:index]
	return el, true
}

type StackOfStrings struct {
	values []string
}

func (s *StackOfStrings) Push(value string) {
	s.values = append(s.values, value)
}

func (s *StackOfStrings) IsEmpty() bool {
	return len(s.values) == 0
}

func (s *StackOfStrings) Pop() (string, bool) {
	if s.IsEmpty() {
		return "", false
	}

	index := len(s.values) - 1
	el := s.values[index]
	s.values = s.values[:index]
	return el, true
}
```

وقد أنشأت دالتين أخريين للمساعدة في التحقق

```go
func AssertTrue(t *testing.T, got bool) {
	t.Helper()
	if !got {
		t.Errorf("got %v, want true", got)
	}
}

func AssertFalse(t *testing.T, got bool) {
	t.Helper()
	if got {
		t.Errorf("got %v, want false", got)
	}
}
```

وهذه هي الاختبارات

```go
func TestStack(t *testing.T) {
	t.Run("integer stack", func(t *testing.T) {
		myStackOfInts := new(StackOfInts)

		// check stack is empty
		AssertTrue(t, myStackOfInts.IsEmpty())

		// add a thing, then check it's not empty
		myStackOfInts.Push(123)
		AssertFalse(t, myStackOfInts.IsEmpty())

		// add another thing, pop it back again
		myStackOfInts.Push(456)
		value, _ := myStackOfInts.Pop()
		AssertEqual(t, value, 456)
		value, _ = myStackOfInts.Pop()
		AssertEqual(t, value, 123)
		AssertTrue(t, myStackOfInts.IsEmpty())
	})

	t.Run("string stack", func(t *testing.T) {
		myStackOfStrings := new(StackOfStrings)

		// check stack is empty
		AssertTrue(t, myStackOfStrings.IsEmpty())

		// add a thing, then check it's not empty
		myStackOfStrings.Push("123")
		AssertFalse(t, myStackOfStrings.IsEmpty())

		// add another thing, pop it back again
		myStackOfStrings.Push("456")
		value, _ := myStackOfStrings.Pop()
		AssertEqual(t, value, "456")
		value, _ = myStackOfStrings.Pop()
		AssertEqual(t, value, "123")
		AssertTrue(t, myStackOfStrings.IsEmpty())
	})
}
```

### المشكلات

- كود `StackOfStrings` و`StackOfInts` متطابق تقريبًا. ومع أن التكرار ليس دائمًا نهاية العالم، فهو كود إضافي للقراءة والكتابة والصيانة.
- ولأننا نكرّر المنطق في نوعين، اضطررنا إلى تكرار الاختبارات أيضًا.

نريد فعلًا التقاط _فكرة_ المكدس في نوع واحد، وأن تكون لدينا مجموعة اختبارات واحدة لها. وينبغي أن نضع قبعة إعادة الهيكلة (refactoring) الآن، أي لا ينبغي أن نغيّر الاختبارات لأننا نريد الحفاظ على السلوك نفسه.

بدون الـ generics، هذا ما _نستطيع_ فعله

```go
type StackOfInts = Stack
type StackOfStrings = Stack

type Stack struct {
	values []interface{}
}

func (s *Stack) Push(value interface{}) {
	s.values = append(s.values, value)
}

func (s *Stack) IsEmpty() bool {
	return len(s.values) == 0
}

func (s *Stack) Pop() (interface{}, bool) {
	if s.IsEmpty() {
		var zero interface{}
		return zero, false
	}

	index := len(s.values) - 1
	el := s.values[index]
	s.values = s.values[:index]
	return el, true
}
```

- ننشئ أسماء مستعارة (aliasing) لتطبيقينا السابقين `StackOfInts` و`StackOfStrings` إلى نوع موحّد جديد هو `Stack`
- أزلنا أمان الأنواع من `Stack` بجعل `values` [شريحة (slice)](https://github.com/quii/learn-go-with-tests/blob/main/arrays-and-slices.md) من `interface{}`

لتجربة هذا الكود، ستحتاج إلى إزالة قيود الأنواع من دوال التحقق لدينا:

```go
func AssertEqual(t *testing.T, got, want interface{})
```

إذا فعلت ذلك، فستظل اختباراتنا تنجح. فمن يحتاج الـ generics إذًا؟

### مشكلة التخلي عن أمان الأنواع

المشكلة الأولى هي نفسها التي رأيناها مع `AssertEquals` — فقدنا أمان الأنواع. فأصبحت الآن قادرًا على `Push` التفاح إلى مكدس من البرتقال.

وحتى لو امتلكنا الانضباط لعدم فعل ذلك، يبقى الكود غير ممتع للعمل معه، لأن الـ methods **التي تُرجع `interface{}` تكون مروّعة في التعامل معها**.

أضف الاختبار التالي،

```go
t.Run("interface stack DX is horrid", func(t *testing.T) {
	myStackOfInts := new(StackOfInts)

	myStackOfInts.Push(1)
	myStackOfInts.Push(2)
	firstNum, _ := myStackOfInts.Pop()
	secondNum, _ := myStackOfInts.Pop()
	AssertEqual(t, firstNum+secondNum, 3)
})
```

ستحصل على خطأ في الترجمة يوضّح ضعف فقدان أمان الأنواع:

```
invalid operation: operator + not defined on firstNum (variable of type interface{})
```

عندما تُرجع `Pop` النوع `interface{}` فهذا يعني أن المترجم لا يملك أي معلومة عن ماهية البيانات، ولذلك يقيّد بشدة ما نستطيع فعله. فهو لا يستطيع أن يعرف أنه ينبغي أن يكون عددًا صحيحًا، فلا يسمح لنا باستخدام المعامل `+`.

للتغلب على ذلك، يجب على المستدعي إجراء [تأكيد النوع (type assertion)](https://golang.org/ref/spec#Type_assertions) لكل قيمة.

```go
t.Run("interface stack dx is horrid", func(t *testing.T) {
	myStackOfInts := new(StackOfInts)

	myStackOfInts.Push(1)
	myStackOfInts.Push(2)
	firstNum, _ := myStackOfInts.Pop()
	secondNum, _ := myStackOfInts.Pop()

	// get our ints from out interface{}
	reallyFirstNum, ok := firstNum.(int)
	AssertTrue(t, ok) // need to check we definitely got an int out of the interface{}

	reallySecondNum, ok := secondNum.(int)
	AssertTrue(t, ok) // and again!

	AssertEqual(t, reallyFirstNum+reallySecondNum, 3)
})
```

وسيتكرر الإزعاج المتصاعد من هذا الاختبار مع كل مستخدم محتمل لتطبيق `Stack`، يا للقرف.

### أنواع البيانات الـ generic تنقذ الموقف

تمامًا كما يمكنك تعريف وسائط generic للدوال، يمكنك تعريف أنواع بيانات generic.

وهذا تطبيقنا الجديد لـ `Stack`، مع نوع بيانات generic.

```go
type Stack[T any] struct {
	values []T
}

func (s *Stack[T]) Push(value T) {
	s.values = append(s.values, value)
}

func (s *Stack[T]) IsEmpty() bool {
	return len(s.values) == 0
}

func (s *Stack[T]) Pop() (T, bool) {
	if s.IsEmpty() {
		var zero T
		return zero, false
	}

	index := len(s.values) - 1
	el := s.values[index]
	s.values = s.values[:index]
	return el, true
}
```

وهذه هي الاختبارات، وتُظهرها تعمل كما نحب أن تعمل، مع أمان كامل للأنواع.

```go
func TestStack(t *testing.T) {
	t.Run("integer stack", func(t *testing.T) {
		myStackOfInts := new(Stack[int])

		// check stack is empty
		AssertTrue(t, myStackOfInts.IsEmpty())

		// add a thing, then check it's not empty
		myStackOfInts.Push(123)
		AssertFalse(t, myStackOfInts.IsEmpty())

		// add another thing, pop it back again
		myStackOfInts.Push(456)
		value, _ := myStackOfInts.Pop()
		AssertEqual(t, value, 456)
		value, _ = myStackOfInts.Pop()
		AssertEqual(t, value, 123)
		AssertTrue(t, myStackOfInts.IsEmpty())

		// can get the numbers we put in as numbers, not untyped interface{}
		myStackOfInts.Push(1)
		myStackOfInts.Push(2)
		firstNum, _ := myStackOfInts.Pop()
		secondNum, _ := myStackOfInts.Pop()
		AssertEqual(t, firstNum+secondNum, 3)
	})
}
```

ستلاحظ أن صياغة تعريف أنواع البيانات الـ generic متوافقة مع تعريف الوسائط الـ generic للدوال.

```go
type Stack[T any] struct {
	values []T
}
```

إنها _تقريبًا_ نفس ما كانت عليه من قبل، غير أننا نقول إن **نوع المكدس يقيّد أنواع القيم التي يمكنك العمل بها**.

وعندما تنشئ `Stack[Orange]` أو `Stack[Apple]`، فلن تسمح لك الـ methods المعرّفة على مكدسنا بتمرير سوى النوع المعيّن للمكدس الذي تعمل معه، ولن تُرجع سواه:

```go
func (s *Stack[T]) Pop() (T, bool)
```

يمكنك أن تتخيل أن تطبيقات الأنواع تُولّد لك بطريقة ما، حسب نوع المكدس الذي تنشئه:

```go
func (s *Stack[Orange]) Pop() (Orange, bool)
```

```go
func (s *Stack[Apple]) Pop() (Apple, bool)
```

الآن بعد أن أنجزنا إعادة الهيكلة هذه، يمكننا أن نحذف اختبار مكدس النصوص بأمان، لأننا لم نعد بحاجة إلى إثبات المنطق نفسه مرارًا وتكرارًا.

لاحظ أننا حتى الآن في أمثلة استدعاء الدوال الـ generic لم نكن بحاجة إلى تحديد الأنواع الـ generic. فمثلًا لاستدعاء `AssertEqual[T]` لا نحتاج إلى تحديد ما هو النوع `T` لأنه يمكن استنباطه من الوسائط. أما في الحالات التي لا يمكن فيها استنباط الأنواع الـ generic، فتحتاج إلى تحديد الأنواع عند استدعاء الدالة. والصياغة هي نفسها عند تعريف الدالة، أي تحدد الأنواع داخل أقواس مربعة قبل الوسائط.

ولمثال ملموس، تأمّل إنشاء دالة إنشاء (constructor) لـ `Stack[T]`.
```go
func NewStack[T any]() *Stack[T] {
	return new(Stack[T])
}
```
لاستخدام دالة الإنشاء هذه لإنشاء مكدس من الأعداد الصحيحة ومكدس من النصوص مثلًا، تستدعيها هكذا:
```go
myStackOfInts := NewStack[int]()
myStackOfStrings := NewStack[string]()
```

وهذا هو تطبيق `Stack` والاختبارات بعد إضافة دالة الإنشاء.

```go
type Stack[T any] struct {
	values []T
}

func NewStack[T any]() *Stack[T] {
	return new(Stack[T])
}

func (s *Stack[T]) Push(value T) {
	s.values = append(s.values, value)
}

func (s *Stack[T]) IsEmpty() bool {
	return len(s.values) == 0
}

func (s *Stack[T]) Pop() (T, bool) {
	if s.IsEmpty() {
		var zero T
		return zero, false
	}

	index := len(s.values) - 1
	el := s.values[index]
	s.values = s.values[:index]
	return el, true
}
```

```go
func TestStack(t *testing.T) {
	t.Run("integer stack", func(t *testing.T) {
		myStackOfInts := NewStack[int]()

		// check stack is empty
		AssertTrue(t, myStackOfInts.IsEmpty())

		// add a thing, then check it's not empty
		myStackOfInts.Push(123)
		AssertFalse(t, myStackOfInts.IsEmpty())

		// add another thing, pop it back again
		myStackOfInts.Push(456)
		value, _ := myStackOfInts.Pop()
		AssertEqual(t, value, 456)
		value, _ = myStackOfInts.Pop()
		AssertEqual(t, value, 123)
		AssertTrue(t, myStackOfInts.IsEmpty())

		// can get the numbers we put in as numbers, not untyped interface{}
		myStackOfInts.Push(1)
		myStackOfInts.Push(2)
		firstNum, _ := myStackOfInts.Pop()
		secondNum, _ := myStackOfInts.Pop()
		AssertEqual(t, firstNum+secondNum, 3)
	})
}
```


باستخدام نوع بيانات generic أصبح لدينا:

- تقليل تكرار منطق مهم.
- جعل `Pop` تُرجع `T`، فإذا أنشأنا `Stack[int]` فسنحصل عمليًا على `int` من `Pop`؛ ويمكننا الآن استخدام `+` دون الحاجة إلى ألعاب تأكيد النوع.
- منع سوء الاستخدام وقت الترجمة. فلا يمكنك `Push` برتقال إلى مكدس تفاح.

## الخلاصة

ينبغي أن يكون هذا الفصل قد أعطاك لمحة عن صياغة الـ generics، وبعض الأفكار عن سبب كونها مفيدة. لقد كتبنا دوال `Assert` الخاصة بنا التي يمكننا إعادة استخدامها بأمان لتجربة أفكار أخرى حول الـ generics، ونفّذنا بنية بيانات بسيطة لتخزين أي نوع من البيانات نريده، بطريقة آمنة الأنواع.

### الـ generics أبسط من استخدام `interface{}` في معظم الحالات

إذا لم تكن لديك خبرة باللغات ذات الأنواع الثابتة، فقد لا تكون فائدة الـ generics واضحة فورًا، لكنني آمل أن تكون الأمثلة في هذا الفصل قد وضّحت المواضع التي لا تكون فيها لغة Go معبّرة كما نريد. وبشكل خاص، استخدام `interface{}` يجعل كودك:

- أقل أمانًا (خلط التفاح بالبرتقال)، ويتطلب معالجة أخطاء أكثر
- أقل تعبيرًا، فـ `interface{}` لا يخبرك بأي شيء عن البيانات
- أكثر عرضة للاعتماد على [الـ reflection](https://github.com/quii/learn-go-with-tests/blob/main/reflection.md)، وتأكيدات الأنواع وغيرها، ما يجعل كودك أصعب في التعامل وأكثر عرضة للأخطاء لأنه ينقل عمليات الفحص من وقت الترجمة إلى وقت التشغيل

استخدام اللغات ذات الأنواع الثابتة فعل من أفعال وصف القيود. وإذا أحسنت ذلك، فستصنع كودًا ليس آمنًا وسهل الاستخدام فحسب، بل أسهل في الكتابة أيضًا، لأن فضاء الحلول الممكنة أصغر.

وتمنحنا الـ generics طريقة جديدة للتعبير عن القيود في كودنا، وهي كما أظهرنا ستتيح لنا دمج وتبسيط كود لم يكن ممكنًا قبل Go 1.18.

### هل ستحوّل الـ generics لغة Go إلى Java؟

- لا.

هناك الكثير من [الـ FUD (الخوف وعدم اليقين والشك)](https://en.wikipedia.org/wiki/Fear,_uncertainty,_and_doubt) في مجتمع Go بشأن أن تؤدي الـ generics إلى تجريدات كابوسية وقواعد كود محيّرة. وعادةً ما يُستدرك على ذلك بعبارة "يجب استخدامها بحذر".

ومع أن هذا صحيح، فهو ليس نصيحة مفيدة بشكل خاص، لأن هذا ينطبق على أي ميزة لغوية.

لا يشتكي كثيرون من قدرتنا على تعريف الـ interfaces، وهي مثل الـ generics طريقة لوصف القيود داخل كودنا. فعندما تصف واجهة (interface) فإنك تتخذ قرارًا تصميميًا _قد يكون سيّئًا_، والـ generics ليست فريدة في قدرتها على إنتاج كود مربك ومزعج الاستخدام.

### أنت تستخدم الـ generics بالفعل

إذا كنت قد استخدمت المصفوفات أو الشرائح أو الـ maps؛ فقد _كنت بالفعل مستهلكًا لكود generic_.

```
var myApples []Apple
// You can't do this!
append(myApples, Orange{})
```

### التجريد ليس كلمة ممنوعة

من السهل السخرية من [AbstractSingletonProxyFactoryBean](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/aop/framework/AbstractSingletonProxyFactoryBean.html)، لكن دعنا لا نتظاهر بأن قاعدة كود بلا أي تجريد على الإطلاق ليست سيئة أيضًا. ومهمتك أن _تجمع_ المفاهيم المترابطة عند الحاجة، ليكون نظامك أسهل فهمًا وتغييرًا؛ لا أن يكون مجموعة من دوال وأنواع متشتتة تفتقر إلى الوضوح.

### [اجعله يعمل، اجعله صحيحًا، اجعله سريعًا](https://wiki.c2.com/?MakeItWorkMakeItRightMakeItFast#:~:text=%22Make%20it%20work%2C%20make%20it,to%20DesignForPerformance%20ahead%20of%20time.)

يقع الناس في مشكلات مع الـ generics عندما يجرّدون بسرعة كبيرة دون معلومات كافية لاتخاذ قرارات تصميم جيدة.

دورة TDD المتمثلة في الأحمر والأخضر وإعادة الهيكلة تعني أن لديك إرشادًا أكبر حول الكود الذي _تحتاجه فعلًا_ لتقديم سلوكك، **بدلًا من تخيّل تجريدات مسبقًا**؛ لكنك ما زلت بحاجة إلى الحذر.

لا توجد قواعد صارمة وثابتة هنا، لكن قاوم جعل الأشياء generic حتى ترى أن لديك تعميمًا مفيدًا. وعندما أنشأنا تطبيقات `Stack` المختلفة، بدأنا بشكل مهم بسلوك _ملموس_ مثل `StackOfStrings` و`StackOfInts` مدعومًا باختبارات. ومن كودنا _الحقيقي_ بدأنا نرى أنماطًا حقيقية، ومدعومين باختباراتنا، استطعنا استكشاف إعادة الهيكلة نحو حل أكثر عمومية.

غالبًا ما ينصحك الناس بألا تعمّم إلا عندما ترى الكود نفسه ثلاث مرات، ويبدو ذلك قاعدة أولية جيدة.

ومن المسارات الشائعة التي سلكتها في لغات برمجة أخرى:

- دورة TDD واحدة لدفع بعض السلوك
- دورة TDD أخرى لاختبار بعض السيناريوهات المرتبطة الأخرى

> همم، تبدو هذه الأشياء متشابهة - لكن قليلًا من التكرار أفضل من الارتباط بتجريد سيئ

- نم عليها وفكّر فيها غدًا
- دورة TDD أخرى

> حسنًا، أرغب في أن أرى إن كنت أستطيع تعميم هذا الشيء. لحسن الحظ أنني ذكي ووسيم لأنني أستخدم TDD، فأستطيع إعادة الهيكلة وقتما أشاء، وقد ساعدتني العملية على فهم السلوك الذي أحتاجه فعلًا قبل أن أفرط في التصميم.

- هذا التجريد يبدو لطيفًا! ما زالت الاختبارات تنجح، والكود أبسط
- أستطيع الآن حذف عدد من الاختبارات، فقد التقطت _جوهر_ السلوك وأزلت التفاصيل غير الضرورية.
