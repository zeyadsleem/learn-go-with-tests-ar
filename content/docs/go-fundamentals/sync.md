---
title: الـ Sync
weight: 140
---

# الـ Sync

**[يمكنك العثور على كل كود هذا الفصل هنا](https://github.com/quii/learn-go-with-tests/tree/main/sync)**

نريد أن نصنع عدّادًا (counter) يمكن استخدامه بأمان في الـ concurrency.

سنبدأ بعدّاد غير آمن ونتحقق من أن سلوكه سليم في بيئة أحادية الـ thread.

ثم نستعرض عدم أمانه عبر اختبار تُحاول فيه عدة goroutines استخدام العدّاد في الوقت نفسه، ثم نصلح المشكلة.

## اكتب الاختبار أولًا

نريد أن تمنحنا واجهة برمجتنا (API) method لزيادة العدّاد ثم استرجاع قيمته.

```go
func TestCounter(t *testing.T) {
	t.Run("incrementing the counter 3 times leaves it at 3", func(t *testing.T) {
		counter := Counter{}
		counter.Inc()
		counter.Inc()
		counter.Inc()

		if counter.Value() != 3 {
			t.Errorf("got %d, want %d", counter.Value(), 3)
		}
	})
}
```

## جرّب تشغيل الاختبار

```
./sync_test.go:9:14: undefined: Counter
```

## اكتب أصغر قدر من الكود ليعمل الاختبار وافحص مخرجاته الفاشلة

لنعرّف `Counter`.

```go
type Counter struct {
}
```

جرّب مرة أخرى، وسيفشل بالرسائل التالية

```
./sync_test.go:14:10: counter.Inc undefined (type Counter has no field or method Inc)
./sync_test.go:18:13: counter.Value undefined (type Counter has no field or method Value)
```

ولنجعل الاختبار يعمل أخيرًا يمكننا تعريف هذين الـ methods

```go
func (c *Counter) Inc() {

}

func (c *Counter) Value() int {
	return 0
}
```

من المفترض الآن أن يعمل ويفشل

```
=== RUN   TestCounter
=== RUN   TestCounter/incrementing_the_counter_3_times_leaves_it_at_3
--- FAIL: TestCounter (0.00s)
    --- FAIL: TestCounter/incrementing_the_counter_3_times_leaves_it_at_3 (0.00s)
    	sync_test.go:27: got 0, want 3
```

## اكتب كودًا كافيًا لنجاح الاختبار

لا ينبغي أن يكون هذا صعبًا على خبراء Go أمثالنا. علينا أن نحتفظ ببعض الحالة للعدّاد في نوع البيانات لدينا، ثم نزيدها مع كل استدعاء لـ `Inc`.

```go
type Counter struct {
	value int
}

func (c *Counter) Inc() {
	c.value++
}

func (c *Counter) Value() int {
	return c.value
}
```

## إعادة الهيكلة

ليس هناك الكثير لنعيد هيكلته، لكن بما أننا سنكتب المزيد من الاختبارات حول `Counter` فسنكتب دالة تحقق (assertion) صغيرة اسمها `assertCount` ليصبح الاختبار أوضح قليلًا عند القراءة.

```go
t.Run("incrementing the counter 3 times leaves it at 3", func(t *testing.T) {
	counter := Counter{}
	counter.Inc()
	counter.Inc()
	counter.Inc()

	assertCounter(t, counter, 3)
})
```
```go
func assertCounter(t testing.TB, got Counter, want int) {
	t.Helper()
	if got.Value() != want {
		t.Errorf("got %d, want %d", got.Value(), want)
	}
}
```

## الخطوات التالية

كان ذلك سهلًا بما يكفي، لكن أصبح لدينا الآن متطلب بأن يكون استخدامه آمنًا في بيئة تتعدد فيها الـ goroutines. وسنحتاج إلى كتابة اختبار فاشل لاستعراض ذلك.

## اكتب الاختبار أولًا

```go
t.Run("it runs safely concurrently", func(t *testing.T) {
	wantedCount := 1000
	counter := Counter{}

	var wg sync.WaitGroup
	wg.Add(wantedCount)

	for i := 0; i < wantedCount; i++ {
		go func() {
			counter.Inc()
			wg.Done()
		}()
	}
	wg.Wait()

	assertCounter(t, counter, wantedCount)
})
```

ستمر هذه الحلقة على `wantedCount` وتُطلق goroutine لاستدعاء `counter.Inc()`.

نحن نستخدم [`sync.WaitGroup`](https://golang.org/pkg/sync/#WaitGroup)، وهي طريقة مريحة لمزامنة العمليات التي تعمل في الوقت نفسه (concurrent processes).

> ينتظر الـ WaitGroup مجموعة من الـ goroutines حتى تنتهي. فيستدعي الـ goroutine الرئيسي الدالة Add لتحديد عدد الـ goroutines التي يجب انتظارها. ثم تعمل كل goroutine وتستدعي Done عند انتهائها. وفي الوقت نفسه، يمكن استخدام Wait للحجب (block) حتى تنتهي كل الـ goroutines.

وبانتظار انتهاء `wg.Wait()` قبل تنفيذ التحققات، نضمن أن كل الـ goroutines لدينا حاولت تنفيذ `Inc` على الـ `Counter`.

## جرّب تشغيل الاختبار

```
=== RUN   TestCounter/it_runs_safely_concurrently
--- FAIL: TestCounter (0.00s)
    --- FAIL: TestCounter/it_runs_safely_concurrently (0.00s)
    	sync_test.go:26: got 939, want 1000
FAIL
```

سيفشل الاختبار على _الأرجح_ برقم مختلف، لكنه مع ذلك يُظهر أنه لا يعمل عندما تحاول عدة goroutines تغيير قيمة العدّاد في الوقت نفسه.

### لماذا يحدث هذا؟

تبدو `c.value++` عملية واحدة غير قابلة للتجزئة، لكنها ليست كذلك. فهي اختصار لشيء أقرب إلى:

```go
tmp := c.value // 1. read
tmp = tmp + 1  // 2. increment
c.value = tmp  // 3. write
```

كل خطوة من هذه الخطوات الثلاث عملية مستقلة، ويستطيع الـ runtime في Go التبديل بين الـ goroutines في أي نقطة بينها. فإذا استدعت كلتا الـ goroutines الدالة `Inc` في الوقت نفسه تقريبًا، يمكن أن تتشابك خطواتهما، مثل:

```
goroutine A: reads c.value (0)
goroutine B: reads c.value (0)
goroutine A: increments its copy to 1
goroutine B: increments its copy to 1
goroutine A: writes c.value = 1
goroutine B: writes c.value = 1
```

استدعت الـ goroutines كلتاهما `Inc` مرة واحدة لكل منهما، لذا كنا نريد أن تنتهي `c.value` بالقيمة `2`، لكنها انتهت بالقيمة `1`. ضاعت إحدى الزيادتين بصمت لأن كلتا الـ goroutines قرأت القيمة نفسها قبل أن تكتب أي منهما نتيجتها. وتُسمى هذه _حالة تسابق_ (race condition)، ومع ألف goroutine تتسابق كلها على القراءة والزيادة والكتابة في الوقت نفسه، لا عجب أن تضيع بعض تلك الزيادات.

## اكتب كودًا كافيًا لنجاح الاختبار

الحل البسيط هو إضافة قفل (lock) إلى `Counter`، بحيث لا تستطيع إلا goroutine واحدة زيادة العدّاد في المرة الواحدة. ويوفر [`Mutex`](https://golang.org/pkg/sync/#Mutex) في Go قفلًا كهذا:

```go
type Counter struct {
	mu    sync.Mutex
	value int
}

func (c *Counter) Inc() {
	c.mu.Lock()
	defer c.mu.Unlock()
	c.value++
}
```

معنى ذلك أن أي goroutine تستدعي `Inc` ستحصل على قفل الـ `Counter` إذا كانت أول الواصلين. أما بقية الـ goroutines فسيكون عليها انتظار تحرير القفل بـ `Unlock` قبل أن تتمكن من الوصول.

إذا أعدت تشغيل الاختبار الآن فمن المفترض أن ينجح، لأن كل goroutine يجب أن تنتظر دورها قبل إجراء أي تغيير.

## رأيت أمثلة أخرى يُضمَّن فيها `sync.Mutex` داخل الـ struct.

قد ترى أمثلة كهذه

```go
type Counter struct {
	sync.Mutex
	value int
}
```

قد يُقال إن هذا يجعل الكود أكثر أناقة قليلًا.

```go
func (c *Counter) Inc() {
	c.Lock()
	defer c.Unlock()
	c.value++
}
```

قد تبدو هذه الطريقة _أنيقة_، لكن مع أن البرمجة مجال ذاتي إلى حد كبير، فهذا **سيئ وخاطئ**.

ينسى الناس أحيانًا أن تضمين الأنواع يعني أن methods ذلك النوع تصبح _جزءًا من الواجهة العامة_، وأنت في الغالب لا تريد ذلك. تذكّر أنه ينبغي أن نكون حذرين جدًا مع واجهاتنا العامة، فلحظة جعل شيء ما عامًا هي لحظة تمكن كود آخر من الارتباط به. ونريد دائمًا تجنّب الارتباط غير الضروري.

وكشف `Lock` و`Unlock` مربك في أفضل الأحوال، لكنه في أسوأ الأحوال قد يضر برمجيتك ضررًا كبيرًا إذا بدأ مستدعو نوعك باستدعاء هذين الـ methods.

![صورة توضح كيف يمكن لمستخدم هذه الواجهة تغيير حالة القفل تغييرًا خاطئًا](https://i.imgur.com/SWYNpwm.png)

_يبدو هذا فكرة سيئة حقًا_

## نسخ الـ mutexes

ينجح اختبارنا، لكن كودنا ما زال خطيرًا بعض الشيء.

إذا شغّلت `go vet` على كودك فينبغي أن تحصل على خطأ كهذا

```
sync/v2/sync_test.go:16: call of assertCounter copies lock value: v1.Counter contains sync.Mutex
sync/v2/sync_test.go:39: assertCounter passes lock by value: v1.Counter contains sync.Mutex
```

وإلقاء نظرة على توثيق [`sync.Mutex`](https://golang.org/pkg/sync/#Mutex) يخبرنا بالسبب

> لا يجوز نسخ الـ Mutex بعد أول استخدام.

عندما نمرّر `Counter` (بالقيمة) إلى `assertCounter` فسيحاول إنشاء نسخة من الـ mutex.

ولحل هذه المشكلة ينبغي أن نمرّر مؤشرًا (pointer) إلى `Counter` بدلًا من ذلك، لذا غيّر توقيع `assertCounter`

```go
func assertCounter(t testing.TB, got *Counter, want int)
```

لن تُترجم اختباراتنا بعد الآن لأننا نحاول تمرير `Counter` بدلًا من `*Counter`. ولحل ذلك أفضل إنشاء دالة إنشاء (constructor) توضح لقرّاء واجهتك البرمجية أنه من الأفضل ألا تهيّئ النوع بنفسك.

```go
func NewCounter() *Counter {
	return &Counter{}
}
```

استخدم هذه الدالة في اختباراتك عند تهيئة `Counter`.

## بديل: sync/atomic

`Mutex` أداة عامة الأغراض، فهي تحمي أي عدد من الحقول (fields) وأي علاقة ثابتة (invariant) بينها، ما دمت تتذكر تنفيذ `Lock`/`Unlock` حول كل عملية وصول. لكن `Counter` لدينا أبسط شكل ممكن للحالة المشتركة: عدد صحيح واحد تزيده عدة goroutines. ولهذه الحالة تحديدًا توفر حزمة [`sync/atomic`](https://pkg.go.dev/sync/atomic) أنواعًا مثل [`atomic.Int64`](https://pkg.go.dev/sync/atomic#Int64) تمنحك وصولًا متزامنًا آمنًا إلى قيمة واحدة دون قفل منفصل على الإطلاق:

```go
type Counter struct {
	value atomic.Int64
}

func NewCounter() *Counter {
	return &Counter{}
}

func (c *Counter) Inc() {
	c.value.Add(1)
}

func (c *Counter) Value() int64 {
	return c.value.Load()
}
```

لا `Mutex` ولا `Lock`/`Unlock`، ومع ذلك يظل استدعاء `Inc` آمنًا من أي عدد تريده من الـ goroutines في الوقت نفسه، والاختبار نفسه الذي كتبناه سابقًا ينجح دون تغيير. وتستخدم هذه الأنواع في العمق تعليمات CPU منخفضة المستوى تجعل الزيادة نفسها ذرية (atomic)، وهي عادةً أسرع من `Mutex` في حالات بسيطة كهذه. كما أن `atomic.Int64` (وأخواتها مثل `atomic.Int32` و`atomic.Bool`) لا يمكن استخدامها استخدامًا خاطئًا بالطريقة التي أمكن بها مع استدعاءات الدوال الخام من نمط `atomic.AddInt64(&x, 1)` في إصدارات Go الأقدم، فلا يمكنك نسيان تمرير مؤشر، ولا يمكنك قراءة الحقل عن طريق الخطأ دون المرور بـ `Load`.

الجأ إلى أنواع `sync/atomic` عندما تحمي قيمة واحدة؛ والجأ إلى `Mutex` حين تحتاج إلى إبقاء عدة حقول، أو علاقة ثابتة بينها، متناسقة معًا.

## الخلاصة

لقد غطّينا بضعة أمور من [حزمة sync](https://golang.org/pkg/sync/)

- تتيح لنا `Mutex` إضافة أقفال إلى بياناتنا
- الـ `WaitGroup` وسيلة لانتظار انتهاء الـ goroutines من مهامها
- توفر `sync/atomic` وصولًا آمنًا بلا أقفال إلى القيم المفردة، ويستحق اللجوء إليها عندما يكون استخدام `Mutex` كاملًا مبالغة

### متى تستخدم الأقفال بدلًا من الـ channels والـ goroutines؟

[لقد غطّينا الـ goroutines سابقًا في أول فصل عن الـ concurrency](concurrency.md)، وهي التي تتيح لنا كتابة كود متزامن آمن، فلماذا إذن تستخدم الأقفال؟
[وفي wiki الخاص بـ Go صفحة مخصصة لهذا الموضوع: Mutex Or Channel](https://go.dev/wiki/MutexOrChannel)

> من الأخطاء الشائعة عند المبتدئين في Go الإفراط في استخدام الـ channels والـ goroutines لمجرد أن ذلك ممكن، أو لأنه ممتع. فلا تخف من استخدام sync.Mutex إذا كان الأنسب لمشكلتك. فـ Go عملية (pragmatic) وتتيح لك استخدام الأدوات التي تحل مشكلتك على أفضل وجه، ولا تفرض عليك أسلوبًا واحدًا في الكتابة.

وبإعادة الصياغة:

- **استخدم الـ channels عند نقل ملكية البيانات**
- **استخدم الـ mutexes لإدارة الحالة**

### go vet

تذكّر استخدام go vet في سكربتات البناء، فهو ينبّهك إلى بعض الأخطاء الخفية في كودك قبل أن تصطدم بمستخدميك المساكين.

### لا تستخدم التضمين لمجرد أنه مريح

- فكّر في تأثير التضمين على واجهتك العامة (public API).
- هل تريد _حقًا_ كشف هذه الـ methods وربط الناس أكوادهم بها؟
- وفيما يخص الـ mutexes، قد يكون هذا كارثيًا بطرق غريبة وغير متوقعة جدًا، فتخيّل كودًا خبيثًا يحرّر mutex دون أن ينبغي له ذلك؛ سيؤدي هذا إلى أخطاء غريبة جدًا يصعب تتبعها.
