---
title: أنواع الأخطاء
weight: 350
---

# أنواع الأخطاء

**[يمكنك العثور على كل كود هذا الفصل هنا](https://github.com/quii/learn-go-with-tests/tree/main/q-and-a/error-types)**

**إنشاء أنواع خاصة بك للأخطاء يمكن أن يكون طريقة أنيقة لترتيب كودك، وتجعل كودك أسهل استخدامًا واختبارًا.**

يسأل Pedro على Gopher Slack

> إذا كنت أنشئ خطأ مثل `fmt.Errorf("%s must be foo, got %s", bar, baz)`، فهل هناك طريقة لاختبار التساوي دون مقارنة قيمة النص؟

لنصنع دالة تساعدنا على استكشاف هذه الفكرة.

```go
// DumbGetter will get the string body of url if it gets a 200
func DumbGetter(url string) (string, error) {
	res, err := http.Get(url)

	if err != nil {
		return "", fmt.Errorf("problem fetching from %s, %v", url, err)
	}

	if res.StatusCode != http.StatusOK {
		return "", fmt.Errorf("did not get 200 from %s, got %d", url, res.StatusCode)
	}

	defer res.Body.Close()
	body, _ := io.ReadAll(res.Body) // ignoring err for brevity

	return string(body), nil
}
```

ليس نادرًا أن تكتب دالة قد تفشل لأسباب مختلفة، ونريد أن نتأكد أننا نتعامل مع كل سيناريو بشكل صحيح.

وكما يقول Pedro، _يمكننا_ كتابة اختبار لخطأ الحالة (status error) هكذا.

```go
t.Run("when you don't get a 200 you get a status error", func(t *testing.T) {

	svr := httptest.NewServer(http.HandlerFunc(func(res http.ResponseWriter, req *http.Request) {
		res.WriteHeader(http.StatusTeapot)
	}))
	defer svr.Close()

	_, err := DumbGetter(svr.URL)

	if err == nil {
		t.Fatal("expected an error")
	}

	want := fmt.Sprintf("did not get 200 from %s, got %d", svr.URL, http.StatusTeapot)
	got := err.Error()

	if got != want {
		t.Errorf(`got "%v", want "%v"`, got, want)
	}
})
```

ينشئ هذا الاختبار خادمًا يُرجع دائمًا `StatusTeapot`، ثم نستخدم رابطه (URL) كوسيط لدالة `DumbGetter` لنرى أنه يتعامل بشكل صحيح مع ردود ليست `200`.

## مشكلات هذه الطريقة في الاختبار

يحاول هذا الكتاب التأكيد على _أنصت إلى اختباراتك_، وهذا الاختبار لا _يبدو_ جيدًا:

- نحن نبني نفس النص الذي يبنيه كود الإنتاج لاختباره
- من المزعج قراءته وكتابته
- هل نص رسالة الخطأ الدقيق هو ما _يهمنا فعلًا_؟

ماذا يخبرنا هذا؟ إن سهولة استخدام اختبارنا ستنعكس على أي جزء آخر من الكود يحاول استخدام كودنا.

كيف يتفاعل مستخدم كودنا مع النوع المحدد من الأخطاء التي نُرجعها؟ أفضل ما يستطيعه هو النظر إلى نص الخطأ، وهو شديد العرضة للخطأ ومزعج في الكتابة.

## ما ينبغي أن نفعله

مع التطوير الموجه بالاختبار (TDD) لدينا ميزة الدخول في عقلية:

> كيف _أريد_ أنا استخدام هذا الكود؟

ما يمكننا فعله من أجل `DumbGetter` هو توفير طريقة يستخدم بها المستخدمون نظام الأنواع (type system) لمعرفة نوع الخطأ الذي حدث.

ماذا لو كان بإمكان `DumbGetter` أن يُرجع لنا شيئًا مثل

```go
type BadStatusError struct {
	URL    string
	Status int
}
```

فبدلًا من نص سحري، لدينا _بيانات_ حقيقية نعمل عليها.

لنغيّر اختبارنا الحالي ليعكس هذه الحاجة

```go
t.Run("when you don't get a 200 you get a status error", func(t *testing.T) {

	svr := httptest.NewServer(http.HandlerFunc(func(res http.ResponseWriter, req *http.Request) {
		res.WriteHeader(http.StatusTeapot)
	}))
	defer svr.Close()

	_, err := DumbGetter(svr.URL)

	if err == nil {
		t.Fatal("expected an error")
	}

	got, isStatusErr := err.(BadStatusError)

	if !isStatusErr {
		t.Fatalf("was not a BadStatusError, got %T", err)
	}

	want := BadStatusError{URL: svr.URL, Status: http.StatusTeapot}

	if got != want {
		t.Errorf("got %v, want %v", got, want)
	}
})
```

سيتعين علينا أن نجعل `BadStatusError` يحقق واجهة الخطأ (error interface).

```go
func (b BadStatusError) Error() string {
	return fmt.Sprintf("did not get 200 from %s, got %d", b.URL, b.Status)
}
```

### ماذا يفعل الاختبار؟

فبدلًا من فحص النص الدقيق للخطأ، نقوم بـ [توكيد النوع (type assertion)](https://tour.golang.org/methods/15) على الخطأ لنرى إن كان `BadStatusError`. وهذا يعكس رغبتنا في أن يكون _نوع_ الخطأ أوضح. وبافتراض أن التوكيد ينجح، يمكننا بعدها التحقق أن خصائص الخطأ صحيحة.

عندما نشغّل الاختبار، يخبرنا أننا لم نُرجع نوع الخطأ الصحيح

```
--- FAIL: TestDumbGetter (0.00s)
    --- FAIL: TestDumbGetter/when_you_dont_get_a_200_you_get_a_status_error (0.00s)
    	error-types_test.go:56: was not a BadStatusError, got *errors.errorString
```

لنصلح `DumbGetter` بتحديث كود معالجة الأخطاء لدينا ليستخدم نوعنا

```go
if res.StatusCode != http.StatusOK {
	return "", BadStatusError{URL: url, Status: res.StatusCode}
}
```

كان لهذا التغيير بعض _الآثار الإيجابية الحقيقية_

- أصبحت دالة `DumbGetter` أبسط، فلم تعد معنية بتفاصيل نص الخطأ، بل تُنشئ فقط `BadStatusError`.
- تعكس اختباراتنا الآن (وتوثّق) ما _يستطيع_ مستخدم كودنا فعله إذا قرر القيام بمعالجة أخطاء أكثر تطورًا من مجرد التسجيل (logging). يكفيه أن يقوم بتوكيد نوع، فيحصل على وصول سهل إلى خصائص الخطأ.
- فهو لا يزال "مجرد" `error`، لذا إن اختاروا يمكنهم تمريره عبر مكدس الاستدعاءات (call stack) أو تسجيله مثل أي `error` آخر.

## الخلاصة

إذا وجدت نفسك تختبر حالات خطأ متعددة، فلا تقع في فخ مقارنة رسائل الخطأ.

فهذا يؤدي إلى اختبارات هشة وصعبة القراءة والكتابة، ويعكس الصعوبات التي سيواجهها مستخدمو كودك إذا احتاجوا هم أيضًا إلى التصرف بشكل مختلف حسب نوع الأخطاء التي حدثت.

احرص دائمًا أن تعكس اختباراتك كيف _تحب_ أنت استخدام كودك، ومن هذه الزاوية فكّر في إنشاء أنواع للأخطاء لتغليف أنواع أخطائك. فهذا يجعل التعامل مع الأنواع المختلفة من الأخطاء أسهل لمستخدمي كودك، ويجعل كتابة كود معالجة الأخطاء أبسط وأسهل قراءة.

## ملحق

بدءًا من Go 1.13 ظهرت طرق جديدة للتعامل مع الأخطاء في المكتبة القياسية، وقد غُطّيت في [مدونة Go](https://blog.golang.org/go1.13-errors)

```go
t.Run("when you don't get a 200 you get a status error", func(t *testing.T) {

	svr := httptest.NewServer(http.HandlerFunc(func(res http.ResponseWriter, req *http.Request) {
		res.WriteHeader(http.StatusTeapot)
	}))
	defer svr.Close()

	_, err := DumbGetter(svr.URL)

	if err == nil {
		t.Fatal("expected an error")
	}

	var got BadStatusError
	isBadStatusError := errors.As(err, &got)
	want := BadStatusError{URL: svr.URL, Status: http.StatusTeapot}

	if !isBadStatusError {
		t.Fatalf("was not a BadStatusError, got %T", err)
	}

	if got != want {
		t.Errorf("got %v, want %v", got, want)
	}
})
```

في هذه الحالة نستخدم [`errors.As`](https://pkg.go.dev/errors#example-As) لمحاولة استخراج خطأنا إلى نوعنا المخصص. وهي تُرجع قيمة `bool` تدل على النجاح، وتستخرج الخطأ لنا في `got`.
