---
title: الـ Mocking
weight: 100
---

# الـ Mocking

**[يمكنك العثور على كل كود هذا الفصل هنا](https://github.com/quii/learn-go-with-tests/tree/main/mocking)**

طُلب منك كتابة برنامج يعدّ تنازليًا من 3، ويطبع كل رقم في سطر جديد (مع توقف لمدة ثانية واحدة)، وعندما يصل إلى الصفر يطبع "Go!" ويخرج.

```
3
2
1
Go!
```

سنعالج هذا بكتابة دالة اسمها `Countdown` ثم نضعها داخل برنامج `main` ليصبح الأمر هكذا:

```go
package main

func main() {
	Countdown()
}
```

ومع أن هذا برنامج بسيط للغاية، فإن اختباره اختبارًا كاملًا سيقتضي منا كما في كل مرة اتباع منهج _تكراري_ و_موجه بالاختبار_.

وما أعنيه بالتكراري؟ أن نحرص على أن نخطو أصغر الخطوات الممكنة للحصول على _برمجية مفيدة_.

لا نريد أن نقضي وقتًا طويلًا مع كود سيعمل نظريًا بعد بعض العبث، فهذه غالبًا هي الطريقة التي يسقط بها المطورون في جحور الأرانب. **ومن المهارات المهمة أن تجزّئ المتطلبات إلى أصغر أجزاء ممكنة لتحصل على _برمجية تعمل_.**

وإليك كيف يمكننا تقسيم عملنا والتقدم فيه:

* اطبع 3
* اطبع 3 و2 و1 وGo!
* انتظر ثانية بين كل سطر

## اكتب الاختبار أولًا

برمجيتنا تحتاج إلى الكتابة إلى stdout، وقد رأينا كيف يمكننا استخدام حقن الاعتماديات (dependency injection) لتسهيل اختبار ذلك في قسم حقن الاعتماديات.

```go
func TestCountdown(t *testing.T) {
	buffer := &bytes.Buffer{}

	Countdown(buffer)

	got := buffer.String()
	want := "3"

	if got != want {
		t.Errorf("got %q want %q", got, want)
	}
}
```

إذا كان شيء مثل `buffer` غير مألوف لك، فأعد قراءة [القسم السابق](dependency-injection.md).

نحن نعلم أننا نريد لدالة `Countdown` أن تكتب البيانات في مكان ما، وتُعد `io.Writer` الطريقة الفعلية المعتمدة للتعبير عن ذلك كواجهة (interface) في Go.

* في `main` سنرسل المخرجات إلى `os.Stdout` ليرى مستخدمونا العد التنازلي مطبوعًا في الطرفية.
* وفي الاختبار سنرسل إلى `bytes.Buffer` لتتمكن اختباراتنا من التقاط البيانات التي تُنتَج.

## جرّب تشغيل الاختبار

`./countdown_test.go:11:2: undefined: Countdown`

## اكتب أصغر قدر من الكود ليعمل الاختبار وافحص مخرجاته الفاشلة

عرّف `Countdown`

```go
func Countdown() {}
```

جرّب مرة أخرى

```
./countdown_test.go:11:11: too many arguments in call to Countdown
    have (*bytes.Buffer)
    want ()
```

يخبرك المترجم بما يمكن أن يكون عليه توقيع دالتك، فحدّثه.

```go
func Countdown(out *bytes.Buffer) {}
```

`countdown_test.go:17: got '' want '3'`

ممتاز!

## اكتب كودًا كافيًا لنجاح الاختبار

```go
func Countdown(out *bytes.Buffer) {
	fmt.Fprint(out, "3")
}
```

نستخدم `fmt.Fprint` التي تأخذ `io.Writer` (مثل `*bytes.Buffer`) وترسل إليه `string`. ومن المفترض أن ينجح الاختبار.

## إعادة الهيكلة

نعلم أنه مع أن `*bytes.Buffer` يعمل، فسيكون من الأفضل استخدام واجهة عامة الغرض بدلًا منه.

```go
func Countdown(out io.Writer) {
	fmt.Fprint(out, "3")
}
```

أعد تشغيل الاختبارات، ومن المفترض أن تنجح.

ولإكمال الأمور، لنربط الآن دالتنا ببرنامج `main` ليكون لدينا برنامج يعمل يطمئننا أننا نحرز تقدمًا.

```go
package main

import (
	"fmt"
	"io"
	"os"
)

func Countdown(out io.Writer) {
	fmt.Fprint(out, "3")
}

func main() {
	Countdown(os.Stdout)
}
```

جرّب تشغيل البرنامج، وستُعجب بما أنجزته يداك.

نعم يبدو هذا بسيطًا، لكن هذا هو الأسلوب الذي أوصي به في أي مشروع. **خُذ شريحة رفيعة من الوظائف واجعلها تعمل من البداية إلى النهاية، مدعومة بالاختبارات.**

بعد ذلك يمكننا جعله يطبع 2 و1 ثم "Go!".

## اكتب الاختبار أولًا

بالاستثمار في إتقان السباكة العامة، يمكننا التقدم في حلنا بأمان وسهولة. فلن نحتاج بعد الآن إلى التوقف وإعادة تشغيل البرنامج للتأكد من عمله، لأن كل المنطق مُختبر.

```go
func TestCountdown(t *testing.T) {
	buffer := &bytes.Buffer{}

	Countdown(buffer)

	got := buffer.String()
	want := `3
2
1
Go!`

	if got != want {
		t.Errorf("got %q want %q", got, want)
	}
}
```

صياغة backtick طريقة أخرى لإنشاء `string` تتيح لك تضمين أشياء مثل الأسطر الجديدة، وهو أمر مثالي لاختبارنا.

## جرّب تشغيل الاختبار

```
countdown_test.go:21: got '3' want '3
        2
        1
        Go!'
```

## اكتب كودًا كافيًا لنجاح الاختبار

```go
func Countdown(out io.Writer) {
	for i := 3; i > 0; i-- {
		fmt.Fprintln(out, i)
	}
	fmt.Fprint(out, "Go!")
}
```

استخدم حلقة `for` تعد تنازليًا بـ `i--`، واستخدم `fmt.Fprintln` للطباعة إلى `out` مع الرقم متبوعًا بمحرف سطر جديد. وأخيرًا استخدم `fmt.Fprint` لإرسال "Go!" بعد ذلك.

## إعادة الهيكلة

لا يوجد الكثير لنعيد هيكلته سوى تحويل بعض القيم السحرية إلى ثوابت مُسمّاة.

```go
const finalWord = "Go!"
const countdownStart = 3

func Countdown(out io.Writer) {
	for i := countdownStart; i > 0; i-- {
		fmt.Fprintln(out, i)
	}
	fmt.Fprint(out, finalWord)
}
```

إذا شغّلت البرنامج الآن، فمن المفترض أن تحصل على المخرجات المطلوبة، لكن ليس لدينا عدّ تنازلي درامي بتوقفات الثانية الواحدة.

تتيح لك Go تحقيق ذلك بـ `time.Sleep`. جرّب إضافتها إلى كودنا.

```go
func Countdown(out io.Writer) {
	for i := countdownStart; i > 0; i-- {
		fmt.Fprintln(out, i)
		time.Sleep(1 * time.Second)
	}

	fmt.Fprint(out, finalWord)
}
```

إذا شغّلت البرنامج فسيعمل كما نريد.

## الـ Mocking

ما زالت الاختبارات تنجح والبرنامج يعمل كما هو مقصود، لكن لدينا بعض المشكلات:

* تستغرق اختباراتنا 3 ثوانٍ لتعمل.
    * كل مقال بعيد النظر عن تطوير البرمجيات يؤكد أهمية حلقات التغذية الراجعة السريعة.
    * **الاختبارات البطيئة تدمّر إنتاجية المطور**.
    * تخيّل لو أصبحت المتطلبات أكثر تعقيدًا فاحتاجت اختبارات أكثر. هل نرضى بأن تُضاف 3 ثوانٍ إلى زمن تشغيل الاختبارات مع كل اختبار جديد لـ `Countdown`؟
* لم نختبر خاصية مهمة في دالتنا.

لدينا اعتماد على `Sleep` نحتاج إلى استخراجه لتتمكن من التحكم فيه في اختباراتنا.

إذا استطعنا عمل _mock_ لـ `time.Sleep`، فسنستطيع استخدام _حقن الاعتماديات_ لاستخدامها بدلًا من `time.Sleep` "الحقيقية"، وحينها نستطيع **التجسس على الاستدعاءات** وعمل تحققات عليها.

## اكتب الاختبار أولًا

لنعرّف اعتمادنا كواجهة. يتيح لنا ذلك استخدام Sleeper _حقيقية_ في `main` و_sleeper متجسسة_ في اختباراتنا. وباستخدام واجهة، تصبح دالتنا `Countdown` غير عابئة بذلك، ويضيف ذلك بعض المرونة للمستدعي.

```go
type Sleeper interface {
	Sleep()
}
```

اتخذت قرارًا تصميميًا بأن دالة `Countdown` لن تكون مسؤولة عن مدة النوم. فهذا يبسّط كودنا قليلًا في الوقت الحالي على الأقل، ويعني أن مستخدم دالتنا يمكنه تهيئة ذلك النوم كما يشاء.

الآن نحتاج إلى عمل _mock_ منه لتستخدمه اختباراتنا.

```go
type SpySleeper struct {
	Calls int
}

func (s *SpySleeper) Sleep() {
	s.Calls++
}
```

_المتجسسات_ (Spies) نوع من الـ _mock_ يمكنها تسجيل كيفية استخدام اعتماد ما. فيمكنها تسجيل الوسائط المُرسلة، وعدد مرات الاستدعاء، وهكذا. وفي حالتنا نتتبع عدد مرات استدعاء `Sleep()` لنفحصه في اختبارنا.

حدّث الاختبارات لتحقن اعتمادًا على الـ Spy الخاص بنا، وتحقّق أن النوم استُدعي 3 مرات.

```go
func TestCountdown(t *testing.T) {
	buffer := &bytes.Buffer{}
	spySleeper := &SpySleeper{}

	Countdown(buffer, spySleeper)

	got := buffer.String()
	want := `3
2
1
Go!`

	if got != want {
		t.Errorf("got %q want %q", got, want)
	}

	if spySleeper.Calls != 3 {
		t.Errorf("not enough calls to sleeper, want 3 got %d", spySleeper.Calls)
	}
}
```

## جرّب تشغيل الاختبار

```
too many arguments in call to Countdown
    have (*bytes.Buffer, *SpySleeper)
    want (io.Writer)
```

## اكتب أصغر قدر من الكود ليعمل الاختبار وافحص مخرجاته الفاشلة

نحتاج إلى تحديث `Countdown` لتقبل `Sleeper` خاصتنا

```go
func Countdown(out io.Writer, sleeper Sleeper) {
	for i := countdownStart; i > 0; i-- {
		fmt.Fprintln(out, i)
		time.Sleep(1 * time.Second)
	}

	fmt.Fprint(out, finalWord)
}
```

إذا جرّبت مرة أخرى، فلن يُترجم برنامج `main` بعد الآن للسبب نفسه

```
./main.go:26:11: not enough arguments in call to Countdown
    have (*os.File)
    want (io.Writer, Sleeper)
```

لننشئ sleeper _حقيقية_ تنفذ الواجهة التي نحتاجها

```go
type DefaultSleeper struct{}

func (d *DefaultSleeper) Sleep() {
	time.Sleep(1 * time.Second)
}
```

ثم يمكننا استخدامها في تطبيقنا الحقيقي هكذا

```go
func main() {
	sleeper := &DefaultSleeper{}
	Countdown(os.Stdout, sleeper)
}
```

## اكتب كودًا كافيًا لنجاح الاختبار

الاختبار يُترجم الآن لكنه لا ينجح، لأننا ما زلنا نستدعي `time.Sleep` بدلًا من الاعتماد المحقون. لنصلح ذلك.

```go
func Countdown(out io.Writer, sleeper Sleeper) {
	for i := countdownStart; i > 0; i-- {
		fmt.Fprintln(out, i)
		sleeper.Sleep()
	}

	fmt.Fprint(out, finalWord)
}
```

من المفترض أن ينجح الاختبار وألا يستغرق 3 ثوانٍ بعد الآن.

### ما زالت هناك بعض المشكلات

ما زالت هناك خاصية مهمة أخرى لم نختبرها.

ينبغي أن تنام `Countdown` قبل كل طباعة تالية، هكذا:

* `Print N`
* `Sleep`
* `Print N-1`
* `Sleep`
* `Print Go!`
* وما إلى ذلك

لا يتحقق آخر تغيير لدينا إلا من أنه نام 3 مرات، لكن هذه النومات قد تحدث بترتيب خاطئ.

عند كتابة الاختبارات، إن لم تكن واثقًا أن اختباراتك تمنحك ثقة كافية، فاكسرها! (تأكد أولًا أنك سجّلت تغييراتك في نظام التحكم في الإصدارات). غيّر الكود إلى ما يلي

```go
func Countdown(out io.Writer, sleeper Sleeper) {
	for i := countdownStart; i > 0; i-- {
		sleeper.Sleep()
	}

	for i := countdownStart; i > 0; i-- {
		fmt.Fprintln(out, i)
	}

	fmt.Fprint(out, finalWord)
}
```

إذا شغّلت اختباراتك فمن المفترض أن تظل ناجحة رغم أن التنفيذ خاطئ.

لنستخدم التجسس مرة أخرى باختبار جديد للتحقق من أن ترتيب العمليات صحيح.

لدينا اعتمدان مختلفان ونريد تسجيل كل عملياتهما في قائمة واحدة. لذا سننشئ _متجسسًا واحدًا لكليهما_.

```go
type SpyCountdownOperations struct {
	Calls []string
}

func (s *SpyCountdownOperations) Sleep() {
	s.Calls = append(s.Calls, sleep)
}

func (s *SpyCountdownOperations) Write(p []byte) (n int, err error) {
	s.Calls = append(s.Calls, write)
	return
}

const write = "write"
const sleep = "sleep"
```

تطبق `SpyCountdownOperations` الخاصة بنا كلا من `io.Writer` و`Sleeper`، وتسجّل كل استدعاء في شريحة واحدة. وفي هذا الاختبار لا يهمنا إلا ترتيب العمليات، لذا يكفي تسجيلها كقائمة عمليات مُسمّاة.

يمكننا الآن إضافة اختبار فرعي إلى مجموعة اختباراتنا يتحقق من أن النوم والطباعة يعملان بالترتيب الذي نأمله

```go
t.Run("sleep before every print", func(t *testing.T) {
	spySleepPrinter := &SpyCountdownOperations{}
	Countdown(spySleepPrinter, spySleepPrinter)

	want := []string{
		write,
		sleep,
		write,
		sleep,
		write,
		sleep,
		write,
	}

	if !reflect.DeepEqual(want, spySleepPrinter.Calls) {
		t.Errorf("wanted calls %v got %v", want, spySleepPrinter.Calls)
	}
})
```

من المفترض أن يفشل هذا الاختبار الآن. أعد `Countdown` إلى ما كانت عليه لإصلاح الاختبار.

أصبح لدينا الآن اختباران يتجسسان على `Sleeper`، لذا يمكننا إعادة هيكلة اختبارنا بحيث يختبر أحدهما ما يُطبع ويضمن الآخر أننا ننام بين الطبعات. وأخيرًا يمكننا حذف أول متجسس لدينا لأنه لم يعد مستخدمًا.

```go
func TestCountdown(t *testing.T) {

	t.Run("prints 3 to Go!", func(t *testing.T) {
		buffer := &bytes.Buffer{}
		Countdown(buffer, &SpyCountdownOperations{})

		got := buffer.String()
		want := `3
2
1
Go!`

		if got != want {
			t.Errorf("got %q want %q", got, want)
		}
	})

	t.Run("sleep before every print", func(t *testing.T) {
		spySleepPrinter := &SpyCountdownOperations{}
		Countdown(spySleepPrinter, spySleepPrinter)

		want := []string{
			write,
			sleep,
			write,
			sleep,
			write,
			sleep,
			write,
		}

		if !reflect.DeepEqual(want, spySleepPrinter.Calls) {
			t.Errorf("wanted calls %v got %v", want, spySleepPrinter.Calls)
		}
	})
}
```

أصبح لدينا الآن دالتنا وخاصيتاها المهمتان مختبرتان اختبارًا جيدًا.

## توسيع Sleeper ليكون قابلًا للتهيئة

من الميزات الجميلة أن يكون `Sleeper` قابلًا للتهيئة. فهذا يعني أننا نستطيع ضبط مدة النوم في برنامجنا الرئيسي.

### اكتب الاختبار أولًا

لننشئ أولًا نوعًا جديدًا هو `ConfigurableSleeper` يقبل ما نحتاجه للتهيئة والاختبار.

```go
type ConfigurableSleeper struct {
	duration time.Duration
	sleep    func(time.Duration)
}
```

نستخدم `duration` لتهيئة مدة النوم، و`sleep` كطريقة لتمرير دالة النوم. وتوقيع `sleep` هو نفسه توقيع `time.Sleep`، ما يتيح لنا استخدام `time.Sleep` في تنفيذنا الحقيقي واستخدام الـ spy التالي في اختباراتنا:

```go
type SpyTime struct {
	durationSlept time.Duration
}

func (s *SpyTime) SetDurationSlept(duration time.Duration) {
	s.durationSlept = duration
}
```

وبعد أن أصبح الـ spy جاهزًا، يمكننا إنشاء اختبار جديد للـ sleeper القابل للتهيئة.

```go
func TestConfigurableSleeper(t *testing.T) {
	sleepTime := 5 * time.Second

	spyTime := &SpyTime{}
	sleeper := ConfigurableSleeper{sleepTime, spyTime.SetDurationSlept}
	sleeper.Sleep()

	if spyTime.durationSlept != sleepTime {
		t.Errorf("should have slept for %v but slept for %v", sleepTime, spyTime.durationSlept)
	}
}
```

لا ينبغي أن يكون في هذا الاختبار أي شيء جديد، وهو مُعدّ بشكل مشابه جدًا لاختبارات الـ mock السابقة.

### جرّب تشغيل الاختبار

```
sleeper.Sleep undefined (type ConfigurableSleeper has no field or method Sleep, but does have sleep)

```

من المفترض أن ترى رسالة خطأ واضحة جدًا تشير إلى أننا لم ننشئ method اسمها `Sleep` على `ConfigurableSleeper`.

### اكتب أصغر قدر من الكود ليعمل الاختبار وافحص مخرجاته الفاشلة

```go
func (c *ConfigurableSleeper) Sleep() {
}
```

وبعد تنفيذ دالة `Sleep` الجديدة لدينا، صار لدينا اختبار فاشل.

```
countdown_test.go:56: should have slept for 5s but slept for 0s
```

### اكتب كودًا كافيًا لنجاح الاختبار

كل ما علينا فعله الآن هو تنفيذ دالة `Sleep` الخاصة بـ `ConfigurableSleeper`.

```go
func (c *ConfigurableSleeper) Sleep() {
	c.sleep(c.duration)
}
```

بهذا التغيير، من المفترض أن تنجح كل الاختبارات مرة أخرى، وقد تتساءل عن جدوى كل هذا العناء، إذ لم يتغير البرنامج الرئيسي على الإطلاق. وآمل أن يتضح الأمر بعد القسم التالي.

### التنظيف وإعادة الهيكلة

آخر ما علينا فعله هو استخدام `ConfigurableSleeper` فعليًا في الدالة `main`.

```go
func main() {
	sleeper := &ConfigurableSleeper{1 * time.Second, time.Sleep}
	Countdown(os.Stdout, sleeper)
}
```

إذا شغّلنا الاختبارات والبرنامج يدويًا، يمكننا أن نرى أن السلوك كله يبقى كما هو.

وبما أننا نستخدم `ConfigurableSleeper`، أصبح من الآمن الآن حذف تنفيذ `DefaultSleeper`. وبهذا نُنجز برنامجنا ونحصل على Sleeper أكثر [عمومية](https://stackoverflow.com/questions/19291776/whats-the-difference-between-abstraction-and-generalization) مع عدّ تنازلي طويل كما نشاء.

## لكن أليست الـ mocking شرًا؟

ربما سمعت أن الـ mocking شر. وكأي شيء في تطوير البرمجيات يمكن استخدامه في الشر، تمامًا مثل [DRY](https://en.wikipedia.org/wiki/Don%27t_repeat_yourself).

يقع الناس عادةً في حالة سيئة عندما لا _يستمعون إلى اختباراتهم_ و_لا يحترمون مرحلة إعادة الهيكلة_.

إذا أصبح كود الـ mocking لديك معقدًا، أو وجدت نفسك مضطرًا لعمل mock لأشياء كثيرة لاختبار شيء ما، فينبغي أن _تستمع_ إلى ذلك الشعور السيئ وتفكر في كودك. فعادةً ما تكون هذه علامة على

* أن الشيء الذي تختبره يقوم بأشياء كثيرة جدًا (لأن لديه اعتماديات كثيرة جدًا لعمل mock لها)
  * فكّك الوحدة (module) إلى أجزاء لتفعل أقل
* أن اعتمادياته منقسمة إلى أجزاء صغيرة أكثر من اللازم
  * فكّر في كيفية دمج بعض هذه الاعتماديات في وحدة واحدة ذات معنى
* أن اختبارك منشغل أكثر من اللازم بتفاصيل التنفيذ
  * فضّل اختبار السلوك المتوقع بدلًا من اختبار التنفيذ

وغالبًا ما تشير كثرة الـ mocking إلى _تجريد سيئ_ في كودك.

**ما يراه الناس هنا ضعف في التطوير الموجه بالاختبار، لكنه في الحقيقة قوة**. فكود الاختبار الرديء غالبًا ما يكون نتيجة تصميم سيئ، أو بصياغة أجمل: الكود المصمم جيدًا سهل الاختبار.

### لكن الـ mocks والاختبارات ما زالت تصعّب حياتي!

هل واجهت هذه الحالة يومًا؟

* تريد القيام ببعض إعادة الهيكلة
* ولتفعل ذلك تنتهي بتغيير كثير من الاختبارات
* فتتساءل عن جدوى التطوير الموجه بالاختبار وتكتب مقالًا على Medium بعنوان "Mocking considered harmful"

وهذه عادةً علامة على أنك تختبر قدرًا كبيرًا جدًا من _تفاصيل التنفيذ_. حاول أن تجعل اختباراتك تختبر _سلوكًا مفيدًا_ إلا إذا كان التنفيذ مهمًا فعلًا لطريقة عمل النظام.

يصعب أحيانًا معرفة _المستوى_ الذي ينبغي اختباره بالضبط، لكن إليك بعض العمليات الذهنية والقواعد التي أحاول اتباعها:

* **تعريف إعادة الهيكلة هو أن الكود يتغير لكن السلوك يبقى كما هو**. فإذا قررت أن تجري بعض إعادة الهيكلة نظريًا، ينبغي أن تكون قادرًا على عمل الـ commit دون أي تغييرات في الاختبارات. لذا عند كتابة اختبار اسأل نفسك
  * هل أختبر السلوك الذي أريده أم تفاصيل التنفيذ؟
  * لو أعدت هيكلة هذا الكود، هل سأضطر إلى كثير من التغييرات في الاختبارات؟
* مع أن Go تتيح لك اختبار الدوال الخاصة، سأتجنب ذلك، لأن الدوال الخاصة تفاصيل تنفيذ تدعم السلوك العام. فاختبر السلوك العام. وتصف Sandi Metz الدوال الخاصة بأنها "أقل استقرارًا"، وأنت لا تريد ربط اختباراتك بها.
* أشعر أنه إذا كان الاختبار يعمل مع **أكثر من 3 mocks فهذه إشارة تحذير** - حان وقت إعادة التفكير في التصميم
* استخدم الـ spies بحذر. فالـ spies تتيح لك رؤية داخل الخوارزمية التي تكتبها، وهذا قد يكون مفيدًا جدًا، لكنه يعني ارتباطًا أشد بين كود اختبارك والتنفيذ. **تأكد أنك تهتم فعلًا بهذه التفاصيل إذا كنت ستتجسس عليها**

#### ألا يمكنني استخدام إطار للـ mocking فحسب؟

الـ mocking لا يتطلب أي سحر وهو بسيط نسبيًا؛ واستخدام إطار عمل قد يجعل الـ mocking يبدو أكثر تعقيدًا مما هو عليه. نحن لا نستخدم automocking في هذا الفصل حتى نحصل على:

* فهم أفضل لكيفية عمل الـ mocking
* تدريب على تنفيذ الواجهات

في المشاريع التعاونية هناك قيمة لتوليد الـ mocks تلقائيًا. ففي فريق العمل، تُقنّن أداة توليد الـ mocks الاتساق في الـ test doubles، ما يتجنب كتابة test doubles غير متسقة قد تُترجم إلى اختبارات غير متسقة.

ينبغي أن تستخدم فقط مولّد mocks ينتج test doubles استنادًا إلى واجهة. وأي أداة تتحكم بإفراط في كيفية كتابة الاختبارات، أو تستخدم الكثير من "السحر"، فمكانها البحر.

## الخلاصة

### المزيد عن منهج التطوير الموجه بالاختبار

* عند مواجهة أمثلة أقل بساطة، قسّم المشكلة إلى "شرائح رأسية رفيعة". وحاول أن تصل في أسرع وقت ممكن إلى _برمجية تعمل مدعومة بالاختبارات_، لتتجنب السقوط في جحور الأرانب واتباع منهج "الانفجار الكبير".
* وبعد أن تصبح لديك بعض البرمجيات العاملة، ينبغي أن يصبح _التقدم بخطوات صغيرة_ أسهل حتى تصل إلى البرمجيات التي تحتاجها.

> "متى تستخدم التطوير التكراري؟ ينبغي أن تستخدم التطوير التكراري فقط في المشاريع التي تريد أن تنجح."

Martin Fowler.

### الـ Mocking

* **بدون الـ mocking ستبقى مناطق مهمة من كودك دون اختبار**. ففي حالتنا لن نستطيع اختبار أن كودنا توقف بين كل طباعة، لكن هناك أمثلة لا تُحصى غيرها. الاتصال بخدمة _قد_ تفشل؟ تريد اختبار نظامك في حالة معينة؟ يصعب جدًا اختبار هذه السيناريوهات بدون mocking.
* وبدون الـ mocks قد تضطر إلى إعداد قواعد بيانات وأشياء أخرى من أطراف ثالثة لمجرد اختبار قواعد عمل بسيطة. والأرجح أن تصبح اختباراتك بطيئة، ما يؤدي إلى **حلقات تغذية راجعة بطيئة**.
* وباضطرارك إلى تشغيل قاعدة بيانات أو خدمة ويب لاختبار شيء ما، من الأرجح أن تحصل على **اختبارات هشة** بسبب عدم موثوقية مثل هذه الخدمات.

وبمجرد أن يتعلم المطور الـ mocking، يصبح من السهل جدًا الإفراط في اختبار كل جانب من جوانب النظام من حيث _طريقة عمله_ بدلًا من _ما يفعله_. وكن دائمًا منتبهًا إلى **قيمة اختباراتك** وإلى أثرها في إعادة الهيكلة مستقبلًا.

في هذا الفصل عن الـ mocking لم نتناول إلا **الـ Spies**، وهي نوع من الـ mock. والـ Mocks نوع من "test double".

> [الـ Test Double مصطلح عام يطلق على أي حالة تستبدل فيها كائنًا إنتاجيًا لغرض الاختبار.](https://martinfowler.com/bliki/TestDouble.html)

وتحت مظلة الـ test doubles توجد أنواع عديدة مثل stubs وspies وحتى mocks! ألقِ نظرة على [مقال Martin Fowler](https://martinfowler.com/bliki/TestDouble.html) لمزيد من التفاصيل.

## إضافة - مثال على الـ iterators من Go 1.23

في Go 1.23 [أُضيفت الـ iterators](https://tip.golang.org/doc/go1.23). ويمكننا استخدام الـ iterators بطرق متعددة، وفي هذه الحالة يمكننا عمل iterator باسم `countdownFrom` يُرجع الأرقام للعد التنازلي بترتيب معكوس.

وقبل أن ندخل في كيفية كتابة iterators مخصصة، لنرَ كيف نستخدمها. فبدلًا من كتابة حلقة تبدو أمرية إلى حد كبير للعد تنازليًا من رقم، يمكننا أن نجعل هذا الكود أكثر تعبيرًا بالمرور بـ `range` على iterator المخصص `countdownFrom`.

```go
func Countdown(out io.Writer, sleeper Sleeper) {
	for i := range countDownFrom(3) {
		fmt.Fprintln(out, i)
		sleeper.Sleep()
	}

	fmt.Fprint(out, finalWord)
}
```

لكتابة iterator مثل `countDownFrom`، تحتاج إلى كتابة دالة بطريقة معينة. من التوثيق:

    The “range” clause in a “for-range” loop now accepts iterator functions of the following types
        func(func() bool)
        func(func(K) bool)
        func(func(K, V) bool)

(يرمز `K` و`V` إلى نوعَي المفتاح والقيمة على الترتيب.)

وفي حالتنا لا توجد مفاتيح، بل قيم فقط. وتوفر Go أيضًا نوعًا جاهزًا هو `iter.Seq[T]`، وهو اسم بديل (type alias) للنوع `func(func(T) bool)`.

```go
func countDownFrom(from int) iter.Seq[int] {
	return func(yield func(int) bool) {
		for i := from; i > 0; i-- {
			if !yield(i) {
				return
			}
		}
	}
}
```

هذا iterator بسيط سيُخرج الأرقام بترتيب معكوس بدءًا من `from` - مثالي لحالتنا.
