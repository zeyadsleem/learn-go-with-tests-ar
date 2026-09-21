---
title: Structs والـ Methods والـ Interfaces
weight: 60
---

# Structs والـ Methods والـ Interfaces

**[يمكنك العثور على كل كود هذا الفصل هنا](https://github.com/quii/learn-go-with-tests/tree/main/structs)**

لنفترض أننا نحتاج إلى بعض كود الهندسة لحساب محيط مستطيل بمعلومية الارتفاع والعرض. يمكننا كتابة دالة `Perimeter(width float64, height float64)`، حيث `float64` مخصصة للأعداد العشرية مثل `123.45`.

ومن المفترض أن تكون دورة التطوير الموجه بالاختبار مألوفة لك تمامًا الآن.

## اكتب الاختبار أولًا

```go
func TestPerimeter(t *testing.T) {
	got := Perimeter(10.0, 10.0)
	want := 40.0

	if got != want {
		t.Errorf("got %.2f want %.2f", got, want)
	}
}
```

هل لاحظت نص التنسيق الجديد؟ الحرف `f` مخصص لـ `float64`، و`.2` تعني طباعة منزلتين عشريتين.

## جرّب تشغيل الاختبار

`./shapes_test.go:6:9: undefined: Perimeter`

## اكتب أصغر قدر من الكود ليعمل الاختبار وافحص مخرجاته الفاشلة

```go
func Perimeter(width float64, height float64) float64 {
	return 0
}
```

والنتيجة `shapes_test.go:10: got 0.00 want 40.00`.

## اكتب كودًا كافيًا لنجاح الاختبار

```go
func Perimeter(width float64, height float64) float64 {
	return 2 * (width + height)
}
```

حتى الآن الأمر سهل. لننشئ الآن دالة اسمها `Area(width, height float64)` تُرجع مساحة مستطيل.

جرّب فعل ذلك بنفسك، متبعًا دورة التطوير الموجه بالاختبار.

من المفترض أن تنتهي إلى اختبارات كهذه:

```go
func TestPerimeter(t *testing.T) {
	got := Perimeter(10.0, 10.0)
	want := 40.0

	if got != want {
		t.Errorf("got %.2f want %.2f", got, want)
	}
}

func TestArea(t *testing.T) {
	got := Area(12.0, 6.0)
	want := 72.0

	if got != want {
		t.Errorf("got %.2f want %.2f", got, want)
	}
}
```

وإلى كود كهذا:

```go
func Perimeter(width float64, height float64) float64 {
	return 2 * (width + height)
}

func Area(width float64, height float64) float64 {
	return width * height
}
```

وربما سمعت أن مقارنة الأعداد العشرية بـ `!=`/`==` فكرة سيئة، بسبب طريقة تمثيلها في الذاكرة:

```go
func Example_floatComparison() {
	fmt.Println(0.1+0.2 == 0.3)

	var a, b, c float64 = 0.1, 0.2, 0.3
	fmt.Println(a+b == c)

	// Output:
	// true
	// false
}
```

المقارنة الثانية نتيجتها `false` لأن `0.1` و`0.2` لا يمكن تمثيلهما بدقة كـ `float64`، فجمعهما لا يهبط أيضًا على `0.3` بالضبط. أما الأولى فنتيجتها `true` فقط لأنها مكتوبة كتعبير حرفي (literal expression)، وGo تقيّم هذه بدقة غير محدودة، لا كـ `float64`، حتى تُسند إلى شيء ما.

لكن هذا اللايقين لا ينطبق على اختبار `Area` أعلاه: فكل القيم التي نستخدمها (`12.0` و`6.0` و`72.0` وهكذا) أعداد صحيحة، وهي قيم _يمكن_ لـ `float64` تمثيلها بدقة، وضرب عددين ممثلين بدقة، ما دامت النتيجة في النطاق، ينتج نتيجة أخرى قابلة للتمثيل بدقة. ولا تصبح المقارنة الدقيقة غير آمنة إلا عند إدخال قيم أو حسابات غير قابلة للتمثيل بدقة، مثل `0.1`، أو نتائج القسمة. وإذا وجدت نفسك تكتب اختبارات كهذه، فاستخدم مقارنة بحد تسامح مقبول بدلًا من ذلك، مثل [`cmp.Diff` مع `cmpopts.EquateApprox`](https://pkg.go.dev/github.com/google/go-cmp/cmp/cmpopts#EquateApprox) من `go-cmp`.


## إعادة الهيكلة

كودنا يؤدي المهمة، لكنه لا يحتوي على أي شيء صريح عن المستطيلات. وقد يحاول مطور غير منتبه تمرير عرض وارتفاع مثلث إلى هذه الدوال دون أن يدرك أنها ستُرجع إجابة خاطئة.

كان يمكننا ببساطة إعطاء الدوال أسماء أكثر تحديدًا مثل `RectangleArea`. لكن الحل الأنيق هو تعريف _type_ خاص بنا اسمه `Rectangle` يغلّف هذا المفهوم لنا.

ويمكننا إنشاء type بسيط باستخدام **struct**. فـ [الـ struct](https://golang.org/ref/spec#Struct_types) مجرد مجموعة مُسمّاة من الحقول يمكنك تخزين بياناتك فيها.

عرّف struct في ملف `shapes.go` هكذا:

```go
type Rectangle struct {
	Width  float64
	Height float64
}
```

الآن لنعد هيكلة الاختبارات لتستخدم `Rectangle` بدلًا من `float64` المجردة.

```go
func TestPerimeter(t *testing.T) {
	rectangle := Rectangle{10.0, 10.0}
	got := Perimeter(rectangle)
	want := 40.0

	if got != want {
		t.Errorf("got %.2f want %.2f", got, want)
	}
}

func TestArea(t *testing.T) {
	rectangle := Rectangle{12.0, 6.0}
	got := Area(rectangle)
	want := 72.0

	if got != want {
		t.Errorf("got %.2f want %.2f", got, want)
	}
}
```

تذكّر تشغيل اختباراتك قبل محاولة الإصلاح. ومن المفترض أن تعرض الاختبارات خطأً مفيدًا مثل:

```text
./shapes_test.go:7:18: not enough arguments in call to Perimeter
    have (Rectangle)
    want (float64, float64)
```

ويمكنك الوصول إلى حقول الـ struct بصياغة `myStruct.field`.

عدّل الدالتين لإصلاح الاختبار.

```go
func Perimeter(rectangle Rectangle) float64 {
	return 2 * (rectangle.Width + rectangle.Height)
}

func Area(rectangle Rectangle) float64 {
	return rectangle.Width * rectangle.Height
}
```

آمل أن توافقني أن تمرير `Rectangle` إلى دالة يعبّر عن قصدنا بوضوح أكبر، لكن هناك فوائد أخرى لاستخدام الـ structs سنغطيها لاحقًا.

متطلبنا التالي هو كتابة دالة `Area` للدوائر.

## اكتب الاختبار أولًا

```go
func TestArea(t *testing.T) {

	t.Run("rectangles", func(t *testing.T) {
		rectangle := Rectangle{12, 6}
		got := Area(rectangle)
		want := 72.0

		if got != want {
			t.Errorf("got %g want %g", got, want)
		}
	})

	t.Run("circles", func(t *testing.T) {
		circle := Circle{10}
		got := Area(circle)
		want := 314.1592653589793

		if got != want {
			t.Errorf("got %g want %g", got, want)
		}
	})

}
```

كما ترى، حُلّ الحرف `g` محل `f`، ولسبب وجيه.
فاستخدام `g` سيطبع عددًا عشريًا أدق في رسالة الخطأ \([خيارات fmt](https://golang.org/pkg/fmt/)\).
فمثلًا، باستخدام نصف قطر 1.5 في حساب مساحة دائرة، سيعرض `f` القيمة `7.068583` بينما سيعرض `g` القيمة `7.0685834705770345`.

## جرّب تشغيل الاختبار

`./shapes_test.go:28:13: undefined: Circle`

## اكتب أصغر قدر من الكود ليعمل الاختبار وافحص مخرجاته الفاشلة

علينا تعريف نوع `Circle`.

```go
type Circle struct {
	Radius float64
}
```

جرّب الآن تشغيل الاختبارات مرة أخرى:

`./shapes_test.go:29:14: cannot use circle (type Circle) as type Rectangle in argument to Area`

تسمح لك بعض لغات البرمجة بفعل شيء كهذا:

```go
func Area(circle Circle) float64       {}
func Area(rectangle Rectangle) float64 {}
```

لكن لا يمكنك ذلك في Go:

`./shapes.go:20:32: Area redeclared in this block`

ولدينا خياران:

* يمكن أن تكون لديك دوال بالاسم نفسه في _حزم_ مختلفة. فيمكننا إنشاء `Area(Circle)` في حزمة جديدة، لكن ذلك يبدو مبالغًا فيه هنا.
* يمكننا تعريف [_methods_](https://golang.org/ref/spec#Method_declarations) على الأنواع التي عرّفناها حديثًا بدلًا من ذلك.

### ما هي الـ methods؟

حتى الآن كتبنا _دوال_ (functions) فقط، لكننا كنا نستخدم بعض الـ methods. فعندما نستدعي `t.Errorf` نستدعي الـ method `Errorf` على النسخة `t` ‏(`testing.T`) الخاصة بنا.

الـ method دالة لها مستقبِل (receiver).
ويقوم تعريف الـ method بربط معرّف، هو اسم الـ method، بالـ method، ويربط الـ method بالنوع الأساسي للمستقبِل.

الـ methods شبيهة جدًا بالدوال، لكنها تُستدعى على نسخة من نوع معين. فبينما يمكنك استدعاء الدوال في أي مكان تشاء، مثل `Area(rectangle)`، لا يمكنك استدعاء الـ methods إلا على "أشياء".

ومثال عملي سيساعد، فلنغيّر اختباراتنا أولًا لتستدعي methods، ثم نصلح الكود.

```go
func TestArea(t *testing.T) {

	t.Run("rectangles", func(t *testing.T) {
		rectangle := Rectangle{12, 6}
		got := rectangle.Area()
		want := 72.0

		if got != want {
			t.Errorf("got %g want %g", got, want)
		}
	})

	t.Run("circles", func(t *testing.T) {
		circle := Circle{10}
		got := circle.Area()
		want := 314.1592653589793

		if got != want {
			t.Errorf("got %g want %g", got, want)
		}
	})

}
```

وإذا حاولنا تشغيل الاختبارات، نحصل على:

```text
./shapes_test.go:19:19: rectangle.Area undefined (type Rectangle has no field or method Area)
./shapes_test.go:29:16: circle.Area undefined (type Circle has no field or method Area)
```

> type Circle has no field or method Area

أود أن أؤكد مرة أخرى كم أن المترجم رائع هنا. فمن المهم جدًا أن تأخذ وقتك في قراءة رسائل الخطأ التي تحصل عليها ببطء، فذلك سيساعدك على المدى الطويل.

## اكتب أصغر قدر من الكود ليعمل الاختبار وافحص مخرجاته الفاشلة

لنضف بعض الـ methods إلى أنواعنا.

```go
type Rectangle struct {
	Width  float64
	Height float64
}

func (r Rectangle) Area() float64 {
	return 0
}

type Circle struct {
	Radius float64
}

func (c Circle) Area() float64 {
	return 0
}
```

صياغة تعريف الـ methods تكاد تكون نفس صياغة الدوال، وهذا لأنها شديدة الشبه بها. والفرق الوحيد هو صياغة مستقبِل الـ method `func (receiverName ReceiverType) MethodName(args)`.

وعندما تُستدعى الـ method على متغير من ذلك النوع، تحصل على مرجعك إلى بياناته عبر متغير `receiverName`. وفي كثير من لغات البرمجة الأخرى يحدث هذا ضمنيًا وتصل إلى المستقبِل عبر `this`.

ومن العُرف في Go أن يكون متغير المستقبِل هو الحرف الأول من النوع.

```
r Rectangle
```

وإذا حاولت إعادة تشغيل الاختبارات، فمن المفترض أن تُترجم الآن وتعطيك مخرجات فاشلة.

## اكتب كودًا كافيًا لنجاح الاختبار

لنجعل الآن اختبارات المستطيل تنجح بإصلاح الـ method الجديدة.

```go
func (r Rectangle) Area() float64 {
	return r.Width * r.Height
}
```

وإذا أعدت تشغيل الاختبارات، فمن المفترض أن تنجح اختبارات المستطيل بينما تظل الدائرة فاشلة.

ولجعل دالة `Area` الخاصة بالدائرة تنجح، سنستعير الثابت `Pi` من حزمة `math` (تذكّر استيرادها).

```go
func (c Circle) Area() float64 {
	return math.Pi * c.Radius * c.Radius
}
```

## إعادة الهيكلة

يوجد بعض التكرار في اختباراتنا.

كل ما نريده هو أخذ مجموعة من _الأشكال_، واستدعاء الـ method `Area()` عليها، ثم فحص النتيجة.

ونريد أن نستطيع كتابة نوع من دالة `checkArea` نمرر إليها المستطيلات والدوائر معًا، لكن يفشل الترجمة إذا حاولنا تمرير شيء ليس شكلًا.

ومع Go يمكننا ترجمة هذا القصد باستخدام **الواجهات (interfaces)**.

[الواجهات](https://golang.org/ref/spec#Interface_types) مفهوم قوي جدًا في اللغات ذات الأنواع الثابتة مثل Go، لأنها تتيح لك بناء دوال يمكن استخدامها مع أنواع مختلفة وبناء كود شديد الفصل (decoupled) مع الحفاظ على أمان الأنواع.

لنقدّم ذلك بإعادة هيكلة اختباراتنا.

```go
func TestArea(t *testing.T) {

	checkArea := func(t testing.TB, shape Shape, want float64) {
		t.Helper()
		got := shape.Area()
		if got != want {
			t.Errorf("got %g want %g", got, want)
		}
	}

	t.Run("rectangles", func(t *testing.T) {
		rectangle := Rectangle{12, 6}
		checkArea(t, rectangle, 72.0)
	})

	t.Run("circles", func(t *testing.T) {
		circle := Circle{10}
		checkArea(t, circle, 314.1592653589793)
	})

}
```

نُنشئ دالة مساعدة كما فعلنا في تمارين أخرى، لكننا هذه المرة نطلب تمرير `Shape`. وإذا حاولنا استدعاءها بشيء ليس شكلًا، فلن يُترجم الكود.

وكيف يصبح شيء ما شكلًا؟ نكتفي بإخبار Go ما هو `Shape` بتعريف واجهة:

```go
type Shape interface {
	Area() float64
}
```

نُنشئ `type` جديدًا تمامًا كما فعلنا مع `Rectangle` و`Circle`، لكنه هذه المرة `interface` وليس `struct`.

وبمجرد إضافة ذلك إلى الكود، ستنجح الاختبارات.

### لحظة، ماذا؟

هذا مختلف تمامًا عن الواجهات في معظم لغات البرمجة الأخرى. فعادةً يجب أن تكتب كودًا يقول `My type Foo implements interface Bar`.

لكن في حالتنا:

* `Rectangle` لديها method اسمها `Area` تُرجع `float64`، لذا فهي تحقق واجهة `Shape`
* `Circle` لديها method اسمها `Area` تُرجع `float64`، لذا فهي تحقق واجهة `Shape`
* `string` ليس لديها method كهذه، لذا فهي لا تحقق الواجهة
* وهكذا

في Go **حل الواجهات ضمني (implicit)**. فإذا طابق النوع الذي تمرره ما تطلبه الواجهة، سيُترجم الكود.

### فصل الارتباط (Decoupling)

لاحظ أن دالتنا المساعدة لا تحتاج إلى أن تشغل نفسها بما إذا كان الشكل `Rectangle` أم `Circle` أم `Triangle`. فبتعريف واجهة، تصبح الدالة المساعدة _مفصولة_ عن الأنواع الملموسة ولا تملك إلا الـ method التي تحتاجها لأداء عملها.

وهذا الأسلوب في استخدام الواجهات للإعلان عن **ما تحتاجه فقط** مهم جدًا في تصميم البرمجيات، وسنغطيه بتفصيل أكبر في أقسام لاحقة.

## مزيد من إعادة الهيكلة

بعد أن أصبح لديك بعض الفهم للـ structs، يمكننا تقديم "الاختبارات القائمة على الجدول" (table driven tests).

تكون [الاختبارات القائمة على الجدول](https://go.dev/wiki/TableDrivenTests) مفيدة عندما تريد بناء قائمة من حالات الاختبار يمكن اختبارها بالطريقة نفسها.

```go
func TestArea(t *testing.T) {

	areaTests := []struct {
		shape Shape
		want  float64
	}{
		{Rectangle{12, 6}, 72.0},
		{Circle{10}, 314.1592653589793},
	}

	for _, tt := range areaTests {
		got := tt.shape.Area()
		if got != tt.want {
			t.Errorf("got %g want %g", got, tt.want)
		}
	}

}
```

الصياغة الجديدة الوحيدة هنا هي إنشاء "struct مجهول الاسم" اسمه `areaTests`. فنحن نعرّف شريحة من الـ structs باستخدام `[]struct` بحقلين: `shape` و`want`. ثم نملأ الشريحة بحالات الاختبار.

ثم نمر عليها مثل أي شريحة أخرى، ونستخدم حقول الـ struct لتشغيل اختباراتنا.

ويمكنك أن ترى كم سيكون سهلًا على المطور تقديم شكل جديد، وتنفيذ `Area`، ثم إضافته إلى حالات الاختبار. وبالإضافة إلى ذلك، إذا اكتُشف خطأ في `Area`، فمن السهل جدًا إضافة حالة اختبار جديدة تكشفه قبل إصلاحه.

يمكن أن تكون الاختبارات القائمة على الجدول عنصرًا رائعًا في صندوق أدواتك، لكن تأكد من أنك تحتاج فعلًا إلى الضجيج الإضافي في الاختبارات.
فهي مناسبة تمامًا عندما تريد اختبار تنفيذات مختلفة لواجهة ما، أو إذا كانت البيانات الممررة إلى دالة لها متطلبات مختلفة كثيرة تحتاج إلى اختبار.

لنوضح كل هذا بإضافة شكل آخر واختباره: المثلث.

## اكتب الاختبار أولًا

إضافة اختبار جديد لشكلنا الجديد سهلة جدًا. أضف فقط `{Triangle{12, 6}, 36.0},` إلى قائمتنا.

```go
func TestArea(t *testing.T) {

	areaTests := []struct {
		shape Shape
		want  float64
	}{
		{Rectangle{12, 6}, 72.0},
		{Circle{10}, 314.1592653589793},
		{Triangle{12, 6}, 36.0},
	}

	for _, tt := range areaTests {
		got := tt.shape.Area()
		if got != tt.want {
			t.Errorf("got %g want %g", got, tt.want)
		}
	}

}
```

## جرّب تشغيل الاختبار

تذكّر، واصل محاولة تشغيل الاختبار ودع المترجم يرشدك إلى الحل.

## اكتب أصغر قدر من الكود ليعمل الاختبار وافحص مخرجاته الفاشلة

`./shapes_test.go:25:4: undefined: Triangle`

لم نعرّف `Triangle` بعد.

```go
type Triangle struct {
	Base   float64
	Height float64
}
```

جرّب مرة أخرى:

```text
./shapes_test.go:25:8: cannot use Triangle literal (type Triangle) as type Shape in field value:
    Triangle does not implement Shape (missing Area method)
```

إنه يخبرنا أننا لا نستطيع استخدام `Triangle` كشكل لأنه لا يملك method اسمها `Area()`، فأضف تنفيذًا فارغًا ليعمل الاختبار.

```go
func (t Triangle) Area() float64 {
	return 0
}
```

أخيرًا يُترجم الكود ونحصل على خطئنا:

`shapes_test.go:31: got 0.00 want 36.00`

## اكتب كودًا كافيًا لنجاح الاختبار

```go
func (t Triangle) Area() float64 {
	return (t.Base * t.Height) * 0.5
}
```

واختباراتنا تنجح!

## إعادة الهيكلة

مرة أخرى، التنفيذ جيد لكن اختباراتنا تحتاج إلى بعض التحسين.

فعندما تنظر سريعًا إلى:

```
{Rectangle{12, 6}, 72.0},
{Circle{10}, 314.1592653589793},
{Triangle{12, 6}, 36.0},
```

لا يتضح فورًا ماذا تمثل كل هذه الأرقام، وينبغي أن يكون هدفك أن تكون اختباراتك سهلة الفهم.

حتى الآن لم يُعرض عليك إلا تعريف نسخ من الـ structs بصياغة `MyStruct{val1, val2}`، لكن يمكنك اختياريًا تسمية الحقول.

لنرَ كيف يبدو ذلك:

```
        {shape: Rectangle{Width: 12, Height: 6}, want: 72.0},
        {shape: Circle{Radius: 10}, want: 314.1592653589793},
        {shape: Triangle{Base: 12, Height: 6}, want: 36.0},
```

في كتاب [Test-Driven Development by Example](https://g.co/kgs/yCzDLF) يعيد Kent Beck هيكلة بعض الاختبارات إلى حد ما ثم يؤكد:

> يصبح الاختبار أكثر وضوحًا لنا، وكأنه تأكيد على حقيقة، **لا سلسلة من العمليات**

(التشديد في الاقتباس من عندي)

والآن اختباراتنا، أو بالأحرى قائمة حالات الاختبار، أصبحت تأكيدات على حقائق عن الأشكال ومساحاتها.

## تأكد أن مخرجات اختبارك مفيدة

تذكّر حين كنا ننفذ `Triangle` وكان لدينا الاختبار الفاشل؟ لقد طبع `shapes_test.go:31: got 0.00 want 36.00`.

كنا نعرف أن ذلك متعلق بـ `Triangle` لأننا كنا نعمل عليه للتو.
لكن ماذا لو تسلل خطأ إلى النظام في إحدى 20 حالة في الجدول؟
كيف سيعرف المطور أي حالة فشلت؟
فهذه ليست تجربة رائعة للمطور، إذ سيكون عليه فحص الحالات يدويًا ليعرف أي حالة فشلت فعلًا.

يمكننا تغيير رسالة الخطأ إلى `%#v got %g want %g`. فنص التنسيق `%#v` سيطبع الـ struct الخاص بنا مع قيم حقوله، ليستطيع المطور رؤية الخصائص التي تُختبر لمحة واحدة.

ولزيادة وضوح حالات اختبارنا أكثر، يمكننا إعادة تسمية الحقل `want` إلى شيء أكثر وصفًا مثل `hasArea`.

ونصيحة أخيرة مع الاختبارات القائمة على الجدول: استخدم `t.Run` وسمّ حالات الاختبار.

فبتغليف كل حالة في `t.Run` ستحصل على مخرجات اختبار أوضح عند الفشل، لأنها ستطبع اسم الحالة.

```text
--- FAIL: TestArea (0.00s)
    --- FAIL: TestArea/Rectangle (0.00s)
        shapes_test.go:33: main.Rectangle{Width:12, Height:6} got 72.00 want 72.10
```

ويمكنك تشغيل اختبارات محددة داخل جدولك بالأمر `go test -run TestArea/Rectangle`.

وهذا هو كود اختباراتنا النهائي الذي يجسد كل ذلك:

```go
func TestArea(t *testing.T) {

	areaTests := []struct {
		name    string
		shape   Shape
		hasArea float64
	}{
		{name: "Rectangle", shape: Rectangle{Width: 12, Height: 6}, hasArea: 72.0},
		{name: "Circle", shape: Circle{Radius: 10}, hasArea: 314.1592653589793},
		{name: "Triangle", shape: Triangle{Base: 12, Height: 6}, hasArea: 36.0},
	}

	for _, tt := range areaTests {
		// using tt.name from the case to use it as the `t.Run` test name
		t.Run(tt.name, func(t *testing.T) {
			got := tt.shape.Area()
			if got != tt.hasArea {
				t.Errorf("%#v got %g want %g", tt.shape, got, tt.hasArea)
			}
		})

	}

}
```

## الخلاصة

كان هذا مزيدًا من التدريب على التطوير الموجه بالاختبار، إذ مررنا على حلولنا لمشكلات رياضية أساسية وتعلمنا ميزات لغوية جديدة بدافع من اختباراتنا.

* تعريف الـ structs لبناء أنواع بيانات خاصة بك، تتيح لك تجميع البيانات المرتبطة معًا وجعل قصد كودك أوضح
* تعريف الواجهات لتتمكن من تحديد دوال يمكن استخدامها مع أنواع مختلفة \([تعدد الأشكال الخاص (ad hoc polymorphism)](https://en.wikipedia.org/wiki/Ad_hoc_polymorphism)\)
* إضافة الـ methods لتتمكن من إضافة وظائف إلى أنواع بياناتك، ولكي تنفذ الواجهات
* الاختبارات القائمة على الجدول لجعل تحققاتك أوضح ومجموعات اختباراتك أسهل في التوسيع والصيانة

كان هذا فصلًا مهمًا لأننا بدأنا الآن نعرّف أنواعنا الخاصة. وفي اللغات ذات الأنواع الثابتة مثل Go، تُعد القدرة على تصميم أنواعك الخاصة أمرًا أساسيًا لبناء برمجيات سهلة الفهم والتجميع والاختبار.

الواجهات أداة رائعة لإخفاء التعقيد عن بقية أجزاء النظام. وفي حالتنا، لم يكن على _كود_ المساعدة في اختبارنا أن يعرف الشكل الدقيق الذي يتحقق منه، بل فقط كيف "يسأل" عن مساحته.

ومع تعمقك في Go، ستبدأ في رؤية القوة الحقيقية للواجهات وللمكتبة القياسية. وستتعلم عن واجهات معرَّفة في المكتبة القياسية تُستخدم _في كل مكان_، وبتنفيذها على أنواعك الخاصة، يمكنك إعادة استخدام قدر كبير من الوظائف الرائعة بسرعة كبيرة.
