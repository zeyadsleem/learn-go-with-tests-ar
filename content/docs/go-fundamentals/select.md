---
title: الـ Select
weight: 120
---

# الـ Select

**[يمكنك العثور على كل كود هذا الفصل هنا](https://github.com/quii/learn-go-with-tests/tree/main/select)**

طُلب منك أن تكتب دالة اسمها `WebsiteRacer` تأخذ رابطين (URLs) و"تتسابق" بينهما بإرسال طلب HTTP GET إلى كل منهما، ثم تُرجع الرابط الذي عاد أولًا. وإذا لم يعد أي منهما خلال 10 ثوانٍ، فعليها أن تُرجع `error`.

ولهذا سنستخدم:

- `net/http` لإجراء استدعاءات HTTP.
- `net/http/httptest` لمساعدتنا في اختبارها.
- الـ goroutines.
- `select` لمزامنة العمليات.

## اكتب الاختبار أولًا

لنبدأ بشيء بسيط (naive) لننطلق منه.

```go
func TestRacer(t *testing.T) {
	slowURL := "http://www.facebook.com"
	fastURL := "http://www.quii.dev"

	want := fastURL
	got := Racer(slowURL, fastURL)

	if got != want {
		t.Errorf("got %q, want %q", got, want)
	}
}
```

نعرف أن هذا ليس مثاليًا وفيه مشكلات، لكنه بداية. ومن المهم ألا ننشغل كثيرًا بمحاولة إتقان كل شيء من المرة الأولى.

## جرّب تشغيل الاختبار

`./racer_test.go:14:9: undefined: Racer`

## اكتب أصغر قدر من الكود ليعمل الاختبار وافحص مخرجاته الفاشلة

```go
func Racer(a, b string) (winner string) {
	return
}
```

`racer_test.go:25: got '', want 'http://www.quii.dev'`

## اكتب كودًا كافيًا لنجاح الاختبار

```go
func Racer(a, b string) (winner string) {
	startA := time.Now()
	resp, err := http.Get(a)
	if err == nil {
		resp.Body.Close()
	}
	aDuration := time.Since(startA)

	startB := time.Now()
	resp, err = http.Get(b)
	if err == nil {
		resp.Body.Close()
	}
	bDuration := time.Since(startB)

	if aDuration < bDuration {
		return a
	}

	return b
}
```

لكل رابط:

1. نستخدم `time.Now()` لتسجيل الوقت قبل محاولة جلب الـ `URL` مباشرة.
1. ثم نستخدم [`http.Get`](https://golang.org/pkg/net/http/#Client.Get) لمحاولة تنفيذ طلب HTTP من نوع `GET` على الـ `URL`. وتُرجع هذه الدالة [`http.Response`](https://golang.org/pkg/net/http/#Response) و`error`. ونغلق جسم الاستجابة لتجنب تسريب واصفات الملفات (file descriptors) — فإن لم نفعل ذلك، سيترك كل طلب اتصالًا مفتوحًا.
1. يأخذ `time.Since` وقت البداية ويُرجع `time.Duration` يمثل الفرق.

وبعد أن نفعل ذلك، نقارن المدد ببساطة لنعرف أي الرابطين أسرع.

### المشكلات

قد ينجح هذا الاختبار عندك وقد لا ينجح. فالمشكلة أننا نتصل بمواقع حقيقية لاختبار منطقنا الخاص.

اختبار الكود الذي يستخدم HTTP أمر شائع جدًا، لذا توفر Go أدوات في مكتبتها القياسية تساعدك على اختباره.

في فصلي الـ mocking وحقن الاعتماديات (dependency injection)، تحدثنا عن أننا لا نريد في الوضع المثالي الاعتماد على خدمات خارجية لاختبار كودنا، لأنها قد تكون:

- بطيئة
- متقلبة (flaky)
- لا تسمح باختبار الحالات الحدّية (edge cases)

وفي المكتبة القياسية حزمة اسمها [`net/http/httptest`](https://golang.org/pkg/net/http/httptest/) تتيح للمستخدمين إنشاء خادم HTTP وهمي (mock) بسهولة.

لنغيّر اختباراتنا لتستخدم الـ mocks، فيصبح لدينا خوادم موثوقة نختبر مقابلها ونتحكم فيها.

```go
func TestRacer(t *testing.T) {

	slowServer := httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		time.Sleep(20 * time.Millisecond)
		w.WriteHeader(http.StatusOK)
	}))

	fastServer := httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		w.WriteHeader(http.StatusOK)
	}))

	slowURL := slowServer.URL
	fastURL := fastServer.URL

	want := fastURL
	got := Racer(slowURL, fastURL)

	if got != want {
		t.Errorf("got %q, want %q", got, want)
	}

	slowServer.Close()
	fastServer.Close()
}
```

قد تبدو الصياغة مزدحمة قليلًا، لكن خذ وقتك.

يستقبل `httptest.NewServer` قيمة من نوع `http.HandlerFunc` نمررها عبر _دالة مجهولة (anonymous function)_.

`http.HandlerFunc` نوع يبدو هكذا: `type HandlerFunc func(ResponseWriter, *Request)`.

وكل ما تعنيه حقًا أنها تحتاج دالة تأخذ `ResponseWriter` و`Request`، وهذا ليس مفاجئًا كثيرًا في خادم HTTP.

واتضح أنه لا يوجد سحر إضافي هنا فعلًا، **فهكذا أيضًا ستكتب خادم HTTP _حقيقيًا_ في Go**. والفرق الوحيد أننا نلفّه في `httptest.NewServer`، ما يسهّل استخدامه في الاختبار، إذ يجد منفذًا مفتوحًا للاستماع عليه، ثم يمكنك إغلاقه عند انتهاء اختبارك.

داخل خادمينا، جعلنا الخادم البطيء ينام قليلًا بـ `time.Sleep` عند وصول طلب، ليكون أبطأ من الآخر. ثم يكتب كلا الخادمين استجابة `OK` بـ `w.WriteHeader(http.StatusOK)` عائدة إلى المستدعي.

إذا أعدت تشغيل الاختبار فسينجح الآن بالتأكيد، وسيكون أسرع. وجرّب التلاعب بقيم النوم هذه لتُفشل الاختبار عمدًا.

## إعادة الهيكلة

لدينا بعض التكرار في كود الإنتاج وكود الاختبار معًا.

```go
func Racer(a, b string) (winner string) {
	aDuration := measureResponseTime(a)
	bDuration := measureResponseTime(b)

	if aDuration < bDuration {
		return a
	}

	return b
}

func measureResponseTime(url string) time.Duration {
	start := time.Now()
	resp, err := http.Get(url)
	if err == nil {
		resp.Body.Close()
	}
	return time.Since(start)
}
```

هذا التخلص من التكرار (DRY-ing up) يجعل كود `Racer` أسهل قراءة بكثير.

```go
func TestRacer(t *testing.T) {

	slowServer := makeDelayedServer(20 * time.Millisecond)
	fastServer := makeDelayedServer(0 * time.Millisecond)

	defer slowServer.Close()
	defer fastServer.Close()

	slowURL := slowServer.URL
	fastURL := fastServer.URL

	want := fastURL
	got := Racer(slowURL, fastURL)

	if got != want {
		t.Errorf("got %q, want %q", got, want)
	}
}

func makeDelayedServer(delay time.Duration) *httptest.Server {
	return httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		time.Sleep(delay)
		w.WriteHeader(http.StatusOK)
	}))
}
```

أعدنا هيكلة إنشاء خوادمنا الوهمية (fake) إلى دالة اسمها `makeDelayedServer` لننقل بعض الكود غير المثير للاهتمام خارج الاختبار ونقلل التكرار.

### ‏`defer`

عندما تسبق استدعاء دالة بـ `defer`، فسيستدعي ذلك الدالة _في نهاية الدالة التي تحتويه_.

أحيانًا ستحتاج إلى تحرير الموارد، مثل إغلاق ملف، أو في حالتنا إغلاق خادم حتى لا يواصل الاستماع إلى منفذ.

تريد أن يُنفَّذ ذلك في نهاية الدالة، لكن مع إبقاء التعليمة قريبة من المكان الذي أنشأت فيه الخادم، خدمةً لقارئ الكود مستقبلًا.

إعادة هيكلتنا تحسين وحل معقول بالنظر إلى ميزات Go التي غطيناها حتى الآن، لكن يمكننا جعل الحل أبسط.

### مزامنة العمليات

- لماذا نختبر سرعة الموقعين واحدًا تلو الآخر بينما Go بارعة في الـ concurrency؟ ينبغي أن نستطيع فحص كليهما في الوقت نفسه.
- لا يهمنا فعلًا _زمن الاستجابة الدقيق_ للطلبات، كل ما نريد معرفته هو أيهما يعود أولًا.

وللقيام بذلك، سنقدّم تركيبًا جديدًا اسمه `select` يساعدنا على مزامنة العمليات بسهولة ووضوح شديدين.

```go
func Racer(a, b string) (winner string) {
	select {
	case <-ping(a):
		return a
	case <-ping(b):
		return b
	}
}

func ping(url string) chan struct{} {
	ch := make(chan struct{})
	go func() {
		resp, err := http.Get(url)
		if err == nil {
			resp.Body.Close()
		}
		close(ch)
	}()
	return ch
}
```

#### ‏`ping`

عرّفنا دالة `ping` تُنشئ `chan struct{}` وتُرجعه.

في حالتنا، لا _يهمنا_ نوع القيمة المُرسلة إلى الـ channel، _كل ما نريده هو الإشارة إلى أننا انتهينا_، وإغلاق الـ channel يؤدي الغرض تمامًا!

لماذا `struct{}` وليس نوعًا آخر مثل `bool`؟ حسنًا، `chan struct{}` أصغر نوع بيانات متاح من ناحية الذاكرة، فلا يحدث أي تخصيص (allocation) مقارنة بـ `bool`. وبما أننا نغلق الـ chan ولا نرسل عليه أي شيء، فلماذا نخصص أي ذاكرة؟

وداخل الدالة نفسها، نبدأ goroutine ترسل إشارة إلى ذلك الـ channel بمجرد اكتمال `http.Get(url)`. ونغلق جسم الاستجابة فورًا — فكل ما يهمنا هو أن الطلب اكتمل، لا محتوى الاستجابة، وتركه مفتوحًا سيسبب تسريب واصفات ملفات (file descriptors).

##### أنشئ الـ channels دائمًا بـ `make`

لاحظ أننا مضطرون لاستخدام `make` عند إنشاء channel، بدلًا من كتابة `var ch chan struct{}` مثلًا. فعندما تستخدم `var` سيُهيَّأ المتغير بالقيمة "الصفرية" (zero value) للنوع. فهي للنص `string` تساوي `""`، ولـ `int` تساوي 0، وهكذا.

أما بالنسبة للـ channels فالقيمة الصفرية هي `nil`، وإذا حاولت الإرسال إليها بـ `<-` فسيُحجب (block) إلى الأبد لأنك لا تستطيع الإرسال إلى channels قيمتها `nil`.

[يمكنك رؤية هذا عمليًا في The Go Playground](https://play.golang.org/p/IIbeAox5jKA)
#### ‏`select`

ستتذكر من فصل الـ concurrency أنك تستطيع انتظار القيم المُرسلة إلى channel بـ `myVar := <-ch`. وهذا استدعاء _حاجب (blocking)_، لأنك تنتظر قيمة.

يتيح لك `select` الانتظار على _عدة_ channels. وأول channel يرسل قيمة "يفوز"، ويُنفَّذ الكود الموجود تحت `case` المقابلة.

نستخدم `ping` داخل `select` لتهيئة channelين، واحد لكل `URL` من روابطنا. وأيّهما يكتب إلى الـ channel الخاص به أولًا سيُنفَّذ الكود المقابل له في `select`، فيُرجَع الـ `URL` الخاص به (ويكون هو الفائز).

بعد هذه التغييرات، صار القصد وراء كودنا واضحًا جدًا، وصار التنفيذ أبسط في الواقع.

### المهلات الزمنية (Timeouts)

كان متطلبنا الأخير أن تُرجع الدالة خطأً إذا استغرقت `Racer` أكثر من 10 ثوانٍ.

## اكتب الاختبار أولًا

```go
func TestRacer(t *testing.T) {
	t.Run("compares speeds of servers, returning the url of the fastest one", func(t *testing.T) {
		slowServer := makeDelayedServer(20 * time.Millisecond)
		fastServer := makeDelayedServer(0 * time.Millisecond)

		defer slowServer.Close()
		defer fastServer.Close()

		slowURL := slowServer.URL
		fastURL := fastServer.URL

		want := fastURL
		got, _ := Racer(slowURL, fastURL)

		if got != want {
			t.Errorf("got %q, want %q", got, want)
		}
	})

	t.Run("returns an error if a server doesn't respond within 10s", func(t *testing.T) {
		serverA := makeDelayedServer(11 * time.Second)
		serverB := makeDelayedServer(12 * time.Second)

		defer serverA.Close()
		defer serverB.Close()

		_, err := Racer(serverA.URL, serverB.URL)

		if err == nil {
			t.Error("expected an error but didn't get one")
		}
	})
}
```

جعلنا خوادم الاختبار تستغرق أكثر من 10 ثوانٍ لتعود، لممارسة هذا السيناريو، ونتوقع الآن أن تُرجع `Racer` قيمتين: الرابط الفائز (الذي نتجاهله في هذا الاختبار بـ `_`) و`error`.

لاحظ أننا تعاملنا أيضًا مع قيمة الخطأ المُرجَعة في اختبارنا الأصلي، ونستخدم `_` الآن لضمان تشغيل الاختبارات.

## جرّب تشغيل الاختبار

`./racer_test.go:37:10: assignment mismatch: 2 variables but Racer returns 1 value`

## اكتب أصغر قدر من الكود ليعمل الاختبار وافحص مخرجاته الفاشلة

```go
func Racer(a, b string) (winner string, error error) {
	select {
	case <-ping(a):
		return a, nil
	case <-ping(b):
		return b, nil
	}
}
```

غيّر توقيع `Racer` ليُرجع الفائز و`error`. وأرجع `nil` في الحالات السعيدة.

إذا شغّلته الآن فسيفشل بعد 11 ثانية.

```
--- FAIL: TestRacer (12.00s)
    --- FAIL: TestRacer/returns_an_error_if_a_server_doesn't_respond_within_10s (12.00s)
        racer_test.go:40: expected an error but didn't get one
```

## اكتب كودًا كافيًا لنجاح الاختبار

```go
func Racer(a, b string) (winner string, error error) {
	select {
	case <-ping(a):
		return a, nil
	case <-ping(b):
		return b, nil
	case <-time.After(10 * time.Second):
		return "", fmt.Errorf("timed out waiting for %s and %s", a, b)
	}
}
```

`time.After` دالة مفيدة جدًا عند استخدام `select`. ومع أن ذلك لم يحدث في حالتنا، فقد تكتب كودًا يُحجب إلى الأبد إذا لم تُرجع الـ channels التي تستمع إليها أي قيمة. و`time.After` تُرجع `chan` (مثل `ping`) وترسل إشارة عبرها بعد المدة التي تحددها.

وهذا مثالي لنا؛ فإذا نجح `a` أو `b` في العودة فهو الفائز، أما إذا وصلنا إلى 10 ثوانٍ فستُرسل `time.After` إشارة ونُعيد `error`.

### الاختبارات البطيئة

المشكلة أن هذا الاختبار يستغرق 10 ثوانٍ ليعمل. ولمنطق بهذه البساطة، هذا ليس شعورًا جيدًا.

ما يمكننا فعله هو جعل المهلة الزمنية قابلة للتهيئة. فيكون لدينا في اختبارنا مهلة قصيرة جدًا، ثم عندما يُستخدم الكود في العالم الحقيقي يمكن ضبطها على 10 ثوانٍ.

```go
func Racer(a, b string, timeout time.Duration) (winner string, error error) {
	select {
	case <-ping(a):
		return a, nil
	case <-ping(b):
		return b, nil
	case <-time.After(timeout):
		return "", fmt.Errorf("timed out waiting for %s and %s", a, b)
	}
}
```

لن تترجم اختباراتنا الآن لأننا لا نمرر مهلة زمنية.

وقبل أن نتعجل بإضافة هذه القيمة الافتراضية إلى كلا اختبارينا، لنصغِ إليهما.

- هل تهمنا المهلة الزمنية في اختبار "الحالة السعيدة"؟
- كانت المتطلبات صريحة بشأن المهلة الزمنية.

بناءً على هذه المعرفة، لنُجرِ قليلًا من إعادة الهيكلة التي تراعي كلا اختبارينا ومستخدمي كودنا.

```go
var tenSecondTimeout = 10 * time.Second

func Racer(a, b string) (winner string, error error) {
	return ConfigurableRacer(a, b, tenSecondTimeout)
}

func ConfigurableRacer(a, b string, timeout time.Duration) (winner string, error error) {
	select {
	case <-ping(a):
		return a, nil
	case <-ping(b):
		return b, nil
	case <-time.After(timeout):
		return "", fmt.Errorf("timed out waiting for %s and %s", a, b)
	}
}
```

يستطيع مستخدمونا واختبارنا الأول استخدام `Racer` (التي تستخدم `ConfigurableRacer` في الخلفية)، ويستطيع اختبار المسار الحزين (sad path) استخدام `ConfigurableRacer`.

```go
func TestRacer(t *testing.T) {

	t.Run("compares speeds of servers, returning the url of the fastest one", func(t *testing.T) {
		slowServer := makeDelayedServer(20 * time.Millisecond)
		fastServer := makeDelayedServer(0 * time.Millisecond)

		defer slowServer.Close()
		defer fastServer.Close()

		slowURL := slowServer.URL
		fastURL := fastServer.URL

		want := fastURL
		got, err := Racer(slowURL, fastURL)

		if err != nil {
			t.Fatalf("did not expect an error but got one %v", err)
		}

		if got != want {
			t.Errorf("got %q, want %q", got, want)
		}
	})

	t.Run("returns an error if a server doesn't respond within the specified time", func(t *testing.T) {
		server := makeDelayedServer(25 * time.Millisecond)

		defer server.Close()

		_, err := ConfigurableRacer(server.URL, server.URL, 20*time.Millisecond)

		if err == nil {
			t.Error("expected an error but didn't get one")
		}
	})
}
```

أضفت فحصًا أخيرًا في الاختبار الأول للتأكد من أننا لا نحصل على `error`.

## الخلاصة

### ‏`select`

- تساعدك على الانتظار على عدة channels.
- أحيانًا ستحتاج إلى تضمين `time.After` في إحدى `case` لديك لمنع نظامك من الحجب إلى الأبد.

### ‏`httptest`

- طريقة مريحة لإنشاء خوادم اختبار، لتحصل على اختبارات موثوقة وقابلة للتحكم.
- تستخدم الواجهات نفسها التي تستخدمها خوادم `net/http` "الحقيقية"، وهذا متسق ويوفر عليك الكثير لتتعلمه.
