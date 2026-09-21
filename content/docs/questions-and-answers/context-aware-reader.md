---
title: قارئ يراعي الـ Context
weight: 360
---

# قارئ يراعي الـ Context

**[يمكنك العثور على كل كود هذا الفصل هنا](https://github.com/quii/learn-go-with-tests/tree/main/q-and-a/context-aware-reader)**

يوضح هذا الفصل كيف نطوّر بالاختبار أولًا `io.Reader` يراعي الـ context، كما كتبه Mat Ryer وDavid Hernandez في [مدونة The Pace Dev](https://pace.dev/blog/2020/02/03/context-aware-ioreader-for-golang-by-mat-ryer).

## قارئ يراعي الـ context؟

أولًا، مقدمة سريعة عن `io.Reader`.

إذا كنت قد قرأت فصولًا أخرى في هذا الكتاب فستكون قد صادفت `io.Reader` عند فتح الملفات وترميز JSON ومهام شائعة أخرى متنوعة. إنه تجريد بسيط لقراءة البيانات من _شيء ما_

```go
type Reader interface {
	Read(p []byte) (n int, err error)
}
```

باستخدام `io.Reader` يمكنك تحقيق قدر كبير من إعادة الاستخدام من المكتبة القياسية، فهو تجريد شائع الاستخدام جدًا (إلى جانب نظيره `io.Writer`)

### مراعاة الـ context؟

في [فصل سابق](../go-fundamentals/context.md) ناقشنا كيف يمكننا استخدام `context` لتوفير الإلغاء. وهذا مفيد بشكل خاص إذا كنت تنفذ مهام قد تكون مكلفة حسابيًا وتريد أن تكون قادرًا على إيقافها.

عندما تستخدم `io.Reader` فلا ضمانات لديك بشأن السرعة؛ فقد يستغرق نانوثانية واحدة أو مئات الساعات. وقد تجد أنه من المفيد أن تكون قادرًا على إلغاء هذا النوع من المهام في تطبيقك، وهذا ما كتب عنه Mat وDavid.

دمجا بين تجريدين بسيطين (`context.Context` و`io.Reader`) لحل هذه المشكلة.

لنجرّب تطوير بعض الوظائف بالاختبار أولًا حتى نستطيع لفّ `io.Reader` ليصبح قابلًا للإلغاء.

اختبار هذا يمثل تحديًا مثيرًا للاهتمام. فعادةً عندما تستخدم `io.Reader` تمرره إلى دالة أخرى ولا تشغل نفسك بالتفاصيل؛ مثل `json.NewDecoder` أو `io.ReadAll`.

ما نريد توضيحه هو شيء كهذا

> إذا كان لدينا `io.Reader` يحتوي على "ABCDEF"، وأرسلت إشارة إلغاء في منتصف الطريق، فعندما أحاول مواصلة القراءة لا أحصل على شيء آخر، وبذلك لا أحصل إلا على "ABC"

لننظر إلى الواجهة مرة أخرى.

```go
type Reader interface {
	Read(p []byte) (n int, err error)
}
```

ستقرأ دالة (method) `Read` الخاصة بـ `Reader` المحتوى الموجود لديها إلى `[]byte` نمرره نحن.

فبدلًا من قراءة كل شيء، يمكننا:

 - تمرير مصفوفة بايتات بحجم ثابت لا تتسع لكل المحتوى
 - إرسال إشارة إلغاء
 - المحاولة والقراءة مرة أخرى، وينبغي أن يُرجع ذلك خطأ مع قراءة 0 بايت

في الوقت الحالي، لنكتب اختبار "المسار السعيد" فقط حيث لا يوجد إلغاء، حتى نتعرف على المشكلة دون أن نضطر إلى كتابة أي كود إنتاج بعد.

```go
func TestContextAwareReader(t *testing.T) {
	t.Run("lets just see how a normal reader works", func(t *testing.T) {
		rdr := strings.NewReader("123456")
		got := make([]byte, 3)
		_, err := rdr.Read(got)

		if err != nil {
			t.Fatal(err)
		}

		assertBufferHas(t, got, "123")

		_, err = rdr.Read(got)

		if err != nil {
			t.Fatal(err)
		}

		assertBufferHas(t, got, "456")
	})
}

func assertBufferHas(t testing.TB, buf []byte, want string) {
	t.Helper()
	got := string(buf)
	if got != want {
		t.Errorf("got %q, want %q", got, want)
	}
}
```

- أنشئ `io.Reader` من نص يحتوي على بعض البيانات
- مصفوفة بايتات نقرأ إليها وهي أصغر من محتوى القارئ
- استدعِ القراءة، وافحص المحتوى، وكرّر.

ومن هنا يمكننا أن نتخيل إرسال نوع ما من إشارة إلغاء قبل القراءة الثانية لتغيير السلوك.

وبعد أن رأينا كيف يعمل ذلك، سنطوّر بقية الوظائف بالاختبار أولًا.

## اكتب الاختبار أولًا

نريد أن نكون قادرين على تركيب `io.Reader` مع `context.Context`.

مع التطوير الموجه بالاختبار (TDD)، من الأفضل أن تبدأ بتخيل الـ API الذي تريده وتكتب اختبارًا له.

ومن هناك، دع المترجم ومخرجات الاختبار الفاشل يرشدانك إلى الحل

```go
t.Run("behaves like a normal reader", func(t *testing.T) {
	rdr := NewCancellableReader(strings.NewReader("123456"))
	got := make([]byte, 3)
	_, err := rdr.Read(got)

	if err != nil {
		t.Fatal(err)
	}

	assertBufferHas(t, got, "123")

	_, err = rdr.Read(got)

	if err != nil {
		t.Fatal(err)
	}

	assertBufferHas(t, got, "456")
})
```

## جرّب تشغيل الاختبار

```
./cancel_readers_test.go:12:10: undefined: NewCancellableReader
```
## اكتب أصغر قدر من الكود ليعمل الاختبار وافحص مخرجاته الفاشلة

سنحتاج إلى تعريف هذه الدالة، وينبغي أن تُرجع `io.Reader`

```go
func NewCancellableReader(rdr io.Reader) io.Reader {
	return nil
}
```

ولو جرّبت تشغيله

```
=== RUN   TestCancelReaders
=== RUN   TestCancelReaders/behaves_like_a_normal_reader
panic: runtime error: invalid memory address or nil pointer dereference [recovered]
	panic: runtime error: invalid memory address or nil pointer dereference
[signal SIGSEGV: segmentation violation code=0x1 addr=0x0 pc=0x10f8fb5]
```

كما هو متوقع

## اكتب كودًا كافيًا لنجاح الاختبار

في الوقت الحالي، سنُرجع فقط `io.Reader` الذي نمرره

```go
func NewCancellableReader(rdr io.Reader) io.Reader {
	return rdr
}
```

ينبغي أن ينجح الاختبار الآن.

أعلم، أعلم، يبدو هذا سخيفًا ومفرطًا في التدقيق، لكن قبل الاندفاع إلى العمل الفاخر، من المهم أن تكون لدينا _بعض_ وسائل التحقق من أننا لم نكسر السلوك "العادي" لـ `io.Reader`، وهذا الاختبار سيمنحنا الثقة أثناء تقدمنا.

## اكتب الاختبار أولًا

بعد ذلك نحتاج إلى تجربة الإلغاء.

```go
t.Run("stops reading when cancelled", func(t *testing.T) {
	ctx, cancel := context.WithCancel(context.Background())
	rdr := NewCancellableReader(ctx, strings.NewReader("123456"))
	got := make([]byte, 3)
	_, err := rdr.Read(got)

	if err != nil {
		t.Fatal(err)
	}

	assertBufferHas(t, got, "123")

	cancel()

	n, err := rdr.Read(got)

	if err == nil {
		t.Error("expected an error after cancellation but didn't get one")
	}

	if n > 0 {
		t.Errorf("expected 0 bytes to be read after cancellation but %d were read", n)
	}
})
```

يمكننا تقريبًا نسخ الاختبار الأول، لكننا الآن:
- ننشئ `context.Context` مع إلغاء حتى نستطيع `cancel` بعد القراءة الأولى
- ولكي يعمل كودنا سنحتاج إلى تمرير `ctx` إلى دالتنا
- ثم نتحقق أنه بعد `cancel` لم تُقرأ أي بيانات

## جرّب تشغيل الاختبار

```
./cancel_readers_test.go:33:30: too many arguments in call to NewCancellableReader
	have (context.Context, *strings.Reader)
	want (io.Reader)
```

## اكتب أصغر قدر من الكود ليعمل الاختبار وافحص مخرجاته الفاشلة

المترجم يخبرنا بما يجب فعله؛ لنحدّث التوقيع ليستقبل context

```go
func NewCancellableReader(ctx context.Context, rdr io.Reader) io.Reader {
	return rdr
}
```

(ستحتاج إلى تحديث الاختبار الأول ليمرر `context.Background` أيضًا)

ينبغي أن ترى الآن مخرجات اختبار فاشل واضحة جدًا

```
=== RUN   TestCancelReaders
=== RUN   TestCancelReaders/stops_reading_when_cancelled
--- FAIL: TestCancelReaders (0.00s)
    --- FAIL: TestCancelReaders/stops_reading_when_cancelled (0.00s)
        cancel_readers_test.go:48: expected an error but didn't get one
        cancel_readers_test.go:52: expected 0 bytes to be read after cancellation but 3 were read
```

## اكتب كودًا كافيًا لنجاح الاختبار

في هذه المرحلة، الأمر نسخ ولصق من المقال الأصلي لـ Mat وDavid، لكننا سنلتزم بالتدرج خطوة بخطوة.

نعلم أننا نحتاج إلى نوع يغلّف `io.Reader` الذي نقرأ منه و`context.Context`، فلننشئ ذلك النوع ونحاول إرجاعه من دالتنا بدلًا من `io.Reader` الأصلي

```go
func NewCancellableReader(ctx context.Context, rdr io.Reader) io.Reader {
	return &readerCtx{
		ctx:      ctx,
		delegate: rdr,
	}
}

type readerCtx struct {
	ctx      context.Context
	delegate io.Reader
}
```

وكما أكدت مرارًا في هذا الكتاب، امضِ ببطء ودع المترجم يساعدك

```
./cancel_readers_test.go:60:3: cannot use &readerCtx literal (type *readerCtx) as type io.Reader in return argument:
	*readerCtx does not implement io.Reader (missing Read method)
```

يبدو التجريد مناسبًا، لكنه لا يحقق الواجهة التي نحتاجها (`io.Reader`)، فلنضف الـ method.

```go
func (r *readerCtx) Read(p []byte) (n int, err error) {
	panic("implement me")
}
```

شغّل الاختبارات، وينبغي أن _تُترجم_ لكنها ستؤدي إلى panic. وما زال هذا تقدمًا.

لنجعل الاختبار الأول ينجح بمجرد _تفويض_ الاستدعاء إلى `io.Reader` الأساسي لدينا

```go
func (r readerCtx) Read(p []byte) (n int, err error) {
	return r.delegate.Read(p)
}
```

في هذه المرحلة أصبح اختبار المسار السعيد ينجح مرة أخرى، ويبدو أننا جردنا أمورنا بشكل جيد

لنجعل اختبارنا الثاني ينجح، نحتاج إلى فحص `context.Context` لمعرفة ما إذا كان قد أُلغيت.

```go
func (r readerCtx) Read(p []byte) (n int, err error) {
	if err := r.ctx.Err(); err != nil {
		return 0, err
	}
	return r.delegate.Read(p)
}
```

ينبغي أن تنجح كل الاختبارات الآن. وستلاحظ كيف نُرجع الخطأ من `context.Context`. وهذا يتيح لمستدعي الكود فحص الأسباب المختلفة لحدوث الإلغاء، وقد غُطي هذا بمزيد من التفصيل في المقال الأصلي.

## الخلاصة

- الواجهات الصغيرة جيدة ويسهل تركيبها
- عندما تحاول تعزيز شيء ما (مثل `io.Reader`) بشيء آخر، فعادةً ما تريد اللجوء إلى [نمط التفويض (delegation pattern)](https://en.wikipedia.org/wiki/Delegation_pattern)

> في هندسة البرمجيات، نمط التفويض هو نمط تصميم كائني التوجه يتيح تركيب الكائنات لتحقيق إعادة استخدام الكود نفسها التي تحققها الوراثة.

- من الطرق السهلة لبدء هذا النوع من العمل أن تلفّ المفوَّض (delegate) لديك وتكتب اختبارًا يتحقق من أنه يتصرف كما يتصرف المفوَّض عادةً، قبل أن تبدأ بتركيب أجزاء أخرى لتغيير السلوك. وسيساعدك هذا في إبقاء الأمور تعمل بشكل صحيح بينما تكتب الكود نحو هدفك
