---
title: التزامن (Concurrency)
weight: 110
---

# التزامن (Concurrency)

**[يمكنك العثور على كل كود هذا الفصل هنا](https://github.com/quii/learn-go-with-tests/tree/main/concurrency)**

إليك البداية: كتب زميل لك دالة اسمها `CheckWebsites` تفحص حالة قائمة من عناوين URL.

```go
package concurrency

type WebsiteChecker func(string) bool

func CheckWebsites(wc WebsiteChecker, urls []string) map[string]bool {
	results := make(map[string]bool)

	for _, url := range urls {
		results[url] = wc(url)
	}

	return results
}
```

تُرجع الدالة خريطة (map) تربط كل عنوان URL تم فحصه بقيمة منطقية (boolean): `true` لاستجابة جيدة، و`false` لاستجابة سيئة.

كما يجب عليك تمرير `WebsiteChecker` تأخذ عنوان URL واحدًا وتُرجع قيمة منطقية، وتستخدمها الدالة لفحص جميع المواقع.

وقد أتاح لهم استخدام [حقن الاعتماديات][DI] اختبار الدالة دون إجراء استدعاءات HTTP حقيقية، فصارت موثوقة وسريعة.

وإليك الاختبار الذي كتبوه:

```go
package concurrency

import (
	"reflect"
	"testing"
)

func mockWebsiteChecker(url string) bool {
	return url != "waat://furhurterwe.geds"
}

func TestCheckWebsites(t *testing.T) {
	websites := []string{
		"http://google.com",
		"http://blog.gypsydave5.com",
		"waat://furhurterwe.geds",
	}

	want := map[string]bool{
		"http://google.com":          true,
		"http://blog.gypsydave5.com": true,
		"waat://furhurterwe.geds":    false,
	}

	got := CheckWebsites(mockWebsiteChecker, websites)

	if !reflect.DeepEqual(want, got) {
		t.Fatalf("wanted %v, got %v", want, got)
	}
}
```

الدالة قيد الاستخدام في الإنتاج لفحص مئات المواقع. لكن زميلك بدأ يستقبل شكاوى أنها بطيئة، فطلب منك المساعدة في تسريعها.

## اكتب اختبارًا

لنستخدم قياس أداء (benchmark) لاختبار سرعة `CheckWebsites` حتى نرى أثر تغييراتنا.

```go
package concurrency

import (
	"testing"
	"time"
)

func slowStubWebsiteChecker(_ string) bool {
	time.Sleep(20 * time.Millisecond)
	return true
}

func BenchmarkCheckWebsites(b *testing.B) {
	urls := make([]string, 100)
	for i := 0; i < len(urls); i++ {
		urls[i] = "a url"
	}

	for b.Loop() {
		CheckWebsites(slowStubWebsiteChecker, urls)
	}
}
```

يختبر قياس الأداء دالة `CheckWebsites` باستخدام شريحة من مئة عنوان URL، ويستخدم تطبيقًا وهميًا (fake) جديدًا من `WebsiteChecker`. ودالة `slowStubWebsiteChecker` بطيئة عن قصد؛ فهي تستخدم `time.Sleep` لتنتظر عشرين مللي ثانية بالضبط ثم تُرجع true.


وعندما نشغّل قياس الأداء باستخدام `go test -bench=.` (أو `go test -bench="."` إن كنت تستخدم Windows Powershell):

```sh
pkg: github.com/gypsydave5/learn-go-with-tests/concurrency/v0
BenchmarkCheckWebsites-4               1        2249228637 ns/op
PASS
ok      github.com/gypsydave5/learn-go-with-tests/concurrency/v0        2.268s
```

تم قياس أداء `CheckWebsites` عند 2249228637 نانوثانية، أي نحو ثانيتين وربع.

لنحاول جعل هذا أسرع.

### اكتب كودًا كافيًا لنجاح الاختبار

الآن يمكننا أخيرًا الحديث عن التزامن (concurrency)، وهو يعني في سياق ما يلي "وجود أكثر من شيء قيد التنفيذ في الوقت نفسه". وهذا شيء نفعله بطبيعتنا كل يوم.

مثلًا، هذا الصباح أعددت كوبًا من الشاي. وضعت الغلاية على النار، وبينما كنت أنتظر غليان الماء أخرجت الحليب من الثلاجة، وأخرجت الشاي من الخزانة، ووجدت كوبي المفضل، ووضعت كيس الشاي في الكوب، ثم عندما غلى الماء سكبت الماء في الكوب.

وما _لم_ أفعله هو أن أضع الغلاية على النار ثم أقف أمامها محدقًا ببلاهة حتى تغلي، ثم أفعل كل شيء آخر بعد أن يغلي الماء.

إذا فهمت لماذا يكون إعداد الشاي بالطريقة الأولى أسرع، فستفهم كيف سنجعل `CheckWebsites` أسرع. فبدلًا من انتظار استجابة موقع قبل إرسال طلب إلى الموقع التالي، سنطلب من حاسوبنا إرسال الطلب التالي وهو في انتظار الاستجابة.

عادةً في Go، عندما نستدعي دالة مثل `doSomething()` فإننا ننتظر حتى تُرجع (وحتى إن لم يكن لديها قيمة تُرجعها، نظل ننتظر انتهاءها). ونقول إن هذه العملية *حاجبة* (blocking)، فهي تجعلنا ننتظر انتهاءها. أما العملية التي لا تحجب في Go فتعمل في *عملية* (process) منفصلة تُسمى *goroutine*. تخيل العملية كقارئ يقرأ صفحة كود Go من أعلى إلى أسفل، و"يدخل" داخل كل دالة عند استدعائها ليقرأ ما تفعله. وعندما تبدأ عملية منفصلة، فكأن قارئًا آخر يبدأ القراءة داخل الدالة، تاركًا القارئ الأصلي يواصل النزول في الصفحة.

ولإخبار Go ببدء goroutine جديدة نحوّل استدعاء الدالة إلى عبارة `go` بوضع الكلمة المفتاحية `go` قبله: `go doSomething()`.

```go
package concurrency

type WebsiteChecker func(string) bool

func CheckWebsites(wc WebsiteChecker, urls []string) map[string]bool {
	results := make(map[string]bool)

	for _, url := range urls {
		go func() {
			results[url] = wc(url)
		}()
	}

	return results
}
```

ولأن الطريقة الوحيدة لبدء goroutine هي وضع `go` قبل استدعاء دالة، نستخدم غالبًا *الدوال المجهولة* (anonymous functions) عندما نريد بدء goroutine. وشكل الدالة المجهولة يشبه تمامًا تعريف دالة عادية، لكن بلا اسم (ولا عجب في ذلك). ويمكنك رؤية واحدة منها بالأعلى داخل جسم حلقة `for`.

وللدوال المجهولة مزايا عديدة تجعلها مفيدة، ونستخدم اثنتين منها بالأعلى. أولًا، يمكن تنفيذها في الوقت نفسه الذي تُعرَّف فيه، وهذا ما تفعله `()` في نهاية الدالة المجهولة. وثانيًا، تحتفظ بالوصول إلى النطاق المعجمي (lexical scope) الذي عُرّفت فيه؛ فكل المتغيرات المتاحة عند تعريف الدالة المجهولة تظل متاحة أيضًا داخل جسم الدالة.

وجسم الدالة المجهولة بالأعلى هو نفسه جسم الحلقة السابق. والفرق الوحيد أن كل تكرار من الحلقة سيبدأ goroutine جديدة، متزامنة مع العملية الحالية (دالة `WebsiteChecker`). وستضيف كل goroutine نتيجتها إلى خريطة النتائج.

لكن عندما نشغّل `go test`:

```sh
--- FAIL: TestCheckWebsites (0.00s)
        CheckWebsites_test.go:31: Wanted map[http://google.com:true http://blog.gypsydave5.com:true waat://furhurterwe.geds:false], got map[]
FAIL
exit status 1
FAIL    github.com/gypsydave5/learn-go-with-tests/concurrency/v1        0.010s

```

### استطراد سريع في عالم التزامن...

قد لا تحصل على هذه النتيجة. فقد تظهر لك رسالة panic سنتحدث عنها بعد قليل. لا تقلق إن حدث ذلك، وواصل تشغيل الاختبار حتى _تحصل_ على النتيجة أعلاه. أو تظاهر بأنك حصلت عليها. الأمر يعود إليك. مرحبًا بك في التزامن: عندما لا يُعالَج بشكل صحيح يصعب التنبؤ بما سيحدث. لا تقلق، ولهذا نكتب الاختبارات: لتعرف متى نتعامل مع التزامن بشكل يمكن التنبؤ به.

### ... وعدنا.

لقد أمسك بنا الاختبار الأصلي `CheckWebsites`، فهو الآن يُرجع خريطة فارغة. ما الذي سار خطأً؟

لم تحصل أي من الـ goroutines التي بدأتها حلقة `for` على وقت كافٍ لتضيف نتيجتها إلى خريطة `results`؛ فدالة `CheckWebsites` أسرع منها، وتُرجع الخريطة وهي لا تزال فارغة.

لإصلاح ذلك يمكننا ببساطة الانتظار ريثما تنجز كل الـ goroutines عملها، ثم نُرجع. ثانيتان تكفيان، أليس كذلك؟

```go
package concurrency

import "time"

type WebsiteChecker func(string) bool

func CheckWebsites(wc WebsiteChecker, urls []string) map[string]bool {
	results := make(map[string]bool)

	for _, url := range urls {
		go func() {
			results[url] = wc(url)
		}()
	}

	time.Sleep(2 * time.Second)

	return results
}
```

الآن، إن كنت محظوظًا ستحصل على:

```sh
PASS
ok      github.com/gypsydave5/learn-go-with-tests/concurrency/v1        2.012s
```

لكن إن كنت غير محظوظ (وهذا أكثر احتمالًا إذا شغّلتها مع قياس الأداء لأنك ستحصل على محاولات أكثر)

```sh
fatal error: concurrent map writes

goroutine 8 [running]:
runtime.throw(0x12c5895, 0x15)
        /usr/local/Cellar/go/1.9.3/libexec/src/runtime/panic.go:605 +0x95 fp=0xc420037700 sp=0xc4200376e0 pc=0x102d395
runtime.mapassign_faststr(0x1271d80, 0xc42007acf0, 0x12c6634, 0x17, 0x0)
        /usr/local/Cellar/go/1.9.3/libexec/src/runtime/hashmap_fast.go:783 +0x4f5 fp=0xc420037780 sp=0xc420037700 pc=0x100eb65
github.com/gypsydave5/learn-go-with-tests/concurrency/v3.WebsiteChecker.func1(0xc42007acf0, 0x12d3938, 0x12c6634, 0x17)
        /Users/gypsydave5/go/src/github.com/gypsydave5/learn-go-with-tests/concurrency/v3/websiteChecker.go:12 +0x71 fp=0xc4200377c0 sp=0xc420037780 pc=0x12308f1
runtime.goexit()
        /usr/local/Cellar/go/1.9.3/libexec/src/runtime/asm_amd64.s:2337 +0x1 fp=0xc4200377c8 sp=0xc4200377c0 pc=0x105cf01
created by github.com/gypsydave5/learn-go-with-tests/concurrency/v3.WebsiteChecker
        /Users/gypsydave5/go/src/github.com/gypsydave5/learn-go-with-tests/concurrency/v3/websiteChecker.go:11 +0xa1

        ... many more scary lines of text ...
```

هذا طويل ومخيف، لكن كل ما علينا فعله هو أن نأخذ نفسًا عميقًا ونقرأ التتبع (stacktrace): `fatal error: concurrent map writes`. فأحيانًا، عند تشغيل اختباراتنا، تكتب اثنتان من الـ goroutines في خريطة النتائج في الوقت نفسه بالضبط. والـ maps في Go لا تحب أن يحاول أكثر من شيء الكتابة فيها في آن واحد، ومن هنا يأتي `fatal error`.

هذا *تسابق بيانات* (data race)، وهو خطأ يحدث عندما تصل اثنتان أو أكثر من الـ goroutines إلى موقع الذاكرة نفسه في وقت واحد، ويكون أحد هذه الوصولات على الأقل عملية كتابة. ولأننا لا نستطيع التحكم بدقة في وقت تنفيذ كل goroutine، فإننا معرضون لمحاولة عدة goroutines الكتابة في خريطة `results` في الوقت نفسه بالضبط. والـ maps في Go ليست آمنة للكتابة المتزامنة، لذا يطلق الـ runtime خطأً قاتلًا لمنع إفساد الذاكرة.

وتستطيع Go مساعدتنا في اكتشاف حالات التسابق (race conditions) عبر [_كاشف التسابق_][godoc_race_detector] المدمج فيها. ولتفعيل هذه الميزة، شغّل الاختبارات مع العلامة `race`: `go test -race`.

ومن المفترض أن تحصل على مخرجات تشبه هذه:

```sh
==================
WARNING: DATA RACE
Write at 0x00c420084d20 by goroutine 8:
  runtime.mapassign_faststr()
      /usr/local/Cellar/go/1.9.3/libexec/src/runtime/hashmap_fast.go:774 +0x0
  github.com/gypsydave5/learn-go-with-tests/concurrency/v3.WebsiteChecker.func1()
      /Users/gypsydave5/go/src/github.com/gypsydave5/learn-go-with-tests/concurrency/v3/websiteChecker.go:12 +0x82

Previous write at 0x00c420084d20 by goroutine 7:
  runtime.mapassign_faststr()
      /usr/local/Cellar/go/1.9.3/libexec/src/runtime/hashmap_fast.go:774 +0x0
  github.com/gypsydave5/learn-go-with-tests/concurrency/v3.WebsiteChecker.func1()
      /Users/gypsydave5/go/src/github.com/gypsydave5/learn-go-with-tests/concurrency/v3/websiteChecker.go:12 +0x82

Goroutine 8 (running) created at:
  github.com/gypsydave5/learn-go-with-tests/concurrency/v3.WebsiteChecker()
      /Users/gypsydave5/go/src/github.com/gypsydave5/learn-go-with-tests/concurrency/v3/websiteChecker.go:11 +0xc4
  github.com/gypsydave5/learn-go-with-tests/concurrency/v3.TestWebsiteChecker()
      /Users/gypsydave5/go/src/github.com/gypsydave5/learn-go-with-tests/concurrency/v3/websiteChecker_test.go:27 +0xad
  testing.tRunner()
      /usr/local/Cellar/go/1.9.3/libexec/src/testing/testing.go:746 +0x16c

Goroutine 7 (finished) created at:
  github.com/gypsydave5/learn-go-with-tests/concurrency/v3.WebsiteChecker()
      /Users/gypsydave5/go/src/github.com/gypsydave5/learn-go-with-tests/concurrency/v3/websiteChecker.go:11 +0xc4
  github.com/gypsydave5/learn-go-with-tests/concurrency/v3.TestWebsiteChecker()
      /Users/gypsydave5/go/src/github.com/gypsydave5/learn-go-with-tests/concurrency/v3/websiteChecker_test.go:27 +0xad
  testing.tRunner()
      /usr/local/Cellar/go/1.9.3/libexec/src/testing/testing.go:746 +0x16c
==================
```

التفاصيل مرة أخرى صعبة القراءة، لكن `WARNING: DATA RACE` واضحة كل الوضوح. وبالقراءة في متن الخطأ نرى اثنتين من الـ goroutines مختلفتين تكتبان في خريطة:

`Write at 0x00c420084d20 by goroutine 8:`

تكتب في كتلة الذاكرة نفسها التي كتبت فيها

`Previous write at 0x00c420084d20 by goroutine 7:`

وفوق ذلك، يمكننا رؤية سطر الكود الذي تحدث فيه الكتابة:

`/Users/gypsydave5/go/src/github.com/gypsydave5/learn-go-with-tests/concurrency/v3/websiteChecker.go:12`

وسطر الكود الذي تبدأ فيه الـ goroutines ‏7 و8:

`/Users/gypsydave5/go/src/github.com/gypsydave5/learn-go-with-tests/concurrency/v3/websiteChecker.go:11`

كل ما تحتاج معرفته مطبوع في طرفيتك، وكل ما عليك فعله هو أن تتحلى بالصبر الكافي لقراءته.

### القنوات (Channels)

يمكننا حل تسابق البيانات هذا بتنسيق الـ goroutines باستخدام *القنوات* (channels). والـ channels بنية بيانات في Go تستطيع استقبال القيم وإرسالها معًا. وتتيح هذه العمليات، بتفاصيلها، تواصلًا بين العمليات المختلفة.

في حالتنا هذه نريد التفكير في التواصل بين العملية الأب (parent process) وكل goroutine تُنشئها لأداء مهمة تشغيل دالة `WebsiteChecker` على عنوان URL.

```go
package concurrency

type WebsiteChecker func(string) bool
type result struct {
	string
	bool
}

func CheckWebsites(wc WebsiteChecker, urls []string) map[string]bool {
	results := make(map[string]bool)
	resultChannel := make(chan result)

	for _, url := range urls {
		go func() {
			resultChannel <- result{url, wc(url)}
		}()
	}

	for i := 0; i < len(urls); i++ {
		r := <-resultChannel
		results[r.string] = r.bool
	}

	return results
}
```

> **ملاحظة عن `url` داخل الـ goroutine.** كل تكرار من الحلقة يبدأ goroutine جديدة تشير إلى `url` دون تمريره صراحةً. ومنذ Go 1.22 أصبح هذا آمنًا: فقد عُدّلت مواصفات اللغة ليصبح `url` متغيرًا جديدًا في كل تكرار، فتحتفظ كل goroutine بنسختها الخاصة.
>
> وإذا شغّلت هذا في مشروع يعلن ملف `go.mod` فيه إصدار `go` *أقدم* من `1.22`، فستحصل على السلوك القديم بدلًا من ذلك: إذ يصبح `url` متغيرًا واحدًا تتشاركه كل التكرارات وتعيد استخدامه، لذا بحلول تشغيل الـ goroutines ترجّح أن ترى كلها القيمة نفسها (النهائية على الأرجح) لـ `url`. وهذا ما يُحيّر الناس، لأن *سلسلة أدوات* Go لديك قد تكون جديدة بينما توجيه `go` في `go.mod` قديم — فسلسلة الأدوات تحترم دلالة متغير الحلقة التي يوحي بها الإصدار المعلن. والإصلاح مع ملف `go.mod` قديم هو تمرير `url` إلى الـ goroutine صراحةً: `go func(url string) { ... }(url)`.

إلى جانب خريطة `results` أصبح لدينا الآن `resultChannel`، ننشئها بـ `make` بالطريقة نفسها. و`chan result` هو نوع الـ channel، أي channel من نوع `result`. وقد أُنشئ النوع الجديد `result` لربط القيمة المُرجعة من `WebsiteChecker` بعنوان URL الجاري فحصه؛ فهو struct مكوّن من `string` و`bool`. ولأننا لا نحتاج إلى تسمية أي من القيمتين، فكل منهما مجهولة داخل الـ struct؛ وقد يكون هذا مفيدًا عندما يصعب إيجاد اسم مناسب لقيمة.

الآن عند المرور على عناوين URL، بدلًا من الكتابة في `map` مباشرةً نرسل struct من نوع `result` عن كل استدعاء لـ `wc` إلى `resultChannel` بعبارة *إرسال* (send statement). وتستخدم هذه العبارة المعامل `<-` بحيث يكون الـ channel على اليسار والقيمة على اليمين:

```go
// Send statement
resultChannel <- result{url, wc(url)}
```

أما حلقة `for` التالية فتدور مرة واحدة عن كل عنوان URL. وفي داخلها نستخدم *تعبير استقبال* (receive expression)، الذي يسند إلى متغير قيمةً مستلمةً من channel. ويستخدم هذا أيضًا المعامل `<-`، لكن مع عكس المعاملين هذه المرة: فالـ channel الآن على اليمين والمتغير الذي نسند إليه على اليسار:

```go
// Receive expression
r := <-resultChannel
```

ثم نستخدم الـ `result` المستلم لتحديث الخريطة.

بإرسال النتائج إلى channel نستطيع التحكم في توقيت كل كتابة في خريطة النتائج، بما يضمن حدوثها واحدة تلو الأخرى. فرغم أن كل استدعاء من استدعاءات `wc`، وكل إرسال إلى channel النتائج، يحدث بالتزامن داخل عمليته الخاصة، فإن كل نتيجة تُعالَج واحدة تلو الأخرى بينما نستخرج القيم من channel النتائج بتعبير الاستقبال.

لقد استخدمنا التزامن في جزء الكود الذي أردنا تسريعه، مع ضمان أن الجزء الذي لا يمكن أن يحدث آنيًا يبقى تسلسليًا. وتواصلنا عبر العمليات المتعددة المشاركة باستخدام القنوات.

وعندما نشغّل قياس الأداء:

```sh
pkg: github.com/gypsydave5/learn-go-with-tests/concurrency/v2
BenchmarkCheckWebsites-8             100          23406615 ns/op
PASS
ok      github.com/gypsydave5/learn-go-with-tests/concurrency/v2        2.377s
```
23406615 نانوثانية، أي 0.023 ثانية، نحو مئة ضعف سرعة الدالة الأصلية. نجاح عظيم.

## الخلاصة

كان هذا التمرين أخف قليلًا من المعتاد في ما يخص التطوير الموجه بالاختبار (TDD). فبمعنى ما، شاركنا في إعادة هيكلة طويلة واحدة لدالة `CheckWebsites`؛ فالمدخلات والمخرجات لم تتغير قط، وإنما أصبحت أسرع فقط. لكن الاختبارات التي كانت لدينا، وقياس الأداء الذي كتبناه، أتاحا لنا إعادة هيكلة `CheckWebsites` بطريقة حافظت على ثقتنا بأن البرمجية ما زالت تعمل، مع إثبات أنها صارت أسرع فعلًا.

ولأننا جعلناها أسرع، تعلمنا عن

- *الـ goroutines*، وهي الوحدة الأساسية للتزامن في Go، إذ أتاحت لنا إدارة أكثر من طلب فحص موقع واحد.
- *الدوال المجهولة* (anonymous functions)، التي استخدمناها لبدء كل عملية من العمليات المتزامنة التي تفحص المواقع.
- *القنوات* (channels)، للمساعدة في تنظيم التواصل بين العمليات المختلفة والتحكم فيه، ما أتاح لنا تجنب خطأ *حالة التسابق* (race condition).
- *كاشف التسابق* (race detector) الذي ساعدنا في تصحيح مشكلات الكود المتزامن.

### اجعله سريعًا

من صيغ بناء البرمجيات بالطريقة الرشيقة (agile)، وهي كثيرًا ما تُنسب خطأً إلى Kent Beck:

> [اجعله يعمل، ثم اجعله صحيحًا، ثم اجعله سريعًا][wrf]

فكلمة "يعمل" تعني جعل الاختبارات تنجح، و"صحيحًا" تعني إعادة هيكلة الكود، و"سريعًا" تعني تحسين الكود ليجعل تشغيله سريعًا، على سبيل المثال. ولا نستطيع "جعله سريعًا" إلا بعد أن نجعله يعمل ونجعله صحيحًا. وقد كنا محظوظين لأن الكود الذي أُعطي لنا كان قد ثبت عمله بالفعل ولم يحتج إلى إعادة هيكلة. وينبغي ألا نحاول أبدًا "جعله سريعًا" قبل تنفيذ الخطوتين الأخريين، لأن

> [التحسين المبكر أصل كل الشرور][popt]
> -- Donald Knuth

[DI]: dependency-injection.md
[wrf]: http://wiki.c2.com/?MakeItWorkMakeItRightMakeItFast
[godoc_race_detector]: https://blog.golang.org/race-detector
[popt]: http://wiki.c2.com/?PrematureOptimization
