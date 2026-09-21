---
title: حقن الاعتماديات (Dependency Injection)
weight: 90
---

# حقن الاعتماديات (Dependency Injection)

**[يمكنك العثور على كل كود هذا الفصل هنا](https://github.com/quii/learn-go-with-tests/tree/main/di)**

من المفترض أنك قرأت [فصل الـ structs](structs-methods-and-interfaces.md) من قبل، لأننا سنحتاج إلى بعض الفهم للـ interfaces هنا.

هناك _الكثير_ من سوء الفهم حول حقن الاعتماديات في أوساط البرمجة. آمل أن يوضح لك هذا الدليل أنّ:

* لا تحتاج إلى إطار عمل (framework)
* وأنه لا يزيد تصميمك تعقيدًا
* وأنه يسهّل الاختبار
* وأنه يتيح لك كتابة دوال عامة رائعة.

نريد كتابة دالة تسلّم على شخص، تمامًا كما فعلنا في فصل hello-world، لكننا هذه المرة سنختبر _الطباعة الفعلية_.

وللتذكير، هكذا يمكن أن تبدو تلك الدالة

```go
func Greet(name string) {
	fmt.Printf("Hello, %s", name)
}
```

لكن كيف يمكننا اختبار هذا؟ استدعاء `fmt.Printf` يطبع إلى stdout، وهو أمر صعب علينا التقاطه باستخدام إطار الاختبار.

ما نحتاج إلى فعله هو أن نكون قادرين على **حقن** (inject) اعتمادية الطباعة (وهي مجرد كلمة فاخرة تعني تمرير).

**لا تحتاج دالتنا إلى الاهتمام _بأين_ تحدث الطباعة ولا _كيف_ تحدث، لذا ينبغي أن تقبل _واجهة_ (interface) بدلًا من نوع ملموس.**

وإذا فعلنا ذلك، يمكننا حينها تغيير التنفيذ ليطبع إلى شيء نتحكم فيه حتى نتمكن من اختباره. وفي "الحياة الواقعية" ستحقن شيئًا يكتب إلى stdout.

وإذا نظرت إلى الكود المصدري لـ [`fmt.Printf`](https://pkg.go.dev/fmt#Printf) فسترى طريقة يمكننا من خلالها التوصيل به

```go
// It returns the number of bytes written and any write error encountered.
func Printf(format string, a ...interface{}) (n int, err error) {
	return Fprintf(os.Stdout, format, a...)
}
```

مثير للاهتمام! في العمق، لا يفعل `Printf` سوى استدعاء `Fprintf` وتمرير `os.Stdout` إليه.

وما هي `os.Stdout` بالضبط؟ وما الذي يتوقّع `Fprintf` تمريره إليه في الوسيط الأول؟

```go
func Fprintf(w io.Writer, format string, a ...interface{}) (n int, err error) {
	p := newPrinter()
	p.doPrintf(format, a)
	n, err = w.Write(p.buf)
	p.free()
	return
}
```

إنها `io.Writer`

```go
type Writer interface {
	Write(p []byte) (n int, err error)
}
```

من هذا نستنتج أن `os.Stdout` يحقق `io.Writer`؛ فـ `Printf` يمرر `os.Stdout` إلى `Fprintf` الذي يتوقع `io.Writer`.

وكلما كتبت كود Go أكثر، ستجد هذه الواجهة تظهر كثيرًا، لأنها واجهة عامة رائعة تعني "ضع هذه البيانات في مكان ما".

إذن نعلم أننا في النهاية نستخدم `Writer` لإرسال تحيتنا إلى مكان ما. فلنستخدم هذا التجريد الموجود لجعل كودنا قابلًا للاختبار وأكثر قابلية لإعادة الاستخدام.

## اكتب الاختبار أولًا

```go
func TestGreet(t *testing.T) {
	buffer := bytes.Buffer{}
	Greet(&buffer, "Chris")

	got := buffer.String()
	want := "Hello, Chris"

	if got != want {
		t.Errorf("got %q want %q", got, want)
	}
}
```

نوع `Buffer` من حزمة `bytes` يحقق واجهة `Writer`، لأنه يملك الـ method ‏`Write(p []byte) (n int, err error)`.

لذا سنستخدمه في اختبارنا لنمرّره بصفته `Writer`، ثم يمكننا فحص ما كُتب إليه بعد استدعاء `Greet`.

## جرّب تشغيل الاختبار

لن يُترجم الاختبار

```text
./di_test.go:10:2: undefined: Greet
```

## اكتب أصغر قدر من الكود ليعمل الاختبار وافحص مخرجاته الفاشلة

_أصغِ إلى المترجم_ وأصلح المشكلة.

```go
func Greet(writer *bytes.Buffer, name string) {
	fmt.Printf("Hello, %s", name)
}
```

`Hello, Chris di_test.go:16: got '' want 'Hello, Chris'`

يفشل الاختبار. لاحظ أن الاسم يُطبع، لكنه يذهب إلى stdout.

## اكتب كودًا كافيًا لنجاح الاختبار

استخدم الـ writer لإرسال التحية إلى الـ buffer في اختبارنا. تذكّر أن `fmt.Fprintf` مثل `fmt.Printf`، لكنه يأخذ `Writer` لإرسال النص إليه، بينما `fmt.Printf` يطبع افتراضيًا إلى stdout.

```go
func Greet(writer *bytes.Buffer, name string) {
	fmt.Fprintf(writer, "Hello, %s", name)
}
```

ينجح الاختبار الآن.

## إعادة الهيكلة

قال لنا المترجم سابقًا أن نمرّر مؤشرًا (pointer) إلى `bytes.Buffer`. هذا صحيح تقنيًا لكنه ليس مفيدًا كثيرًا.

ولتوضيح ذلك، جرّب توصيل دالة `Greet` بتطبيق Go نريد فيه أن تطبع إلى stdout.

```go
func main() {
	Greet(os.Stdout, "Elodie")
}
```

`./di.go:14:7: cannot use os.Stdout (type *os.File) as type *bytes.Buffer in argument to Greet`

كما ناقشنا سابقًا، يتيح لك `fmt.Fprintf` تمرير `io.Writer`، ونحن نعلم أن كلاً من `os.Stdout` و`bytes.Buffer` يحققه.

وإذا غيّرنا كودنا ليستخدم الواجهة الأكثر عمومية، يمكننا الآن استخدامه في الاختبارات وفي تطبيقنا معًا.

```go
package main

import (
	"fmt"
	"io"
	"os"
)

func Greet(writer io.Writer, name string) {
	fmt.Fprintf(writer, "Hello, %s", name)
}

func main() {
	Greet(os.Stdout, "Elodie")
}
```

## المزيد عن io.Writer

إلى أي أماكن أخرى يمكننا كتابة البيانات باستخدام `io.Writer`؟ وكم هي عامة دالتنا `Greet`؟

### الإنترنت

شغّل ما يلي

```go
package main

import (
	"fmt"
	"io"
	"log"
	"net/http"
)

func Greet(writer io.Writer, name string) {
	fmt.Fprintf(writer, "Hello, %s", name)
}

func MyGreeterHandler(w http.ResponseWriter, r *http.Request) {
	Greet(w, "world")
}

func main() {
	log.Fatal(http.ListenAndServe(":5001", http.HandlerFunc(MyGreeterHandler)))
}
```

شغّل البرنامج واذهب إلى [http://localhost:5001](http://localhost:5001). سترى دالة التحية لديك وهي مستخدمة.

سنغطي خوادم HTTP في فصل لاحق، فلا تقلق كثيرًا بشأن التفاصيل.

عندما تكتب معالج HTTP، يُعطى لك `http.ResponseWriter` و`http.Request` الذي استُخدم لإنشاء الطلب. وعندما تنفّذ خادمك، _تكتب_ استجابتك باستخدام الـ writer.

ويمكنك على الأرجح أن تخمّن أن `http.ResponseWriter` يحقق أيضًا `io.Writer`، ولهذا استطعنا إعادة استخدام دالة `Greet` داخل المعالج.

## الخلاصة

لم تكن الجولة الأولى من كودنا سهلة الاختبار لأنها كانت تكتب البيانات إلى مكان لا نتحكم فيه.

_وبدفع من اختباراتنا_ أعدنا هيكلة الكود حتى نتمكن من التحكم في _المكان_ الذي تُكتب فيه البيانات عبر **حقن اعتمادية**، ما أتاح لنا:

* **اختبار كودنا** إذا لم تستطع اختبار دالة _بسهولة_، فالسبب عادةً اعتماديات مثبتة تثبيتًا صلبًا داخل الدالة _أو_ حالة عامة (global state). فإذا كان لديك مثلًا تجمّع اتصالات قاعدة بيانات عام تستخدمه طبقة خدمة ما، فسيكون اختباره صعبًا على الأرجح، وستكون تشغيلاته بطيئة. وسيحفزك حقن الاعتماديات على حقن اعتمادية قاعدة بيانات (عبر واجهة) يمكنك بعدها محاكاتها (mock) بشيء تتحكم فيه في اختباراتك.
* **فصل اهتماماتنا**، بفصل _أين تذهب البيانات_ عن _كيفية توليدها_. فإذا شعرت يومًا أن method أو دالة تتحمل مسؤوليات كثيرة (توليد البيانات _و_ الكتابة إلى قاعدة بيانات؟ معالجة طلبات HTTP _و_ تنفيذ منطق المجال؟) فحقن الاعتماديات هو الأداة التي تحتاجها على الأرجح.
* **إتاحة إعادة استخدام كودنا في سياقات مختلفة** أول سياق "جديد" يمكن استخدام كودنا فيه هو داخل الاختبارات. لكن لاحقًا، إذا أراد أحدهم تجربة شيء جديد مع دالتك، يمكنه حقن اعتمادياته الخاصة.

### ماذا عن الـ mocking؟ سمعت أنك تحتاجه لحقن الاعتماديات وأنه شر أيضًا

سنغطي الـ mocking بالتفصيل لاحقًا (وهو ليس شرًا). تستخدم الـ mocking لاستبدال الأشياء الحقيقية التي تحقنها بنسخة وهمية يمكنك التحكم فيها وفحصها في اختباراتك. لكن في حالتنا هذه، كانت المكتبة القياسية قد أعدّت لنا شيئًا لاستخدامه.

### مكتبة Go القياسية جيدة حقًا، خذ وقتك في دراستها

بمعرفتنا بشيء من واجهة `io.Writer`، استطعنا استخدام `bytes.Buffer` في اختبارنا بصفته `Writer`، ثم يمكننا استخدام `Writer`s أخرى من المكتبة القياسية لاستخدام دالتنا في تطبيق سطر أوامر أو في خادم ويب.

وكلما زادت معرفتك بالمكتبة القياسية، زاد ظهور هذه الواجهات العامة لك، لتستطيع إعادة استخدامها في كودك وتجعل برمجياتك قابلة لإعادة الاستخدام في عدد من السياقات.

هذا المثال متأثر بشدة بفصل من [The Go Programming Language](https://www.amazon.co.uk/Programming-Language-Addison-Wesley-Professional-Computing/dp/0134190440)، فإن استمتعت به، اذهب واشترِه!
