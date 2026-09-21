---
title: الـ Context
weight: 150
---

# الـ Context

**[يمكنك العثور على كل كود هذا الفصل هنا](https://github.com/quii/learn-go-with-tests/tree/main/context)**

كثيرًا ما تُطلق البرمجيات عمليات طويلة الأمد وشرهة للموارد (وغالبًا في صورة goroutines). وإذا أُلغيت العملية التي تسببت في ذلك أو فشلت لسبب ما، فستحتاج إلى إيقاف هذه العمليات بطريقة متسقة في كل أنحاء تطبيقك.

وإذا لم تدِر ذلك، فقد يبدأ تطبيقك الـ Go السريع الذي تفتخر به في الظهور بمشكلات أداء يصعب تتبعها.

في هذا الفصل سنستخدم حزمة `context` لمساعدتنا في إدارة العمليات طويلة الأمد.

سنبدأ بمثال كلاسيكي لخادم ويب يبدأ عند وصول طلب إليه عملية قد تكون طويلة الأمد لجلب بعض البيانات ليردّها في الاستجابة.

وسنمضي في سيناريو يلغي فيه المستخدم الطلب قبل أن تُجلب البيانات، ونتأكد أن العملية تُبلَّغ بأن تتخلى عن مهمتها.

لقد أعددت بعض الكود على المسار السعيد لننطلق منه. هذا هو كود خادمنا.

```go
func Server(store Store) http.HandlerFunc {
	return func(w http.ResponseWriter, r *http.Request) {
		fmt.Fprint(w, store.Fetch())
	}
}
```

تأخذ دالة `Server` قيمة من نوع `Store` وتُرجع لنا `http.HandlerFunc`. و`Store` معرَّفة هكذا:

```go
type Store interface {
	Fetch() string
}
```

وتستدعي الدالة المُرجَعة الـ method المسماة `Fetch` على `store` للحصول على البيانات ثم تكتبها في الاستجابة.

ولدينا spy مقابل لـ `Store` نستخدمه في اختبار.

```go
type SpyStore struct {
	response string
}

func (s *SpyStore) Fetch() string {
	return s.response
}

func TestServer(t *testing.T) {
	data := "hello, world"
	svr := Server(&SpyStore{data})

	request := httptest.NewRequest(http.MethodGet, "/", nil)
	response := httptest.NewRecorder()

	svr.ServeHTTP(response, request)

	if response.Body.String() != data {
		t.Errorf(`got "%s", want "%s"`, response.Body.String(), data)
	}
}
```

وبعد أن أصبح لدينا مسار سعيد، نريد صنع سيناريو أكثر واقعية لا يستطيع فيه `Store` إنهاء `Fetch` قبل أن يلغي المستخدم الطلب.

## اكتب الاختبار أولًا

سيحتاج المعالج (handler) الخاص بنا إلى طريقة يخبر بها `Store` أن يلغي العمل، لذا حدّث الواجهة.

```go
type Store interface {
	Fetch() string
	Cancel()
}
```

سنحتاج إلى تعديل الـ spy ليستغرق بعض الوقت قبل إرجاع `data`، وليكون لديه سبيل لمعرفة أنه تُلقّي أمر الإلغاء. وسيكون عليه إضافة `Cancel` كـ method ليحقق واجهة `Store`.

```go
type SpyStore struct {
	response  string
	cancelled bool
}

func (s *SpyStore) Fetch() string {
	time.Sleep(100 * time.Millisecond)
	return s.response
}

func (s *SpyStore) Cancel() {
	s.cancelled = true
}
```

لنضف اختبارًا جديدًا نلغي فيه الطلب قبل مرور 100 مللي ثانية، ونفحص الـ store لنرى هل يُلغى عمله.

```go
t.Run("tells store to cancel work if request is cancelled", func(t *testing.T) {
	data := "hello, world"
	store := &SpyStore{response: data}
	svr := Server(store)

	request := httptest.NewRequest(http.MethodGet, "/", nil)

	cancellingCtx, cancel := context.WithCancel(request.Context())
	time.AfterFunc(5*time.Millisecond, cancel)
	request = request.WithContext(cancellingCtx)

	response := httptest.NewRecorder()

	svr.ServeHTTP(response, request)

	if !store.cancelled {
		t.Error("store was not told to cancel")
	}
})
```

من [مدونة Go: الـ Context](https://blog.golang.org/context)

> توفر حزمة context دوالًا لاشتقاق قيم Context جديدة من قيم موجودة. وتشكّل هذه القيم شجرة: فعندما يُلغى Context، تُلغى أيضًا كل الـ Contexts المشتقة منه.

من المهم أن تشتق الـ contexts الخاصة بك حتى تنتشر عمليات الإلغاء في كل أنحاء مكدس الاستدعاءات (call stack) لطلب معين.

وما نفعله هو اشتقاق `cancellingCtx` جديدة من `request` الخاص بنا، وهي تُرجع لنا دالة `cancel`. ثم نجدول استدعاء تلك الدالة بعد 5 مللي ثانية باستخدام `time.AfterFunc`. وأخيرًا نستخدم هذا الـ context الجديد في طلبنا باستدعاء `request.WithContext`.

## جرّب تشغيل الاختبار

يفشل الاختبار كما توقعنا.

```
--- FAIL: TestServer (0.00s)
    --- FAIL: TestServer/tells_store_to_cancel_work_if_request_is_cancelled (0.00s)
    	context_test.go:62: store was not told to cancel
```

## اكتب كودًا كافيًا لنجاح الاختبار

تذكّر أن تكون منضبطًا مع التطوير الموجه بالاختبار (TDD). اكتب _أصغر_ قدر من الكود لنجاح اختبارنا.

```go
func Server(store Store) http.HandlerFunc {
	return func(w http.ResponseWriter, r *http.Request) {
		store.Cancel()
		fmt.Fprint(w, store.Fetch())
	}
}
```

هذا يجعل الاختبار ينجح، لكنه لا يمنح شعورًا جيدًا، أليس كذلك؟ فمن المؤكد أننا لا ينبغي أن نستدعي `Cancel()` قبل أن نجلب البيانات في _كل طلب_.

وبانضباطنا، أبرز هذا خللًا في اختباراتنا، وهذا أمر جيد!

سنحتاج إلى تحديث اختبار المسار السعيد ليتأكد أن العملية لا تُلغى.

```go
t.Run("returns data from store", func(t *testing.T) {
	data := "hello, world"
	store := &SpyStore{response: data}
	svr := Server(store)

	request := httptest.NewRequest(http.MethodGet, "/", nil)
	response := httptest.NewRecorder()

	svr.ServeHTTP(response, request)

	if response.Body.String() != data {
		t.Errorf(`got "%s", want "%s"`, response.Body.String(), data)
	}

	if store.cancelled {
		t.Error("it should not have cancelled the store")
	}
})
```

شغّل الاختبارين، وسيفشل الآن اختبار المسار السعيد، فيُجبرنا ذلك على تنفيذ أكثر منطقية.

```go
func Server(store Store) http.HandlerFunc {
	return func(w http.ResponseWriter, r *http.Request) {
		ctx := r.Context()

		data := make(chan string, 1)

		go func() {
			data <- store.Fetch()
		}()

		select {
		case d := <-data:
			fmt.Fprint(w, d)
		case <-ctx.Done():
			store.Cancel()
		}
	}
}
```

ماذا فعلنا هنا؟

لدى `context` الـ method المسماة `Done()` التي تُرجع channel يُرسل إليه إشارة عندما يصبح الـ context "منتهيًا" أو "ملغى". ونريد أن نستمع إلى تلك الإشارة ونستدعي `store.Cancel` إذا وصلتنا، لكننا نريد تجاهلها إذا نجح `Store` في تنفيذ `Fetch` قبلها.

ولإدارة ذلك نشغّل `Fetch` داخل goroutine، وستكتب النتيجة في channel جديدة اسمها `data`. ثم نستخدم `select` لنُجري سباقًا فعليًا بين العمليتين غير المتزامنتين، فنكتب استجابة أو نستدعي `Cancel`.

## إعادة الهيكلة

يمكننا إعادة هيكلة كود اختبارنا قليلًا بصنع methods للتحقق (assertion) على الـ spy

```go
type SpyStore struct {
	response  string
	cancelled bool
	t         *testing.T
}

func (s *SpyStore) assertWasCancelled() {
	s.t.Helper()
	if !s.cancelled {
		s.t.Error("store was not told to cancel")
	}
}

func (s *SpyStore) assertWasNotCancelled() {
	s.t.Helper()
	if s.cancelled {
		s.t.Error("store was told to cancel")
	}
}
```

تذكّر تمرير `*testing.T` عند إنشاء الـ spy.

```go
func TestServer(t *testing.T) {
	data := "hello, world"

	t.Run("returns data from store", func(t *testing.T) {
		store := &SpyStore{response: data, t: t}
		svr := Server(store)

		request := httptest.NewRequest(http.MethodGet, "/", nil)
		response := httptest.NewRecorder()

		svr.ServeHTTP(response, request)

		if response.Body.String() != data {
			t.Errorf(`got "%s", want "%s"`, response.Body.String(), data)
		}

		store.assertWasNotCancelled()
	})

	t.Run("tells store to cancel work if request is cancelled", func(t *testing.T) {
		store := &SpyStore{response: data, t: t}
		svr := Server(store)

		request := httptest.NewRequest(http.MethodGet, "/", nil)

		cancellingCtx, cancel := context.WithCancel(request.Context())
		time.AfterFunc(5*time.Millisecond, cancel)
		request = request.WithContext(cancellingCtx)

		response := httptest.NewRecorder()

		svr.ServeHTTP(response, request)

		store.assertWasCancelled()
	})
}
```

هذه المقاربة مقبولة، لكن هل هي الأسلوب المألوف في Go (idiomatic)؟

هل من المنطقي أن يهتم خادم الويب الخاص بنا بإلغاء `Store` يدويًا؟ وماذا لو كان `Store` يعتمد أيضًا على عمليات أخرى بطيئة؟ سيتعين علينا التأكد أن `Store.Cancel` ينشر الإلغاء بشكل صحيح إلى كل ما يعتمد عليه.

أحد الأهداف الرئيسية لـ `context` أنه طريقة متسقة لتقديم الإلغاء.

[من توثيق Go](https://golang.org/pkg/context/)

> ينبغي للطلبات الواردة إلى خادم أن تُنشئ Context، وينبغي للاستدعاءات الصادرة إلى خوادم أن تقبل Context. ويجب أن تنشر سلسلة استدعاءات الدوال بينهما الـ Context، مع استبداله اختياريًا بـ Context مشتق يُنشأ باستخدام WithCancel أو WithDeadline أو WithTimeout أو WithValue. وعندما يُلغى Context، تُلغى أيضًا كل الـ Contexts المشتقة منه.

ومن [مدونة Go: الـ Context](https://blog.golang.org/context) مرة أخرى:

> في Google، نطلب من مبرمجي Go تمرير بارامتر Context كأول وسيط في كل دالة على مسار الاستدعاء بين الطلبات الواردة والصادرة. وهذا يتيح لكود Go الذي طورته فرق مختلفة كثيرة أن يتكامل جيدًا. كما يوفر تحكمًا بسيطًا في المهلات والإلغاء، ويضمن أن القيم الحرجة مثل بيانات الاعتماد الأمنية تنتقل داخل برامج Go كما ينبغي.

(توقف لحظة وفكّر في العواقب المترتبة على اضطرار كل دالة إلى تمرير context، وفي مدى راحتها في الاستخدام.)

هل تشعر ببعض القلق؟ جيد. لكن لنحاول اتباع تلك المقاربة، بأن نمرر `context` إلى `Store` ونجعله هو المسؤول. وبذلك يستطيع أيضًا تمرير `context` إلى ما يعتمد عليه، فيكون كل منهم مسؤولًا عن إيقاف نفسه.

## اكتب الاختبار أولًا

سيتعين علينا تغيير اختباراتنا الحالية لأن مسؤولياتها تتغير. فكل ما يتحمله المعالج مسؤولية الآن هو التأكد من تمرير context إلى `Store` في الأسفل، وأنه يتعامل مع الخطأ الذي سيأتي من `Store` عند إلغائه.

لنحدّث واجهة `Store` لنعرض المسؤوليات الجديدة.

```go
type Store interface {
	Fetch(ctx context.Context) (string, error)
}
```

احذف الكود الموجود داخل المعالج في الوقت الحالي

```go
func Server(store Store) http.HandlerFunc {
	return func(w http.ResponseWriter, r *http.Request) {
	}
}
```

حدّث `SpyStore` الخاص بنا

```go
type SpyStore struct {
	response string
	t        *testing.T
}

func (s *SpyStore) Fetch(ctx context.Context) (string, error) {
	data := make(chan string, 1)

	go func() {
		var result string
		for _, c := range s.response {
			select {
			case <-ctx.Done():
				log.Println("spy store got cancelled")
				return
			default:
				time.Sleep(10 * time.Millisecond)
				result += string(c)
			}
		}
		data <- result
	}()

	select {
	case <-ctx.Done():
		return "", ctx.Err()
	case res := <-data:
		return res, nil
	}
}
```

علينا أن نجعل الـ spy يتصرف كـ method حقيقية تعمل مع `context`.

فنحن نحاكي عملية بطيئة نبني فيها النتيجة ببطء بإلحاق النص حرفًا حرفًا داخل goroutine. وعندما تنتهي الـ goroutine من عملها، تكتب النص في channel اسمها `data`. وتستمع الـ goroutine إلى `ctx.Done` وتوقف العمل إذا أُرسلت إشارة في ذلك الـ channel.

وأخيرًا يستخدم الكود `select` أخرى لانتظار انتهاء تلك الـ goroutine من عملها أو لحدوث الإلغاء.

وهي مقاربة مشابهة لما فعلناه سابقًا؛ إذ نستخدم عناصر الـ concurrency الأولية في Go لجعل عمليتين غير متزامنتين تتسابقان لتحديد ما نُرجعه.

وستتبع مقاربة مشابهة عندما تكتب دوالك وmethods الخاصة بك التي تقبل `context`، لذا تأكد من أنك تفهم ما يجري.

وأخيرًا يمكننا تحديث اختباراتنا. علّق اختبار الإلغاء لدينا حتى نصلح اختبار المسار السعيد أولًا.

```go
t.Run("returns data from store", func(t *testing.T) {
	data := "hello, world"
	store := &SpyStore{response: data, t: t}
	svr := Server(store)

	request := httptest.NewRequest(http.MethodGet, "/", nil)
	response := httptest.NewRecorder()

	svr.ServeHTTP(response, request)

	if response.Body.String() != data {
		t.Errorf(`got "%s", want "%s"`, response.Body.String(), data)
	}
})
```

## جرّب تشغيل الاختبار

```
=== RUN   TestServer/returns_data_from_store
--- FAIL: TestServer (0.00s)
    --- FAIL: TestServer/returns_data_from_store (0.00s)
    	context_test.go:22: got "", want "hello, world"
```

## اكتب كودًا كافيًا لنجاح الاختبار

```go
func Server(store Store) http.HandlerFunc {
	return func(w http.ResponseWriter, r *http.Request) {
		data, _ := store.Fetch(r.Context())
		fmt.Fprint(w, data)
	}
}
```

من المفترض أن يكون مسارنا السعيد... سعيدًا. ويمكننا الآن إصلاح الاختبار الآخر.

## اكتب الاختبار أولًا

نحتاج إلى اختبار أننا لا نكتب أي نوع من الاستجابة في حالة الخطأ. وللأسف لا يملك `httptest.ResponseRecorder` طريقة لاكتشاف ذلك، لذا سيتعين علينا صنع spy خاص بنا لاختبار هذا.

```go
type SpyResponseWriter struct {
	written bool
}

func (s *SpyResponseWriter) Header() http.Header {
	s.written = true
	return nil
}

func (s *SpyResponseWriter) Write([]byte) (int, error) {
	s.written = true
	return 0, errors.New("not implemented")
}

func (s *SpyResponseWriter) WriteHeader(statusCode int) {
	s.written = true
}
```

يحقق `SpyResponseWriter` الخاص بنا واجهة `http.ResponseWriter`، لذا يمكننا استخدامه في الاختبار.

```go
t.Run("tells store to cancel work if request is cancelled", func(t *testing.T) {
	data := "hello, world"
	store := &SpyStore{response: data, t: t}
	svr := Server(store)

	request := httptest.NewRequest(http.MethodGet, "/", nil)

	cancellingCtx, cancel := context.WithCancel(request.Context())
	time.AfterFunc(5*time.Millisecond, cancel)
	request = request.WithContext(cancellingCtx)

	response := &SpyResponseWriter{}

	svr.ServeHTTP(response, request)

	if response.written {
		t.Error("a response should not have been written")
	}
})
```

## جرّب تشغيل الاختبار

```
=== RUN   TestServer
=== RUN   TestServer/tells_store_to_cancel_work_if_request_is_cancelled
--- FAIL: TestServer (0.01s)
    --- FAIL: TestServer/tells_store_to_cancel_work_if_request_is_cancelled (0.01s)
    	context_test.go:47: a response should not have been written
```

## اكتب كودًا كافيًا لنجاح الاختبار

```go
func Server(store Store) http.HandlerFunc {
	return func(w http.ResponseWriter, r *http.Request) {
		data, err := store.Fetch(r.Context())

		if err != nil {
			return // todo: log error however you like
		}

		fmt.Fprint(w, data)
	}
}
```

نرى بعد ذلك أن كود الخادم صار أبسط، إذ لم يعد مسؤولًا صراحةً عن الإلغاء، بل يمرر `context` فحسب ويعتمد على الدوال في الأسفل لتحترم أي إلغاء قد يحدث.

## الخلاصة

### ما غطّيناه

- كيف تختبر معالج HTTP أُلغي طلبه من العميل.
- كيف تستخدم context لإدارة الإلغاء.
- كيف تكتب دالة تقبل `context` وتستخدمه لإلغاء نفسها بالاستعانة بالـ goroutines و`select` والـ channels.
- اتباع إرشادات Google في إدارة الإلغاء بنشر context المرتبط بالطلب عبر مكدس الاستدعاءات.
- كيف تصنع spy خاصًا بك لـ `http.ResponseWriter` إذا احتجت إليه.

### ماذا عن context.Value؟

[Michal Štrba](https://faiface.github.io/post/context-should-go-away-go2/) وأنا نتفق في رأي مشابه.

> إذا استخدمت ctx.Value في شركتي (غير الموجودة)، فأنت مفصول من عملك

دعا بعض المهندسين إلى تمرير القيم عبر `context` لأنه _يبدو مريحًا_.

والراحة غالبًا ما تكون سببًا لكود سيئ.

مشكلة `context.Values` أنها مجرد map بلا أنواع، فلا تملك أمان الأنواع (type-safety) وعليك أن تتعامل مع احتمال أنها لا تحتوي قيمتك فعلًا. وسيتعين عليك إنشاء ارتباط بين مفاتيح الـ map من وحدة إلى أخرى، وإذا غيّر أحدهم شيئًا بدأت الأمور تنكسر.

وباختصار، **إذا احتاجت دالة إلى بعض القيم، فضعها كوسائط محدَّدة الأنواع بدلًا من محاولة جلبها من `context.Value`**. فهذا يجعلها مفحوصة ساكنًا (statically checked) وموثقة ليراها الجميع.

#### لكن...

ومن جهة أخرى، قد يكون من المفيد تضمين معلومات متعامدة (orthogonal) مع الطلب في الـ context، مثل trace id. وربما لن تحتاج كل دالة في مكدس الاستدعاءات إلى هذه المعلومات، وستجعل توقيعات دوالك في غاية الفوضى.

[يقول Jack Lindamood إن **Context.Value ينبغي أن يُعلم، لا أن يتحكم**](https://medium.com/@cep21/how-to-correctly-use-context-context-in-go-1-7-8f2c0fafdf39)

> محتوى context.Value مخصص للقائمين على الصيانة لا للمستخدمين. وينبغي ألا يكون أبدًا مدخلًا مطلوبًا لنتائج موثقة أو متوقعة.

### مواد إضافية

- لقد استمتعت حقًا بقراءة [Context should go away for Go 2 بقلم Michal Štrba](https://faiface.github.io/post/context-should-go-away-go2/). وحجته أن الاضطرار إلى تمرير `context` في كل مكان رائحة كود (smell) تشير إلى قصور في اللغة فيما يتعلق بالإلغاء. ويقول إنه سيكون أفضل لو حُلّت هذه المشكلة بطريقة ما على مستوى اللغة، لا على مستوى المكتبة. وإلى أن يحدث ذلك، ستحتاج إلى `context` إذا أردت إدارة العمليات طويلة الأمد.
- تصف [مدونة Go الدافع وراء العمل بـ `context` بمزيد من التفصيل وتقدّم بعض الأمثلة](https://blog.golang.org/context)
