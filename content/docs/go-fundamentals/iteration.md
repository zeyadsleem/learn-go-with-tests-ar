---
title: التكرار
weight: 40
---

# التكرار

**[يمكنك العثور على كل كود هذا الفصل هنا](https://github.com/quii/learn-go-with-tests/tree/main/for)**

لتكرار عمل ما في Go، ستحتاج إلى `for`. فلا توجد في Go كلمات مفتاحية مثل `while` و`do` و`until`، وكل ما يمكنك استخدامه هو `for`. وهذا شيء جيد!

لنكتب اختبارًا لدالة تكرر حرفًا خمس مرات.

لا يوجد جديد حتى الآن، لذا جرّب كتابته بنفسك للتدريب.

## اكتب الاختبار أولًا

```go
package iteration

import "testing"

func TestRepeat(t *testing.T) {
	repeated := Repeat("a")
	expected := "aaaaa"

	if repeated != expected {
		t.Errorf("expected %q but got %q", expected, repeated)
	}
}
```

## جرّب تشغيل الاختبار

`./repeat_test.go:6:14: undefined: Repeat`

## اكتب أصغر قدر من الكود ليعمل الاختبار وافحص مخرجاته الفاشلة

_حافظ على الانضباط!_ لا تحتاج إلى معرفة أي شيء جديد الآن ليفشل الاختبار بالشكل الصحيح.

كل ما تحتاجه الآن هو ما يجعل الكود يُترجم، حتى تتأكد أن اختبارك مكتوب جيدًا.

```go
package iteration

func Repeat(character string) string {
	return ""
}
```

أليس من الجميل أن تعرف أنك تعرف من Go ما يكفي لكتابة اختبارات لبعض المشكلات الأساسية؟ وهذا يعني أنك تستطيع الآن العبث بكود الإنتاج كما تشاء وأنت مطمئن أنه يتصرف كما تأمل.

`repeat_test.go:10: expected 'aaaaa' but got ''`

## اكتب كودًا كافيًا لنجاح الاختبار

صياغة `for` عادية جدًا وتتبع معظم اللغات الشبيهة بـ C.

```go
func Repeat(character string) string {
	var repeated string
	for i := 0; i < 5; i++ {
		repeated = repeated + character
	}
	return repeated
}
```

وعلى عكس لغات أخرى مثل C وJava وJavaScript، لا توجد أقواس تحيط بالمكونات الثلاثة لجملة for، بينما الأقواس المعقوفة `{ }` مطلوبة دائمًا. وربما تتساءل عما يحدث في السطر:

```go
	var repeated string
```

فقد كنا نستخدم `:=` حتى الآن لتعريف المتغيرات وتهيئتها. لكن `:=` مجرد [اختصار للخطوتين معًا](https://gobyexample.com/variables). وهنا نعرّف متغيرًا من نوع `string` فقط، ومن هنا جاءت الصيغة الصريحة. ويمكننا أيضًا استخدام `var` لتعريف الدوال، كما سنرى لاحقًا.

شغّل الاختبار، وينبغي أن ينجح.

وهناك صيغ أخرى لحلقة for موصوفة [هنا](https://gobyexample.com/for).

## إعادة الهيكلة

حان وقت إعادة الهيكلة وتقديم تركيب آخر هو معامل الإسناد `+=`.

```go
const repeatCount = 5

func Repeat(character string) string {
	var repeated string
	for i := 0; i < repeatCount; i++ {
		repeated += character
	}
	return repeated
}
```

يُسمى `+=` _"معامل الجمع والإسناد"_، وهو يضيف المعامل الأيمن إلى المعامل الأيسر ثم يُسند النتيجة إلى المعامل الأيسر. وهو يعمل مع أنواع أخرى مثل الأعداد الصحيحة.

### قياس الأداء (Benchmarking)

كتابة [قياسات الأداء](https://golang.org/pkg/testing/#hdr-Benchmarks) في Go ميزة أخرى من الدرجة الأولى في اللغة، وهي شبيهة جدًا بكتابة الاختبارات.

```go
func BenchmarkRepeat(b *testing.B) {
	for b.Loop() {
		Repeat("a")
	}
}
```

سترى أن الكود شبيه جدًا بالاختبار.

ويمنحك `testing.B` إمكانية الوصول إلى دالة الحلقة. وتُرجع `Loop()` القيمة true طالما ينبغي أن يستمر قياس الأداء في العمل.

وعند تنفيذ كود القياس، يُقاس الوقت الذي يستغرقه. وبعد أن تُرجع `Loop()` القيمة false، يحتوي `b.N` على إجمالي عدد التكرارات التي نُفّذت.

ولا ينبغي أن يهمّك عدد مرات تشغيل الكود، فالإطار هو الذي سيحدد ما هي القيمة "الجيدة" لذلك ليعطيك نتائج معقولة.

ولتشغيل قياسات الأداء نفّذ `go test -bench=.` (أو على Windows Powershell نفّذ `go test -bench="."`).

```text
goos: darwin
goarch: amd64
pkg: github.com/quii/learn-go-with-tests/for/v4
10000000           136 ns/op
PASS
```

وما يعنيه `136 ns/op` هو أن دالتنا تستغرق في المتوسط 136 نانوثانية لتعمل (على حاسوبي). وهذا جيد جدًا! ولاختبار ذلك شُغّلت 10000000 مرة.

**ملاحظة:** تُشغَّل قياسات الأداء افتراضيًا بشكل تسلسلي.

ويُقاس وقت جسم الحلقة فقط؛ إذ يستبعد الإطار تلقائيًا كود الإعداد والتنظيف من قياس الوقت. والقياس النموذجي يُبنى هكذا:

```go
func Benchmark(b *testing.B) {
	//... setup ...
	for b.Loop() {
		//... code to measure ...
	}
	//... cleanup ...
}
```

النصوص في Go غير قابلة للتغيير (immutable)، ما يعني أن كل عملية دمج، كما في دالتنا `Repeat`، تتضمن نسخًا للذاكرة لاستيعاب النص الجديد. وهذا يؤثر على الأداء، خصوصًا عند دمج النصوص بكثافة.

وتوفر المكتبة القياسية النوع `strings.Builder`[stringsBuilder] الذي يقلل نسخ الذاكرة.
وهو يوفّر دالة `WriteString` يمكننا استخدامها لدمج النصوص:

```go
const repeatCount = 5

func Repeat(character string) string {
	var repeated strings.Builder
	for i := 0; i < repeatCount; i++ {
		repeated.WriteString(character)
	}
	return repeated.String()
}
```

**ملاحظة**: علينا استدعاء دالة `String` لاسترجاع النتيجة النهائية.

ويمكننا استخدام `BenchmarkRepeat` للتأكد أن `strings.Builder` يحسّن الأداء بشكل ملحوظ.
شغّل `go test -bench=. -benchmem`:

```text
goos: darwin
goarch: amd64
pkg: github.com/quii/learn-go-with-tests/for/v4
10000000           25.70 ns/op           8 B/op           1 allocs/op
PASS
```

ويعرض معامل `-benchmem` معلومات عن عمليات تخصيص الذاكرة:

* `B/op`: عدد البايتات المخصصة في كل تكرار
* `allocs/op`: عدد عمليات تخصيص الذاكرة في كل تكرار

## تمارين تدريبية

* عدّل الاختبار ليتمكن المستدعي من تحديد عدد مرات تكرار الحرف، ثم أصلح الكود
* اكتب `ExampleRepeat` لتوثيق دالتك
* ألقِ نظرة على حزمة [strings](https://golang.org/pkg/strings). ابحث عن دوال ترى أنها قد تكون مفيدة وجرّبها بكتابة اختبارات مثل ما فعلنا هنا. والاستثمار في تعلم المكتبة القياسية سيعود عليك كثيرًا مع الوقت.

## الخلاصة

* مزيد من التدريب على التطوير الموجه بالاختبار
* تعلمنا `for`
* تعلمنا كيفية كتابة قياسات الأداء

[stringsBuilder]: https://pkg.go.dev/strings#Builder
