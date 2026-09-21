---
title: الـ Reflection
weight: 130
---

# الـ Reflection

**[يمكنك العثور على كل كود هذا الفصل هنا](https://github.com/quii/learn-go-with-tests/tree/main/reflection)**

[من X](https://x.com/peterbourgon/status/1011403901419937792?s=09)

> تحدي golang: اكتب دالة `walk(x interface{}, fn func(string))` تأخذ struct اسمه `x` وتستدعي `fn` على كل حقول النصوص الموجودة بداخله. مستوى الصعوبة: العودية.

لفعل ذلك سنحتاج إلى استخدام _الـ Reflection_.

> الـ Reflection في الحوسبة هو قدرة برنامج على فحص بنيته الخاصة، ولا سيما عبر الأنواع؛ وهو شكل من أشكال البرمجة الفوقية (metaprogramming). وهو أيضًا مصدر كبير للحيرة.

من [مدونة Go: الـ Reflection](https://blog.golang.org/laws-of-reflection)

## ما هو `interface{}`؟

لقد استمتعنا بأمان الأنواع (type-safety) الذي تقدمه لنا Go في الدوال التي تعمل مع أنواع معروفة، مثل `string` و`int` وأنواعنا الخاصة مثل `BankAccount`.

وهذا يعني أننا نحصل على بعض التوثيق مجانًا، وسيشتكي المترجم إن حاولت تمرير نوع خاطئ إلى دالة.

لكن قد تصادف حالات تريد فيها كتابة دالة لا تعرف نوعها وقت الترجمة.

تتيح لنا Go تجاوز ذلك بالنوع `interface{}`، ويمكنك التفكير فيه ببساطة على أنه _أي_ نوع (بل إن `any` في Go مجرد [اسم بديل](https://cs.opensource.google/go/go/+/master:src/builtin/builtin.go;drc=master;l=95) لـ `interface{}`).

لذا ستقبل `walk(x interface{}, fn func(string))` أي قيمة لـ `x`.

### إذن لماذا لا نستخدم `interface{}` في كل شيء ونكتب دوالًا مرنة حقًا؟

- بصفتك مستخدمًا لدالة تأخذ `interface{}` فأنت تفقد أمان الأنواع. فماذا لو كنت تقصد تمرير `Herd.species` من النوع `string` إلى دالة، لكنك مرّرت بدلًا منه `Herd.count` وهو `int`؟ لن يستطيع المترجم تنبيهك إلى خطئك. كما لن تكون لديك أي فكرة عن _ما_ يُسمح لك بتمريره إلى الدالة. فمعرفة أن دالة تأخذ `UserService` مثلًا مفيدة جدًا.
- وبصفتك كاتبًا لدالة كهذه، يجب أن تكون قادرًا على فحص _أي شيء_ يُمرَّر إليك ومحاولة معرفة نوعه وما يمكنك فعله به. ويتم ذلك باستخدام _الـ Reflection_. وهذا قد يكون أخرق ويصعب قراءته، ويكون عمومًا أقل أداءً (لأنك مضطر إلى إجراء فحوصات وقت التشغيل).

باختصار، لا تستخدم الـ reflection إلا إذا كنت حقًا بحاجة إليه.

وإذا أردت دوال متعددة الأشكال (polymorphic functions)، ففكّر إن كان يمكنك تصميمها حول واجهة (interface) (وليست `interface{}`، وهذا مربك بالفعل) بحيث يستطيع المستخدمون استخدام دالتك مع أنواع متعددة إذا نفّذوا الـ methods التي تحتاجها دالتك لتعمل.

ستحتاج دالتنا إلى العمل مع أشياء كثيرة مختلفة. وكما جرت العادة سنتبع أسلوبًا تكراريًا، نكتب اختبارًا لكل شيء جديد نريد دعمه ونعيد الهيكلة على طول الطريق حتى ننتهي.

## اكتب الاختبار أولًا

سنريد استدعاء دالتنا بـ struct يحتوي حقلًا نصيًا (`x`). ثم يمكننا التجسس على الدالة (`fn`) المُمرَّرة لنرى هل استُدعيت أم لا.

```go
func TestWalk(t *testing.T) {

	expected := "Chris"
	var got []string

	x := struct {
		Name string
	}{expected}

	walk(x, func(input string) {
		got = append(got, input)
	})

	if len(got) != 1 {
		t.Errorf("wrong number of function calls, got %d want %d", len(got), 1)
	}
}
```

- نريد تخزين شريحة من النصوص (`got`) تحفظ النصوص التي مرّرها `walk` إلى `fn`. غالبًا في الفصول السابقة أنشأنا أنواعًا مخصصة لهذا لتتجسس على استدعاءات الدوال والـ methods، لكن في هذه الحالة يمكننا ببساطة تمرير دالة مجهولة إلى `fn` تغلق على `got`.
- نستخدم `struct` مجهولًا يحتوي حقل `Name` من نوع string لنسلك أبسط مسار "سعيد".
- وأخيرًا نستدعي `walk` بـ `x` والـ spy، ونكتفي الآن بفحص طول `got`؛ وسنكون أكثر تحديدًا في التحقق (assertion) حين يصبح لدينا شيء أساسي يعمل.

## جرّب تشغيل الاختبار

```
./reflection_test.go:21:2: undefined: walk
```

## اكتب أصغر قدر من الكود ليعمل الاختبار وافحص مخرجاته الفاشلة

نحتاج إلى تعريف `walk`

```go
func walk(x interface{}, fn func(input string)) {

}
```

جرّب تشغيل الاختبار مرة أخرى

```
=== RUN   TestWalk
--- FAIL: TestWalk (0.00s)
    reflection_test.go:19: wrong number of function calls, got 0 want 1
FAIL
```

## اكتب كودًا كافيًا لنجاح الاختبار

يمكننا استدعاء الـ spy بأي نص لنجعل الاختبار ينجح.

```go
func walk(x interface{}, fn func(input string)) {
	fn("I still can't believe South Korea beat Germany 2-0 to put them last in their group")
}
```

من المفترض أن ينجح الاختبار الآن. والشيء التالي الذي سنحتاج إليه هو تحقق أكثر تحديدًا لما يُستدعى به `fn`.

## اكتب الاختبار أولًا

أضف ما يلي إلى الاختبار الحالي للتأكد أن النص المُمرَّر إلى `fn` صحيح

```go
if got[0] != expected {
	t.Errorf("got %q, want %q", got[0], expected)
}
```

## جرّب تشغيل الاختبار

```
=== RUN   TestWalk
--- FAIL: TestWalk (0.00s)
    reflection_test.go:23: got 'I still can't believe South Korea beat Germany 2-0 to put them last in their group', want 'Chris'
FAIL
```

## اكتب كودًا كافيًا لنجاح الاختبار

```go
func walk(x interface{}, fn func(input string)) {
	val := reflect.ValueOf(x)
	field := val.Field(0)
	fn(field.String())
}
```

هذا الكود _غير آمن إطلاقًا وساذج جدًا_، لكن تذكّر: هدفنا عندما نكون في الحالة "الحمراء" (فشل الاختبارات) هو كتابة أصغر قدر ممكن من الكود. ثم نكتب المزيد من الاختبارات لمعالجة مخاوفنا.

نحتاج إلى استخدام الـ reflection لإلقاء نظرة على `x` ومحاولة فحص خصائصه.

تحتوي [حزمة reflect](https://pkg.go.dev/reflect) على دالة `ValueOf` تُرجع لنا `Value` لمتغير معطى. ولها طرق تتيح لنا فحص قيمة، بما في ذلك حقولها التي نستخدمها في السطر التالي.

ثم نضع بعض الافتراضات المتفائلة جدًا حول القيمة المُمرَّرة:

- ننظر إلى الحقل الأول والوحيد. لكن قد لا توجد أي حقول على الإطلاق، وهذا سيؤدي إلى panic.
- ثم نستدعي `String()`، وهي تُرجع القيمة الأساسية كنص. لكن ذلك سيكون خاطئًا لو كان الحقل شيئًا غير النص.

## إعادة الهيكلة

كودنا ينجح في الحالة البسيطة، لكننا نعلم أن فيه الكثير من أوجه القصور.

سنكتب عددًا من الاختبارات نمرّر فيها قيمًا مختلفة ونتحقق من مصفوفة النصوص التي استُدعي بها `fn`.

ينبغي أن نعيد هيكلة اختبارنا إلى اختبار قائم على الجدول (table based test) ليكون متابعة اختبار سيناريوهات جديدة أسهل.

```go
func TestWalk(t *testing.T) {

	cases := []struct {
		Name          string
		Input         interface{}
		ExpectedCalls []string
	}{
		{
			"struct with one string field",
			struct {
				Name string
			}{"Chris"},
			[]string{"Chris"},
		},
	}

	for _, test := range cases {
		t.Run(test.Name, func(t *testing.T) {
			var got []string
			walk(test.Input, func(input string) {
				got = append(got, input)
			})

			if !reflect.DeepEqual(got, test.ExpectedCalls) {
				t.Errorf("got %v, want %v", got, test.ExpectedCalls)
			}
		})
	}
}
```

الآن يمكننا بسهولة إضافة سيناريو لنرى ماذا يحدث إذا كان لدينا أكثر من حقل نصي.

## اكتب الاختبار أولًا

أضف السيناريو التالي إلى `cases`.

```
{
    "struct with two string fields",
    struct {
        Name string
        City string
    }{"Chris", "London"},
    []string{"Chris", "London"},
},
```

## جرّب تشغيل الاختبار

```
=== RUN   TestWalk/struct_with_two_string_fields
    --- FAIL: TestWalk/struct_with_two_string_fields (0.00s)
        reflection_test.go:40: got [Chris], want [Chris London]
```

## اكتب كودًا كافيًا لنجاح الاختبار

```go
func walk(x interface{}, fn func(input string)) {
	val := reflect.ValueOf(x)

	for i := 0; i < val.NumField(); i++ {
		field := val.Field(i)
		fn(field.String())
	}
}
```

لـ `val` method اسمه `NumField` يُرجع عدد الحقول في القيمة. وهذا يتيح لنا المرور على الحقول واستدعاء `fn`، فينجح اختبارنا.

## إعادة الهيكلة

لا يبدو أن هناك أي إعادة هيكلة واضحة هنا ستحسّن الكود، فلنمضِ قدمًا.

القصور التالي في `walk` هو أنه يفترض أن كل حقل `string`. لنكتب اختبارًا لهذا السيناريو.

## اكتب الاختبار أولًا

أضف الحالة التالية

```
{
    "struct with non string field",
    struct {
        Name string
        Age  int
    }{"Chris", 33},
    []string{"Chris"},
},
```

## جرّب تشغيل الاختبار

```
=== RUN   TestWalk/struct_with_non_string_field
    --- FAIL: TestWalk/struct_with_non_string_field (0.00s)
        reflection_test.go:46: got [Chris <int Value>], want [Chris]
```

## اكتب كودًا كافيًا لنجاح الاختبار

نحتاج إلى التحقق أن نوع الحقل هو `string`.

```go
func walk(x interface{}, fn func(input string)) {
	val := reflect.ValueOf(x)

	for i := 0; i < val.NumField(); i++ {
		field := val.Field(i)

		if field.Kind() == reflect.String {
			fn(field.String())
		}
	}
}
```

يمكننا فعل ذلك بفحص [`Kind`](https://pkg.go.dev/reflect#Kind) الخاص به.

## إعادة الهيكلة

مرة أخرى يبدو الكود معقولًا بما يكفي في الوقت الحالي.

السيناريو التالي هو: ماذا لو لم يكن الـ `struct` "مسطحًا"؟ بعبارة أخرى، ماذا يحدث إذا كان لدينا `struct` فيه بعض الحقول المتداخلة؟

## اكتب الاختبار أولًا

لقد كنا نستخدم صياغة الـ struct المجهول لتعريف أنواع بشكل مؤقت في اختباراتنا، لذا يمكننا الاستمرار في ذلك هكذا

```
{
    "nested fields",
    struct {
        Name string
        Profile struct {
            Age  int
            City string
        }
    }{"Chris", struct {
        Age  int
        City string
    }{33, "London"}},
    []string{"Chris", "London"},
},
```

لكننا نرى أن الصياغة تصبح فوضوية قليلًا مع الـ structs المجهولة الداخلية. و[هناك اقتراح لجعل الصياغة أجمل](https://github.com/golang/go/issues/12854).

لنعد هيكلة هذا ببساطة عبر إنشاء نوع معروف لهذا السيناريو والإشارة إليه في الاختبار. وفيه غير مباشرية صغيرة، إذ إن بعض كود اختبارنا خارج الاختبار، لكن ينبغي أن يستطيع القراء استنتاج بنية الـ `struct` بالنظر إلى تهيئته.

أضف تعريفات الأنواع التالية في مكان ما من ملف اختبارك

```go
type Person struct {
	Name    string
	Profile Profile
}

type Profile struct {
	Age  int
	City string
}
```

الآن يمكننا إضافة هذا إلى حالاتنا، وهو أكثر وضوحًا بكثير من قبل

```
{
    "nested fields",
    Person{
        "Chris",
        Profile{33, "London"},
    },
    []string{"Chris", "London"},
},
```

## جرّب تشغيل الاختبار

```
=== RUN   TestWalk/Nested_fields
    --- FAIL: TestWalk/nested_fields (0.00s)
        reflection_test.go:54: got [Chris], want [Chris London]
```

المشكلة أننا نمر فقط على الحقول في المستوى الأول من تسلسل النوع.

## اكتب كودًا كافيًا لنجاح الاختبار

```go
func walk(x interface{}, fn func(input string)) {
	val := reflect.ValueOf(x)

	for i := 0; i < val.NumField(); i++ {
		field := val.Field(i)

		if field.Kind() == reflect.String {
			fn(field.String())
		}

		if field.Kind() == reflect.Struct {
			walk(field.Interface(), fn)
		}
	}
}
```

الحل بسيط جدًا؛ نفحص `Kind` مرة أخرى، وإن صادف أنه `struct` نستدعي `walk` مرة أخرى على ذلك الـ `struct` الداخلي.

## إعادة الهيكلة

```go
func walk(x interface{}, fn func(input string)) {
	val := reflect.ValueOf(x)

	for i := 0; i < val.NumField(); i++ {
		field := val.Field(i)

		switch field.Kind() {
		case reflect.String:
			fn(field.String())
		case reflect.Struct:
			walk(field.Interface(), fn)
		}
	}
}
```

عندما تجري مقارنة على القيمة نفسها أكثر من مرة، فـ _عمومًا_ ستؤدي إعادة الهيكلة إلى `switch` إلى تحسين القراءة وجعل كودك أسهل في التوسعة.

ماذا لو كانت قيمة الـ struct المُمرَّر مؤشرًا (pointer)؟

## اكتب الاختبار أولًا

أضف هذه الحالة

```
{
    "pointers to things",
    &Person{
        "Chris",
        Profile{33, "London"},
    },
    []string{"Chris", "London"},
},
```

## جرّب تشغيل الاختبار

```
=== RUN   TestWalk/pointers_to_things
panic: reflect: call of reflect.Value.NumField on ptr Value [recovered]
    panic: reflect: call of reflect.Value.NumField on ptr Value
```

## اكتب كودًا كافيًا لنجاح الاختبار

```go
func walk(x interface{}, fn func(input string)) {
	val := reflect.ValueOf(x)

	if val.Kind() == reflect.Pointer {
		val = val.Elem()
	}

	for i := 0; i < val.NumField(); i++ {
		field := val.Field(i)

		switch field.Kind() {
		case reflect.String:
			fn(field.String())
		case reflect.Struct:
			walk(field.Interface(), fn)
		}
	}
}
```

لا يمكنك استخدام `NumField` على `Value` من نوع مؤشر؛ نحتاج إلى استخراج القيمة الأساسية قبل ذلك باستخدام `Elem()`.

## إعادة الهيكلة

لنُغلّف مسؤولية استخراج `reflect.Value` من `interface{}` معطى داخل دالة.

```go
func walk(x interface{}, fn func(input string)) {
	val := getValue(x)

	for i := 0; i < val.NumField(); i++ {
		field := val.Field(i)

		switch field.Kind() {
		case reflect.String:
			fn(field.String())
		case reflect.Struct:
			walk(field.Interface(), fn)
		}
	}
}

func getValue(x interface{}) reflect.Value {
	val := reflect.ValueOf(x)

	if val.Kind() == reflect.Pointer {
		val = val.Elem()
	}

	return val
}
```

هذا يضيف في الواقع كودًا _أكثر_، لكنني أشعر أن مستوى التجريد مناسب.

- الحصول على `reflect.Value` الخاص بـ `x` حتى أستطيع فحصه، ولا يهمني كيف.
- المرور على الحقول وفعل ما يجب فعله حسب نوعها.

بعد ذلك، نحتاج إلى تغطية الشرائح (slices).

## اكتب الاختبار أولًا

```
{
    "slices",
    []Profile {
        {33, "London"},
        {34, "Reykjavík"},
    },
    []string{"London", "Reykjavík"},
},
```

## جرّب تشغيل الاختبار

```
=== RUN   TestWalk/slices
panic: reflect: call of reflect.Value.NumField on slice Value [recovered]
    panic: reflect: call of reflect.Value.NumField on slice Value
```

## اكتب أصغر قدر من الكود ليعمل الاختبار وافحص مخرجاته الفاشلة

هذا مشابه لسيناريو المؤشر السابق؛ نحاول استدعاء `NumField` على `reflect.Value` لكنه لا يملكها لأنه ليس struct.

## اكتب كودًا كافيًا لنجاح الاختبار

```go
func walk(x interface{}, fn func(input string)) {
	val := getValue(x)

	if val.Kind() == reflect.Slice {
		for i := 0; i < val.Len(); i++ {
			walk(val.Index(i).Interface(), fn)
		}
		return
	}

	for i := 0; i < val.NumField(); i++ {
		field := val.Field(i)

		switch field.Kind() {
		case reflect.String:
			fn(field.String())
		case reflect.Struct:
			walk(field.Interface(), fn)
		}
	}
}
```

## إعادة الهيكلة

هذا يعمل لكنه بشع. لا تقلق، لدينا كود يعمل ومدعوم باختبارات، لذا نحن أحرار في العبث به كما نشاء.

إذا فكرت بتجريد قليل، فنحن نريد استدعاء `walk` على أي من

- كل حقل في struct
- كل _شيء_ في slice

كودنا يفعل ذلك الآن، لكنه لا يعكسه جيدًا. فلدينا فقط فحص في البداية لمعرفة هل هو slice (مع `return` لإيقاف تنفيذ بقية الكود)، وإذا لم يكن كذلك نفترض مباشرة أنه struct.

لنُعد صياغة الكود بحيث نفحص النوع _أولًا_ ثم نقوم بعملنا.

```go
func walk(x interface{}, fn func(input string)) {
	val := getValue(x)

	switch val.Kind() {
	case reflect.Struct:
		for i := 0; i < val.NumField(); i++ {
			walk(val.Field(i).Interface(), fn)
		}
	case reflect.Slice:
		for i := 0; i < val.Len(); i++ {
			walk(val.Index(i).Interface(), fn)
		}
	case reflect.String:
		fn(val.String())
	}
}
```

يبدو أفضل بكثير! إذا كان struct أو slice، نمر على قيمه مستدعين `walk` على كل منها. وإلا، إذا كان `reflect.String` يمكننا استدعاء `fn`.

ومع ذلك يبدو لي أنه يمكن أن يكون أفضل. فهناك تكرار لعملية المرور على الحقول/القيم ثم استدعاء `walk`، لكنهما متماثلان من حيث المفهوم.

```go
func walk(x interface{}, fn func(input string)) {
	val := getValue(x)

	numberOfValues := 0
	var getField func(int) reflect.Value

	switch val.Kind() {
	case reflect.String:
		fn(val.String())
	case reflect.Struct:
		numberOfValues = val.NumField()
		getField = val.Field
	case reflect.Slice:
		numberOfValues = val.Len()
		getField = val.Index
	}

	for i := 0; i < numberOfValues; i++ {
		walk(getField(i).Interface(), fn)
	}
}
```

إذا كانت `value` من نوع `reflect.String` فسنستدعي `fn` كالمعتاد.

وإلا، سيستخرج `switch` لدينا شيئين حسب النوع

- عدد الحقول الموجودة
- كيفية استخراج `Value` (`Field` أو `Index`)

وبعد أن نحدد هذين الأمرين يمكننا المرور على `numberOfValues` مستدعين `walk` بنتيجة دالة `getField`.

وبعد أن فعلنا هذا، ينبغي أن تكون معالجة المصفوفات (arrays) أمرًا بسيطًا.

## اكتب الاختبار أولًا

أضف إلى الحالات

```
{
    "arrays",
    [2]Profile {
        {33, "London"},
        {34, "Reykjavík"},
    },
    []string{"London", "Reykjavík"},
},
```

## جرّب تشغيل الاختبار

```
=== RUN   TestWalk/arrays
    --- FAIL: TestWalk/arrays (0.00s)
        reflection_test.go:78: got [], want [London Reykjavík]
```

## اكتب كودًا كافيًا لنجاح الاختبار

يمكن التعامل مع المصفوفات بالطريقة نفسها التي نتعامل بها مع الشرائح، فأضفها إلى الحالة بفاصلة فقط

```go
func walk(x interface{}, fn func(input string)) {
	val := getValue(x)

	numberOfValues := 0
	var getField func(int) reflect.Value

	switch val.Kind() {
	case reflect.String:
		fn(val.String())
	case reflect.Struct:
		numberOfValues = val.NumField()
		getField = val.Field
	case reflect.Slice, reflect.Array:
		numberOfValues = val.Len()
		getField = val.Index
	}

	for i := 0; i < numberOfValues; i++ {
		walk(getField(i).Interface(), fn)
	}
}
```

النوع التالي الذي نريد معالجته هو `map`.

## اكتب الاختبار أولًا

```
{
    "maps",
    map[string]string{
        "Cow": "Moo",
        "Sheep": "Baa",
    },
    []string{"Moo", "Baa"},
},
```

## جرّب تشغيل الاختبار

```
=== RUN   TestWalk/maps
    --- FAIL: TestWalk/maps (0.00s)
        reflection_test.go:86: got [], want [Moo Baa]
```

## اكتب كودًا كافيًا لنجاح الاختبار

مرة أخرى، إذا فكرت بتجريد قليل فسترى أن `map` شبيهة جدًا بالـ `struct`، إلا أن المفاتيح (keys) غير معروفة وقت الترجمة.

```go
func walk(x interface{}, fn func(input string)) {
	val := getValue(x)

	numberOfValues := 0
	var getField func(int) reflect.Value

	switch val.Kind() {
	case reflect.String:
		fn(val.String())
	case reflect.Struct:
		numberOfValues = val.NumField()
		getField = val.Field
	case reflect.Slice, reflect.Array:
		numberOfValues = val.Len()
		getField = val.Index
	case reflect.Map:
		for _, key := range val.MapKeys() {
			walk(val.MapIndex(key).Interface(), fn)
		}
	}

	for i := 0; i < numberOfValues; i++ {
		walk(getField(i).Interface(), fn)
	}
}
```

لكن، بحكم التصميم، لا يمكنك الحصول على قيم من map عبر الفهرس؛ فذلك يتم فقط بواسطة الـ _key_، إذن هذا يكسر تجريدنا، للأسف.

## إعادة الهيكلة

كيف تشعر الآن؟ ربما بدا تجريدًا جميلًا في حينه، لكن الكود الآن يبدو مرتبكًا قليلًا.

_لا بأس!_ إعادة الهيكلة رحلة، وأحيانًا سنرتكب أخطاء. ومن أهم أهداف التطوير الموجه بالاختبار أنه يمنحنا الحرية لتجربة هذه الأشياء.

وبخطوات صغيرة مدعومة بالاختبارات، لا يكون هذا وضعًا غير قابل للتراجع بأي حال. فلنعده ببساطة إلى ما كان عليه قبل إعادة الهيكلة.

```go
func walk(x interface{}, fn func(input string)) {
	val := getValue(x)

	walkValue := func(value reflect.Value) {
		walk(value.Interface(), fn)
	}

	switch val.Kind() {
	case reflect.String:
		fn(val.String())
	case reflect.Struct:
		for i := 0; i < val.NumField(); i++ {
			walkValue(val.Field(i))
		}
	case reflect.Slice, reflect.Array:
		for i := 0; i < val.Len(); i++ {
			walkValue(val.Index(i))
		}
	case reflect.Map:
		for _, key := range val.MapKeys() {
			walkValue(val.MapIndex(key))
		}
	}
}
```

لقد قدّمنا `walkValue` لتقليل التكرار في استدعاءات `walk` داخل `switch`، بحيث لا يبقى عليها إلا استخراج قيم `reflect.Value` من `val`.

### مشكلة أخيرة

تذكّر أن الـ maps في Go لا تضمن الترتيب. لذا ستفشل اختباراتك أحيانًا لأننا نتحقق أن استدعاءات `fn` تتم بترتيب معين.

لإصلاح ذلك، سنحتاج إلى نقل التحقق الخاص بالـ maps إلى اختبار جديد لا يهمنا فيه الترتيب.

```go
t.Run("with maps", func(t *testing.T) {
	aMap := map[string]string{
		"Cow":   "Moo",
		"Sheep": "Baa",
	}

	var got []string
	walk(aMap, func(input string) {
		got = append(got, input)
	})

	assertContains(t, got, "Moo")
	assertContains(t, got, "Baa")
})
```

وهكذا تُعرَّف `assertContains`

```go
func assertContains(t testing.TB, haystack []string, needle string) {
	t.Helper()
	contains := false
	for _, x := range haystack {
		if x == needle {
			contains = true
		}
	}
	if !contains {
		t.Errorf("expected %v to contain %q but it didn't", haystack, needle)
	}
}
```

بما أننا نقلنا الـ maps إلى اختبار جديد، فلم نرَ رسالة الفشل. اكسر اختبار `with maps` هنا عن قصد لتفحص رسالة الخطأ، ثم أصلحه من جديد لتعود كل الاختبارات إلى النجاح.

تخلّينا عن فحص _ترتيب_ `got` لأن الـ maps لا تضمن ترتيبًا، لكن هذا لا يعني أننا يجب أن نتخلى عن فحص كل شيء يخص شكله. فالـ `assertContains` وحدها ستظل تنجح لو زار `walk` عنصرًا في map مرتين، أو أغفل عنصرًا وفحص العناصر المتبقية فقط - ما دامت القيم المحددة التي نبحث عنها موجودة في مكان ما، بأي عدد من المرات. وخلافًا للترتيب، فإن _طول_ `got` متوقع تمامًا بغض النظر عن ترتيب المرور على الـ map، فلنتحقق من ذلك أيضًا.

```go
func assertLength(t testing.TB, got []string, want int) {
	t.Helper()
	if len(got) != want {
		t.Errorf("got %d values but expected %d", len(got), want)
	}
}
```

أضف استدعاءً لها في بداية اختبار `with maps`.

```go
t.Run("with maps", func(t *testing.T) {
	aMap := map[string]string{
		"Cow":   "Moo",
		"Sheep": "Baa",
	}

	var got []string
	walk(aMap, func(input string) {
		got = append(got, input)
	})

	assertLength(t, got, len(aMap))
	assertContains(t, got, "Moo")
	assertContains(t, got, "Baa")
})
```

النوع التالي الذي نريد معالجته هو `chan`.

## اكتب الاختبار أولًا

```go
t.Run("with channels", func(t *testing.T) {
	aChannel := make(chan Profile)

	go func() {
		aChannel <- Profile{33, "Berlin"}
		aChannel <- Profile{34, "Katowice"}
		close(aChannel)
	}()

	var got []string
	want := []string{"Berlin", "Katowice"}

	walk(aChannel, func(input string) {
		got = append(got, input)
	})

	if !reflect.DeepEqual(got, want) {
		t.Errorf("got %v, want %v", got, want)
	}
})
```

## جرّب تشغيل الاختبار

```
--- FAIL: TestWalk (0.00s)
    --- FAIL: TestWalk/with_channels (0.00s)
        reflection_test.go:115: got [], want [Berlin Katowice]
```

## اكتب كودًا كافيًا لنجاح الاختبار

يمكننا المرور على كل القيم المُرسَلة عبر channel حتى إغلاقه باستخدام Recv()

```go
func walk(x interface{}, fn func(input string)) {
	val := getValue(x)

	walkValue := func(value reflect.Value) {
		walk(value.Interface(), fn)
	}

	switch val.Kind() {
	case reflect.String:
		fn(val.String())
	case reflect.Struct:
		for i := 0; i < val.NumField(); i++ {
			walkValue(val.Field(i))
		}
	case reflect.Slice, reflect.Array:
		for i := 0; i < val.Len(); i++ {
			walkValue(val.Index(i))
		}
	case reflect.Map:
		for _, key := range val.MapKeys() {
			walkValue(val.MapIndex(key))
		}
	case reflect.Chan:
		for {
			if v, ok := val.Recv(); ok {
				walkValue(v)
			} else {
				break
			}
		}
	}
}
```
النوع التالي الذي نريد معالجته هو `func`.

## اكتب الاختبار أولًا

```go
t.Run("with function", func(t *testing.T) {
	aFunction := func() (Profile, Profile) {
		return Profile{33, "Berlin"}, Profile{34, "Katowice"}
	}

	var got []string
	want := []string{"Berlin", "Katowice"}

	walk(aFunction, func(input string) {
		got = append(got, input)
	})

	if !reflect.DeepEqual(got, want) {
		t.Errorf("got %v, want %v", got, want)
	}
})
```

## جرّب تشغيل الاختبار

```
--- FAIL: TestWalk (0.00s)
    --- FAIL: TestWalk/with_function (0.00s)
        reflection_test.go:132: got [], want [Berlin Katowice]
```

## اكتب كودًا كافيًا لنجاح الاختبار

لا تبدو الدوال التي تأخذ وسائط ذات معنى كبير في هذا السيناريو. لكن ينبغي أن نسمح بقيم إرجاع اعتباطية.

```go
func walk(x interface{}, fn func(input string)) {
	val := getValue(x)

	walkValue := func(value reflect.Value) {
		walk(value.Interface(), fn)
	}

	switch val.Kind() {
	case reflect.String:
		fn(val.String())
	case reflect.Struct:
		for i := 0; i < val.NumField(); i++ {
			walkValue(val.Field(i))
		}
	case reflect.Slice, reflect.Array:
		for i := 0; i < val.Len(); i++ {
			walkValue(val.Index(i))
		}
	case reflect.Map:
		for _, key := range val.MapKeys() {
			walkValue(val.MapIndex(key))
		}
	case reflect.Chan:
		for v, ok := val.Recv(); ok; v, ok = val.Recv() {
			walkValue(v)
		}
	case reflect.Func:
		valFnResult := val.Call(nil)
		for _, res := range valFnResult {
			walkValue(res)
		}
	}
}
```

## الخلاصة

- قدّمنا بعض المفاهيم من حزمة `reflect`.
- استخدمنا العودية (recursion) للمرور على بنى بيانات اعتباطية.
- أجرينا إعادة هيكلة اتضح لاحقًا أنها سيئة، لكننا لم ننزعج منها كثيرًا. فالعمل بشكل تكراري مع الاختبارات يجعل الأمر ليس بهذه الضخامة.
- هذا غطّى جانبًا صغيرًا فقط من الـ reflection. و[لمدونة Go مقال ممتاز يغطي تفاصيل أكثر](https://blog.golang.org/laws-of-reflection).
- والآن بعد أن عرفت الـ reflection، ابذل جهدك لتتجنب استخدامه.

### قيد معروف: المراجع الدائرية

ستحدث لـ `walk` مشكلة تجاوز سعة المكدس (stack-overflow) إذا مرّرت إليه struct يحتوي مؤشرًا يعود إلى نفسه (أو أي حلقة من المؤشرات). على سبيل المثال:

```go
type Person struct {
    Name   string
    Friend *Person
}
p := Person{Name: "Alice"}
p.Friend = &p  // cycle!
walk(p, fn)  // fatal error: stack overflow
```

إصلاح هذا متروك كتمرين. والأسلوب المعتاد هو تتبع عناوين المؤشرات التي زرناها بالفعل وتخطيها عند الدخول إليها من جديد. وستحتاج إلى `map[uintptr]bool` — والـ `uintptr` هو العنوان الرقمي الخام لمؤشر، ويمكن الحصول عليه عبر `reflect.Value.Pointer()` لأنواع المؤشرات. ولأن الـ map يجب أن تبقى طوال عملية المرور، فستريد دالة مساعدة داخلية تحملها كوسيط، بينما تنشئ `walk` العامة الـ map وتستدعيها.
