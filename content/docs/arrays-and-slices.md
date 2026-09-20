---
title: المصفوفات والشرائح
weight: 50
---

# المصفوفات والشرائح

**[يمكنك العثور على كل كود هذا الفصل هنا](https://github.com/quii/learn-go-with-tests/tree/main/arrays)**

تتيح لك المصفوفات (arrays) تخزين عدة عناصر من النوع نفسه في متغير واحد بترتيب معين.

وعندما يكون لديك مصفوفات، فمن الشائع جدًا أن تحتاج إلى المرور عليها عنصرًا عنصرًا. لذا دعنا نستخدم [معرفتنا الجديدة بـ `for`](iteration.md) لبناء دالة `Sum` تأخذ مصفوفة أرقام وتُرجع المجموع.

لنستخدم مهاراتنا في التطوير الموجه بالاختبار.

## اكتب الاختبار أولًا

أنشئ مجلدًا جديدًا للعمل فيه. ثم أنشئ ملفًا جديدًا اسمه `sum_test.go` وضع فيه ما يلي:

```go
package main

import "testing"

func TestSum(t *testing.T) {

	numbers := [5]int{1, 2, 3, 4, 5}

	got := Sum(numbers)
	want := 15

	if got != want {
		t.Errorf("got %d want %d given, %v", got, want, numbers)
	}
}
```

للمصفوفات _سعة ثابتة_ تحددها عند تعريف المتغير. ويمكننا تهيئة المصفوفة بطريقتين:

* \[N\]type{value1, value2, ..., valueN} مثل `numbers := [5]int{1, 2, 3, 4, 5}`
* \[...\]type{value1, value2, ..., valueN} مثل `numbers := [...]int{1, 2, 3, 4, 5}`

ومن المفيد أحيانًا طباعة المدخلات إلى الدالة في رسالة الخطأ أيضًا.
وهنا نستخدم العنصر البديل `%v` لطباعة الصيغة "الافتراضية"، وهي مناسبة للمصفوفات.

[اقرأ المزيد عن نصوص التنسيق](https://golang.org/pkg/fmt/)

## جرّب تشغيل الاختبار

إذا كنت قد هيأت go mod بالأمر `go mod init main` فسيظهر لك خطأ
`_testmain.go:13:2: cannot import "main"`. والسبب أن الحزمة main، حسب الممارسة الشائعة، لا تحتوي إلا على تركيب الحزم الأخرى ولا تحتوي كودًا قابلًا لاختبار الوحدات، ولذلك لا تسمح Go باستيراد حزمة اسمها `main`.

ولإصلاح ذلك، يمكنك إعادة تسمية الوحدة main في `go.mod` إلى أي اسم آخر.

وبعد إصلاح الخطأ أعلاه، إذا شغّلت `go test` فسيفشل المترجم بالخطأ المألوف
`./sum_test.go:10:15: undefined: Sum`. ويمكننا الآن المتابعة بكتابة الدالة الفعلية التي نريد اختبارها.

## اكتب أصغر قدر من الكود ليعمل الاختبار وافحص مخرجاته الفاشلة

في الملف `sum.go`:

```go
package main

func Sum(numbers [5]int) int {
	return 0
}
```

من المفترض أن يفشل اختبارك الآن بـ _رسالة خطأ واضحة_

`sum_test.go:13: got 0 want 15 given, [1 2 3 4 5]`

## اكتب كودًا كافيًا لنجاح الاختبار

```go
func Sum(numbers [5]int) int {
	sum := 0
	for i := 0; i < 5; i++ {
		sum += numbers[i]
	}
	return sum
}
```

للحصول على قيمة من مصفوفة عند فهرس معين، استخدم ببساطة صياغة `array[index]`.
وفي هذه الحالة نستخدم `for` للمرور 5 مرات على المصفوفة وإضافة كل عنصر إلى `sum`.

## إعادة الهيكلة

لنقدّم [`range`](https://gobyexample.com/range) للمساعدة في تنسيق كودنا.

```go
func Sum(numbers [5]int) int {
	sum := 0
	for _, number := range numbers {
		sum += number
	}
	return sum
}
```

تتيح لك `range` المرور على مصفوفة. وفي كل تكرار تُرجع `range` قيمتين: الفهرس والقيمة.
ونحن نختار تجاهل قيمة الفهرس باستخدام `_`، وهي [المعرّف الفارغ (blank identifier)](https://golang.org/doc/effective_go.html#blank).

### المصفوفات وأنواعها

من الخصائص المثيرة للمصفوفات أن الحجم مُشفَّر في نوعها. فإذا حاولت تمرير `[4]int` إلى دالة تتوقع `[5]int`، فلن يُترجم الكود.
فهما نوعان مختلفان، تمامًا كما لو حاولت تمرير `string` إلى دالة تريد `int`.

وربما تفكر أن ثبات طول المصفوفات مُرهق، وأنك في معظم الأوقات لن تستخدمها أصلًا!

وتمتلك Go _الشرائح_ (slices) التي لا يشفر نوعها حجم المجموعة، ويمكن أن يكون لها أي حجم.

ومتطلبنا التالي هو جمع مجموعات بأحجام متفاوتة.

## اكتب الاختبار أولًا

سنستخدم الآن [نوع الشريحة (slice)][slice] الذي يتيح لنا مجموعات بأي حجم. والصياغة شبيهة جدًا بالمصفوفات، لكنك فقط تحذف الحجم عند تعريفها:

`mySlice := []int{1,2,3}` بدلًا من `myArray := [3]int{1,2,3}`

```go
func TestSum(t *testing.T) {

	t.Run("collection of 5 numbers", func(t *testing.T) {
		numbers := [5]int{1, 2, 3, 4, 5}

		got := Sum(numbers)
		want := 15

		if got != want {
			t.Errorf("got %d want %d given, %v", got, want, numbers)
		}
	})

	t.Run("collection of any size", func(t *testing.T) {
		numbers := []int{1, 2, 3}

		got := Sum(numbers)
		want := 6

		if got != want {
			t.Errorf("got %d want %d given, %v", got, want, numbers)
		}
	})

}
```

## جرّب تشغيل الاختبار

هذا لا يُترجم:

`./sum_test.go:22:13: cannot use numbers (type []int) as type [5]int in argument to Sum`

## اكتب أصغر قدر من الكود ليعمل الاختبار وافحص مخرجاته الفاشلة

المشكلة هنا أنه يمكننا إما:

* كسر الواجهة الحالية (API) بتغيير وسيط `Sum` ليكون شريحة بدلًا من مصفوفة. وعندما نفعل ذلك، سنُفسد يوم شخص ما على الأرجح، لأن اختبارنا _الآخر_ لن يُترجم بعد الآن!
* إنشاء دالة جديدة

وفي حالتنا لا أحد غيرنا يستخدم الدالة، لذا بدلًا من وجود دالتين للصيانة، لنجعلها دالة واحدة.

```go
func Sum(numbers []int) int {
	sum := 0
	for _, number := range numbers {
		sum += number
	}
	return sum
}
```

وإذا حاولت تشغيل الاختبارات فلن تُترجم لا يزال، وسيكون عليك تغيير الاختبار الأول ليمرر شريحة بدلًا من مصفوفة.

## اكتب كودًا كافيًا لنجاح الاختبار

اتضح أن إصلاح مشكلات المترجم كان كل ما نحتاجه هنا، والاختبارات تنجح!

## إعادة الهيكلة

لقد أعدنا هيكلة `Sum` بالفعل؛ فكل ما فعلناه هو استبدال المصفوفات بالشرائح، فلا حاجة إلى تغييرات إضافية.
وتذكّر أنه يجب ألا نهمل كود اختباراتنا في مرحلة إعادة الهيكلة، إذ يمكننا تحسين اختبارات `Sum` أكثر.

```go
func TestSum(t *testing.T) {

	t.Run("collection of 5 numbers", func(t *testing.T) {
		numbers := []int{1, 2, 3, 4, 5}

		got := Sum(numbers)
		want := 15

		if got != want {
			t.Errorf("got %d want %d given, %v", got, want, numbers)
		}
	})

	t.Run("collection of any size", func(t *testing.T) {
		numbers := []int{1, 2, 3}

		got := Sum(numbers)
		want := 6

		if got != want {
			t.Errorf("got %d want %d given, %v", got, want, numbers)
		}
	})

}
```

من المهم أن تتساءل عن قيمة اختباراتك. فلا ينبغي أن يكون الهدف هو امتلاك أكبر عدد ممكن من الاختبارات، بل امتلاك أكبر قدر ممكن من _الثقة_ في قاعدة كودك. فكثرة الاختبارات قد تتحول إلى مشكلة حقيقية وتضيف فقط أعباء صيانة. **فلكل اختبار تكلفة**.

وفي حالتنا هذه، ترى أن وجود اختبارين لهذه الدالة تكرار لا داعي له.
فإذا كانت تعمل مع شريحة بحجم معين، فمن المرجح جدًا أنها ستعمل مع شريحة بأي حجم (في حدود المعقول).

وتتضمن عدة اختبارات Go المدمجة [أداة تغطية](https://blog.golang.org/cover).
ومع أن السعي لتغطية 100% لا ينبغي أن يكون هدفك النهائي، فأداة التغطية تساعد في تحديد أجزاء كودك غير المغطاة بالاختبارات. وإذا كنت متزمتًا مع التطوير الموجه بالاختبار، فمن المرجح أن تكون تغطيتك قريبة من 100% على أي حال.

جرّب تنفيذ:

`go test -cover`

ومن المفترض أن ترى:

```bash
PASS
coverage: 100.0% of statements
```

احذف الآن أحد الاختبارين وافحص التغطية مرة أخرى.

وبعد أن رضينا أن لدينا دالة مُختبرة جيدًا، ينبغي أن تسجّل عملك الرائع في commit قبل مواجهة التحدي التالي.

نحتاج إلى دالة جديدة اسمها `SumAll` تأخذ عددًا متغيرًا من الشرائح، وتُرجع شريحة جديدة تحتوي على مجاميع كل شريحة مُرَّرة.

على سبيل المثال:

`SumAll([]int{1,2}, []int{0,9})` سيُرجع `[]int{3, 9}`

أو:

`SumAll([]int{1,1,1})` سيُرجع `[]int{3}`

## اكتب الاختبار أولًا

```go
func TestSumAll(t *testing.T) {

	got := SumAll([]int{1, 2}, []int{0, 9})
	want := []int{3, 9}

	if got != want {
		t.Errorf("got %v want %v", got, want)
	}
}
```

## جرّب تشغيل الاختبار

`./sum_test.go:23:9: undefined: SumAll`

## اكتب أصغر قدر من الكود ليعمل الاختبار وافحص مخرجاته الفاشلة

علينا تعريف `SumAll` حسب ما يريده اختبارنا.

تتيح لك Go كتابة [_دوال متغيرة الوسائط (variadic functions)_](https://gobyexample.com/variadic-functions) يمكنها أخذ عدد متغير من الوسائط.

```go
func SumAll(numbersToSum ...[]int) []int {
	return nil
}
```

هذا صحيح، لكن اختباراتنا لن تُترجم لا يزال!

`./sum_test.go:26:9: invalid operation: got != want (slice can only be compared to nil)`

لا تسمح لك Go باستخدام معاملات المساواة مع الشرائح. ويمكنك _بالطبع_ كتابة دالة تمر على كل من شريحتي `got` و`want` وتفحص قيمهما، لكن ماذا لو كان لدينا طريقة أسهل لفعل ذلك؟

منذ Go 1.21 صارت حزمة [slices](https://pkg.go.dev/slices#pkg-overview) القياسية متاحة، وفيها الدالة [slices.Equal](https://pkg.go.dev/slices#Equal) التي تقوم بمقارنة سطحية بسيطة للشرائح، دون أن تقلق بشأن الأنواع كما في الحالة أعلاه.
لاحظ أن هذه الدالة تتوقع أن تكون العناصر [قابلة للمقارنة (comparable)](https://pkg.go.dev/builtin#comparable).
لذا لا يمكن تطبيقها على شرائح بعناصر غير قابلة للمقارنة مثل الشرائح ثنائية الأبعاد.

لنمضِ قدمًا ونضع هذا موضع التطبيق!

```go
func TestSumAll(t *testing.T) {

	got := SumAll([]int{1, 2}, []int{0, 9})
	want := []int{3, 9}

	if !slices.Equal(got, want) {
		t.Errorf("got %v want %v", got, want)
	}
}
```

ومن المفترض أن تحصل على مخرجات اختبار كهذه:
`sum_test.go:30: got [] want [3 9]`

## اكتب كودًا كافيًا لنجاح الاختبار

ما نحتاج إلى فعله هو المرور على الوسائط المتغيرة، وحساب المجموع باستخدام دالتنا الحالية `Sum`، ثم إضافته إلى الشريحة التي سنُرجعها.

```go
func SumAll(numbersToSum ...[]int) []int {
	lengthOfNumbers := len(numbersToSum)
	sums := make([]int, lengthOfNumbers)

	for i, numbers := range numbersToSum {
		sums[i] = Sum(numbers)
	}

	return sums
}
```

الكثير من الأشياء الجديدة لتتعلمها!

هناك طريقة جديدة لإنشاء شريحة. فـ `make` تتيح لك إنشاء شريحة بسعة ابتدائية تساوي `len` للـ `numbersToSum` التي نحتاج إلى المرور عليها. وطول الشريحة هو عدد العناصر التي تحتويها `len(mySlice)`، أما السعة فهي عدد العناصر التي يمكن أن تستوعبها في المصفوفة الأساسية `cap(mySlice)`، فمثلًا `make([]int, 0, 5)` تنشئ شريحة طولها 0 وسعتها 5.

ويمكنك فهرسة الشرائح مثل المصفوفات بـ `mySlice[N]` للحصول على قيمة، أو إسناد قيمة جديدة لها بـ `=`

من المفترض أن تنجح الاختبارات الآن.

## إعادة الهيكلة

كما ذكرنا، للشرائح سعة. فإذا كانت لديك شريحة سعتها 2 وحاولت تنفيذ `mySlice[10] = 1` فستحصل على خطأ _وقت تشغيل_ (runtime error).

لكن يمكنك استخدام دالة `append` التي تأخذ شريحة وقيمة جديدة، ثم تُرجع شريحة جديدة تحتوي على كل العناصر.

```go
func SumAll(numbersToSum ...[]int) []int {
	var sums []int
	for _, numbers := range numbersToSum {
		sums = append(sums, Sum(numbers))
	}

	return sums
}
```

في هذا التنفيذ نقلق أقل بشأن السعة. نبدأ بشريحة فارغة `sums` ونضيف إليها نتيجة `Sum` أثناء مرورنا على الوسائط المتغيرة.

متطلبنا التالي هو تغيير `SumAll` إلى `SumAllTails`، بحيث تحسب مجاميع "ذيول" كل شريحة. وذيل المجموعة هو كل عناصرها ما عدا العنصر الأول ("الرأس").

## اكتب الاختبار أولًا

```go
func TestSumAllTails(t *testing.T) {
	got := SumAllTails([]int{1, 2}, []int{0, 9})
	want := []int{2, 9}

	if !slices.Equal(got, want) {
		t.Errorf("got %v want %v", got, want)
	}
}
```

## جرّب تشغيل الاختبار

`./sum_test.go:26:9: undefined: SumAllTails`

## اكتب أصغر قدر من الكود ليعمل الاختبار وافحص مخرجاته الفاشلة

أعد تسمية الدالة إلى `SumAllTails` وأعد تشغيل الاختبار.

`sum_test.go:30: got [3 9] want [2 9]`

## اكتب كودًا كافيًا لنجاح الاختبار

```go
func SumAllTails(numbersToSum ...[]int) []int {
	var sums []int
	for _, numbers := range numbersToSum {
		tail := numbers[1:]
		sums = append(sums, Sum(tail))
	}

	return sums
}
```

يمكن تقطيع الشرائح! والصياغة هي `slice[low:high]`. وإذا حذفت القيمة من أحد جانبي `:` فسيشمل كل شيء إلى ذلك الجانب. وفي حالتنا نقول "خذ من 1 إلى النهاية" بـ `numbers[1:]`. وربما ترغب في قضاء بعض الوقت بكتابة اختبارات أخرى حول الشرائح والتجربة بمعامل التقطيع لتعتاد عليه أكثر.

## إعادة الهيكلة

لا يوجد الكثير لنعيد هيكلته هذه المرة.

ما رأيك فيما سيحدث لو مرّرت شريحة فارغة إلى دالتنا؟ ما هو "ذيل" شريحة فارغة؟ وماذا يحدث عندما تطلب من Go أخذ كل العناصر من `myEmptySlice[1:]`؟

## اكتب الاختبار أولًا

```go
func TestSumAllTails(t *testing.T) {

	t.Run("make the sums of some slices", func(t *testing.T) {
		got := SumAllTails([]int{1, 2}, []int{0, 9})
		want := []int{2, 9}

		if !slices.Equal(got, want) {
			t.Errorf("got %v want %v", got, want)
		}
	})

	t.Run("safely sum empty slices", func(t *testing.T) {
		got := SumAllTails([]int{}, []int{3, 4, 5})
		want := []int{0, 9}

		if !slices.Equal(got, want) {
			t.Errorf("got %v want %v", got, want)
		}
	})

}
```

## جرّب تشغيل الاختبار

```text
panic: runtime error: slice bounds out of range [recovered]
    panic: runtime error: slice bounds out of range
```

أوه لا! من المهم ملاحظة أن الاختبار _تُرجم فعلًا_، لكنه _به خطأ في وقت التشغيل_.

فأخطاء وقت الترجمة صديقة لنا لأنها تساعدنا على كتابة برمجيات تعمل، أما أخطاء وقت التشغيل فأعداء لنا لأنها تؤثر على مستخدمينا.

## اكتب كودًا كافيًا لنجاح الاختبار

```go
func SumAllTails(numbersToSum ...[]int) []int {
	var sums []int
	for _, numbers := range numbersToSum {
		if len(numbers) == 0 {
			sums = append(sums, 0)
		} else {
			tail := numbers[1:]
			sums = append(sums, Sum(tail))
		}
	}

	return sums
}
```

## إعادة الهيكلة

في اختباراتنا بعض الكود المكرر حول التحقق مرة أخرى، فلنستخرجه إلى دالة.

```go
func TestSumAllTails(t *testing.T) {

	checkSums := func(t *testing.T, got, want []int) {
		t.Helper()
		if !slices.Equal(got, want) {
			t.Errorf("got %v want %v", got, want)
		}
	}

	t.Run("make the sums of tails of", func(t *testing.T) {
		got := SumAllTails([]int{1, 2}, []int{0, 9})
		want := []int{2, 9}
		checkSums(t, got, want)
	})

	t.Run("safely sum empty slices", func(t *testing.T) {
		got := SumAllTails([]int{}, []int{3, 4, 5})
		want := []int{0, 9}
		checkSums(t, got, want)
	})

}
```

كان يمكننا إنشاء دالة جديدة `checkSums` كما نفعل عادة، لكننا في هذه الحالة نعرض تقنية جديدة: إسناد دالة إلى متغير. قد يبدو الأمر غريبًا، لكنه لا يختلف عن إسناد متغير إلى `string` أو `int`، فالدوال في النهاية قيم أيضًا.

ولم يظهر ذلك هنا، لكن هذه التقنية يمكن أن تكون مفيدة عندما تريد ربط دالة بمتغيرات محلية أخرى في "النطاق" (scope) (أي بين بعض الأقواس `{}`). كما أنها تتيح لك تقليل مساحة واجهتك (API).

فبتعريف هذه الدالة داخل الاختبار، لا يمكن لدوال أخرى في هذه الحزمة استخدامها. وإخفاء المتغيرات والدوال التي لا حاجة إلى تصديرها اعتبار تصميمي مهم.

ومن الآثار الجانبية المفيدة لذلك أن هذا يضيف قليلًا من أمان الأنواع إلى كودنا. فإذا أضاف مطور بالخطأ اختبارًا جديدًا مثل `checkSums(t, got, "dave")` فسيوقفه المترجم في مكانه.

```bash
$ go test
./sum_test.go:52:21: cannot use "dave" (type string) as type []int in argument to checkSums
```

## الخلاصة

لقد غطينا:

* المصفوفات
* الشرائح
  * الطرق المختلفة لإنشائها
  * كيف أن لها سعة _ثابتة_، لكن يمكنك إنشاء شرائح جديدة من القديمة باستخدام `append`
  * كيف تقطّع الشرائح!
* `len` للحصول على طول مصفوفة أو شريحة
* أداة تغطية الاختبارات
* `slices.Equal` ولماذا نحتاجها بدلًا من معاملات المساواة العادية

لقد استخدمنا الشرائح والمصفوفات مع الأعداد الصحيحة، لكنها تعمل مع أي نوع آخر أيضًا، بما في ذلك المصفوفات والشرائح نفسها. فيمكنك تعريف متغير من نوع `[][]string` إذا احتجت.

[ألقِ نظرة على تدوينة Go عن الشرائح][blog-slice] للحصول على نظرة معمقة إليها. وجرّب كتابة اختبارات أكثر لترسيخ ما تتعلمه من قراءتها.

وهناك طريقة أخرى مفيدة للتجربة في Go غير كتابة الاختبارات، وهي ملعب Go (Go playground). يمكنك تجربة معظم الأشياء فيه، ويمكنك مشاركة كودك بسهولة إذا احتجت إلى طرح أسئلة. [لقد أعددت لك ملعب Go فيه شريحة لتجرب عليها.](https://play.golang.org/p/ICCWcRGIO68)

[وهذا مثال](https://play.golang.org/p/bTrRmYfNYCp) على تقطيع مصفوفة وكيف يؤثر تغيير الشريحة في المصفوفة الأصلية؛ بينما "نسخة" من الشريحة لن تؤثر في المصفوفة الأصلية.
[ومثال آخر](https://play.golang.org/p/Poth8JS28sc) على لماذا من الجيد أخذ نسخة من الشريحة بعد تقطيع شريحة كبيرة جدًا.

[for]: iteration.md
[blog-slice]: https://blog.golang.org/go-slices-usage-and-internals
[deepEqual]: https://golang.org/pkg/reflect/#DeepEqual
[slice]: https://golang.org/doc/effective_go.html#slices
