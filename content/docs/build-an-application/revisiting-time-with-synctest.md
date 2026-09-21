---
title: إعادة زيارة الوقت مع testing/synctest
weight: 320
---

# إعادة زيارة الوقت مع testing/synctest

**[يمكنك العثور على كل كود هذا الفصل هنا](https://github.com/quii/learn-go-with-tests/tree/main/synctest)**

في [فصل الوقت](time.md) منحنا تطبيق البوكر الطرفي (poker CLI) القدرة على جدولة تنبيهات "الرهان الأعمى يرتفع الآن" باستخدام `time.AfterFunc`. كان اختبار هذا صعبًا: فـ `time.AfterFunc` يشغّل دالة الاستدعاء (callback) في goroutine خاصة به بعد مرور مدة حقيقية، ولا يمكنك مقارنة الدوال في Go، لذا لم نستطع بسهولة فحص ما تمت جدولته.

حللنا ذلك بأداة مألوفة: حقن الاعتماديات (dependency injection). عرّفنا واجهة (interface) اسمها `BlindAlerter`، وفي اختباراتنا استبدلنا التنفيذ الحقيقي بـ spy يسجّل فقط ما طُلب منه جدولته، هكذا تقريبًا:

```go
type BlindAlerter interface {
	ScheduleAlertAt(duration time.Duration, amount int)
}

type SpyBlindAlerter struct {
	Alerts []struct {
		At     time.Duration
		Amount int
	}
}

func (s *SpyBlindAlerter) ScheduleAlertAt(at time.Duration, amount int) {
	s.Alerts = append(s.Alerts, struct {
		At     time.Duration
		Amount int
	}{at, amount})
}
```

هذا تصميم جيد، والاختبارات التي يتيحها سريعة وموثوقة. لكن انظر عن قرب إلى ما تغطيه فعلًا. فهي تؤكد أشياء مثل "تم استدعاء `ScheduleAlertAt` بالمدة `10 * time.Minute` والكمية `200`". وهي لا تدع أي منبّه *حقيقي* يعمل أبدًا. والجزء الوحيد من الكود الذي يستدعي `time.AfterFunc` فعلًا ويطبع شيئًا، أي الجزء الذي يحمل احتمال الخطأ الحقيقي، لا يمر عليه أي اختبار.

هذا ليس سهوًا، بل مقايضة. فاختبار منبّه حقيقي مبني على `time.AfterFunc` اختبارًا سليمًا يعني إما انتظار دقائق حقيقية حتى ينتهي الاختبار، أو تقليص المدد إلى أجزاء من الثانية والأمل ألا تكون الآلة التي تشغّل الاختبار مشغولة أكثر من اللازم. ولا أيّ من الخيارين جاذب. لذا تاريخيًا، لم نفعل ذلك ببساطة.

بدءًا من Go 1.25، يوجد خيار ثالث: [`testing/synctest`](https://pkg.go.dev/testing/synctest). فهو يتيح لنا تشغيل كود حقيقي غير معدّل يستخدم `time.Sleep` و`time.AfterFunc` وأمثالهما، داخل اختبار يتحكم في الوقت تحكمًا كاملًا: بلا انتظار، وبلا تقلّب (flakiness)، وبلا تقليص للمدد مع الأمل في الأفضل.

في هذا الفصل سنبني ذلك المنبّه الحقيقي من الصفر، ونختبره بـ `synctest`. لكن لنستحق ذلك، فلنرَ أولًا ما الذي يخلّصنا منه بالضبط.

## ما يكفي من المعلومات عن `testing/synctest`

يُشغّل `testing/synctest` دالة داخل **فقاعة** (bubble) معزولة. وداخل تلك الفقاعة:

* تستخدم حزمة `time` ساعة وهمية (fake clock). تبدأ عند منتصف الليل بتوقيت UTC في الأول من يناير عام 2000، ولا تتحرك إلا إلى الأمام.
* لا يتقدم الوقت الوهمي إلا عندما تصبح كل goroutine في الفقاعة **محجوبة حجبًا دائمًا** (durably blocked): أي محجوبة بطريقة لا تستطيع فكّ حجبها إلا goroutine أخرى في الفقاعة نفسها. و`time.Sleep`، والاستقبال الحاجب (blocking receive) على channel أُنشئ داخل الفقاعة، و`sync.Cond.Wait`، و`sync.WaitGroup.Wait` كلها تُحتسب. أما حجز `sync.Mutex` فلا يُحتسب، لأن الـ mutexes تُحتجز عادةً لفترة قصيرة فقط.
* يُشغّل `synctest.Test(t, f)` الدالة `f` في فقاعة جديدة، ولا يعود حتى تخرج كل goroutine وُلدت بداخلها. وإذا انتهى الأمر بالفقاعة إلى حجب دائم بلا سبيل إلى التقدم، فإنه يُفشل الاختبار باعتباره جمودًا (deadlock) بدلًا من أن يتعلّق إلى الأبد.
* يحجب `synctest.Wait()` الـ goroutine المستدعية حتى تُحجب كل goroutine *أخرى* في الفقاعة حجبًا دائمًا، ثم يعود. وهكذا تدع العمل الخلفي يستقر قبل إجراء أي تحقق (assertion).

هذا يكفي للبدء. وسنلتقط بعض الحواف الأكثر حدّة أثناء الطريق.

## اكتب الاختبار أولًا

لنبدأ بالشيء البديهي، قبل أن نمدّ أيدينا إلى `synctest` أصلًا: منبّه `BlindAlerter` يأخذ مدة وكمية، وينتظر تلك المدة ثم يكتب رسالة في مكان ما، ويُختبر بالوقت الحقيقي.

```go
package poker

import (
	"bytes"
	"testing"
	"time"
)

func TestStdOutAlerter(t *testing.T) {
	out := &bytes.Buffer{}
	alerter := StdOutAlerter(out)

	alerter.ScheduleAlertAt(5*time.Second, 100)

	time.Sleep(6 * time.Second)

	want := "Blind is now 100"
	if out.String() != want {
		t.Errorf("got %q, want %q", out.String(), want)
	}
}
```

ننام مدة أطول قليلًا من التنبيه المجدول (6 ثوانٍ لا 5) لنمنح الـ goroutine التي يولدها `time.AfterFunc` لحظة لتعمل فعلًا قبل أن نتحقق.

## جرّب تشغيل الاختبار

لم نكتب أي كود إنتاج بعد، لذا لن يترجم هذا:

```
./blind_alerter_test.go:11:13: undefined: StdOutAlerter
```

## اكتب كودًا كافيًا لنجاح الاختبار

```go
package poker

import (
	"fmt"
	"io"
	"time"
)

// BlindAlerter schedules alerts for blind amounts.
type BlindAlerter interface {
	ScheduleAlertAt(duration time.Duration, amount int)
}

// BlindAlerterFunc allows you to implement BlindAlerter with a function.
type BlindAlerterFunc func(duration time.Duration, amount int)

// ScheduleAlertAt is BlindAlerterFunc's implementation of BlindAlerter.
func (a BlindAlerterFunc) ScheduleAlertAt(duration time.Duration, amount int) {
	a(duration, amount)
}

// StdOutAlerter returns a BlindAlerterFunc that schedules alerts and prints them to out.
func StdOutAlerter(out io.Writer) BlindAlerterFunc {
	return func(duration time.Duration, amount int) {
		time.AfterFunc(duration, func() {
			fmt.Fprintf(out, "Blind is now %d", amount)
		})
	}
}
```

شغّله:

```
=== RUN   TestStdOutAlerter
--- PASS: TestStdOutAlerter (6.00s)
PASS
ok  	github.com/quii/learn-go-with-tests/synctest/v1	6.191s
```

ينجح. ويستغرق أيضًا ست ثوانٍ حقيقية، من أجل تنبيه واحد. ولعبتنا الفعلية تجدول أحد عشر تنبيهًا منها، بعضها تفصل بينه ساعة وأربعون دقيقة. لن ينتظر أحد ذلك في كل تشغيل للاختبارات، لذا فعمليًا سيُتخطى هذا الاختبار، أو يُكتب بمدد صغيرة غير واقعية لا تشبه الحقيقي إلا قليلًا. وهذه هي المشكلة التي وُجد `synctest` لحلها.

## تقديم synctest

لنلفّ الاختبار نفسه في فقاعة ونرَ ماذا يحدث إذا كنا متفائلين أكثر قليلًا بشأن ما تفعله "ساعة وهمية" من أجلنا:

```go
func TestStdOutAlerter(t *testing.T) {
	synctest.Test(t, func(t *testing.T) {
		out := &bytes.Buffer{}
		alerter := StdOutAlerter(out)

		alerter.ScheduleAlertAt(5*time.Second, 100)

		want := "Blind is now 100"
		if out.String() != want {
			t.Errorf("got %q, want %q", out.String(), want)
		}
	})
}
```

حذفنا النوم تمامًا: ألا تتدبر لنا الساعة الوهمية ذلك الآن؟ إنها لا تفعل:

```
=== RUN   TestStdOutAlerter
    blind_alerter_test.go:19: got "", want "Blind is now 100"
--- FAIL: TestStdOutAlerter (0.00s)
FAIL
```

لا تدير ساعة الفقاعة الوهمية نفسها إلى الأمام بمؤقت خاص بها. فهي لا تتقدم إلا عندما يُحجب شيء في الفقاعة حجبًا دائمًا بانتظارها. وما زال علينا أن نقول ما الذي ننتظره؛ لكن لم يعد علينا أن ندفع ثمنه بثوانٍ حقيقية. لنعد النوم:

```go
func TestStdOutAlerter(t *testing.T) {
	synctest.Test(t, func(t *testing.T) {
		out := &bytes.Buffer{}
		alerter := StdOutAlerter(out)

		alerter.ScheduleAlertAt(5*time.Second, 100)

		time.Sleep(6 * time.Second)

		want := "Blind is now 100"
		if out.String() != want {
			t.Errorf("got %q, want %q", out.String(), want)
		}
	})
}
```

```
=== RUN   TestStdOutAlerter
--- PASS: TestStdOutAlerter (0.00s)
PASS
ok  	github.com/quii/learn-go-with-tests/synctest/v2	0.133s
```

ينجح، وفورًا: فـ `time.Sleep(6 * time.Second)` داخل فقاعة لا يكلف شيئًا من الوقت الحقيقي. يبدو أننا انتهينا. لكن دعنا نتأكد أنه يصمد تحت `-race`، من باب العادة:

```
go test -race ./...
```

```
==================
WARNING: DATA RACE
Read at 0x00c00010e630 by goroutine 9:
  bytes.(*Buffer).String()
      /usr/local/go/src/bytes/buffer.go:77 +0x174
  github.com/quii/learn-go-with-tests/synctest/v2.TestStdOutAlerter.func1()
      blind_alerter_test.go:20 +0x15c
  testing.tRunner()
      /usr/local/go/src/testing/testing.go:1934 +0x164

Previous write at 0x00c00010e630 by goroutine 11:
  bytes.(*Buffer).grow()
      /usr/local/go/src/bytes/buffer.go:143 +0x354
  bytes.(*Buffer).Write()
      /usr/local/go/src/bytes/buffer.go:185 +0xb4
  fmt.Fprintf()
      /usr/local/go/src/fmt/print.go:225 +0x94
  github.com/quii/learn-go-with-tests/synctest/v2.TestStdOutAlerter.func1.BlindAlerterFunc.ScheduleAlertAt.TestStdOutAlerter.func1.StdOutAlerter.1.2()
      blind_alerter.go:26 +0x6c

Goroutine 11 (finished) created at:
  time.goFunc()
      /usr/local/go/src/time/sleep.go:215 +0x40
==================
--- FAIL: TestStdOutAlerter (0.00s)
    testing.go:1617: race detected during execution of test
FAIL
```

آخ. لاحظ "Goroutine 11 (finished)": فالكتابة حدثت فعلًا قبل قراءتنا، في كل مرة شغّلنا فيها الاختبار. ووظيفيًا، لا يمكن للاختبار أن يفشل في التحقق الفعلي أبدًا. لكن كاشف التسابق (race detector) لا يسأل "هل ساءت الأمور هذه المرة؟"، بل يسأل "هل يوجد ما *يضمن* أنها لا تسوء؟". فـ `time.Sleep` على الـ goroutine الخاصة بنا ودالة الاستدعاء الخاصة بـ `time.AfterFunc` على goroutine خاصة به ما هي إلا goroutines مجدولة كل منها على حدة. ولا شيء في "نمنا بعض الوقت" يضمن أن goroutine أخرى قد انتهت من لمس الذاكرة المشتركة، مهما كان النوم سخيًا. والنوم لمدة أطول لا يصلح هذا، مهما طال؛ إنه فقط يجعل الفشل أقل تكرارًا خارج `-race`.

يستحق أن نتوقف عنده لحظة: هذا التسابق نفسه كان كامنًا بالفعل في نسختنا الأولى البسيطة بالوقت الحقيقي أيضًا، مع `time.Sleep` حقيقية. غير أننا لم نفكر قط في التحقق: من يشغّل `-race`، مرارًا، على اختبار يستغرق أصلًا ست ثوانٍ حقيقية؟

## اكتب الاختبار أولًا

ما نحتاجه هو نقطة مزامنة (synchronization) حقيقية: شيء لا يجعل الكتابة *مرجّحة* أن تحدث أولًا فحسب، بل يثبت فعلًا أنها حدثت. وهذا بالضبط ما وُجد `synctest.Wait()` من أجله:

```go
func TestStdOutAlerter(t *testing.T) {
	synctest.Test(t, func(t *testing.T) {
		out := &bytes.Buffer{}
		alerter := StdOutAlerter(out)

		alerter.ScheduleAlertAt(5*time.Second, 100)

		time.Sleep(6 * time.Second)
		synctest.Wait()

		want := "Blind is now 100"
		if out.String() != want {
			t.Errorf("got %q, want %q", out.String(), want)
		}
	})
}
```

## إعادة الهيكلة

لا يوجد ما نغيّره في كود الإنتاج، لكن لنحرص أن هذه النسخة تصمد فعلًا، مرارًا، تحت `-race`:

```
ok  	github.com/quii/learn-go-with-tests/synctest/v3	1.153s
ok  	github.com/quii/learn-go-with-tests/synctest/v3	1.143s
ok  	github.com/quii/learn-go-with-tests/synctest/v3	1.148s
ok  	github.com/quii/learn-go-with-tests/synctest/v3	1.147s
ok  	github.com/quii/learn-go-with-tests/synctest/v3	1.143s
```

نظيف، في كل مرة. جميل جدًا.

## اكتب الاختبار أولًا

هناك شيء آخر يستحق الاختبار: ألا يكون قد حدث شيء بعد. مع الوقت الحقيقي، يعني ذلك تخمين مدة نوم طويلة بما يكفي لنثق، لكن دون أن يطول الاختبار كثيرًا. أما synctest فلا يحتاج إلى التخمين:

```go
func TestStdOutAlerter(t *testing.T) {
	synctest.Test(t, func(t *testing.T) {
		out := &bytes.Buffer{}
		alerter := StdOutAlerter(out)

		alerter.ScheduleAlertAt(5*time.Second, 100)

		synctest.Wait()
		if out.String() != "" {
			t.Errorf("did not expect anything to be printed yet, got %q", out.String())
		}

		time.Sleep(5 * time.Second)
		synctest.Wait()

		want := "Blind is now 100"
		if out.String() != want {
			t.Errorf("got %q, want %q", out.String(), want)
		}
	})
}
```

لا شيء آخر في الفقاعة يفعل أي شيء عند النقطة التي نجدول فيها التنبيه، لذا ينبغي أن يعود `Wait()` فورًا: فلا شيء ننتظر *من أجله* بعد، لذا ينبغي أن يظل `out` فارغًا. شغّله:

```
=== RUN   TestStdOutAlerter
--- PASS: TestStdOutAlerter (0.00s)
PASS
```

ينجح. لنفحص `-race`، كما صرنا نعرف أنه ينبغي:

```
==================
WARNING: DATA RACE
Write at 0x00c00011c630 by goroutine 11:
  bytes.(*Buffer).grow()
      /usr/local/go/src/bytes/buffer.go:143 +0x354
  bytes.(*Buffer).Write()
      /usr/local/go/src/bytes/buffer.go:185 +0xb4
  fmt.Fprintf()
      /usr/local/go/src/fmt/print.go:225 +0x94
  github.com/quii/learn-go-with-tests/synctest/v4.TestStdOutAlerter.func1.BlindAlerterFunc.ScheduleAlertAt.TestStdOutAlerter.func1.StdOutAlerter.1.2()
      blind_alerter.go:26 +0x6c

Previous read at 0x00c00011c630 by goroutine 9:
  bytes.(*Buffer).String()
      /usr/local/go/src/bytes/buffer.go:77 +0x164
  github.com/quii/learn-go-with-tests/synctest/v4.TestStdOutAlerter.func1()
      blind_alerter_test.go:18 +0x14c
  testing.tRunner()
      /usr/local/go/src/testing/testing.go:1934 +0x164

Goroutine 11 (running) created at:
  time.goFunc()
      /usr/local/go/src/time/sleep.go:215 +0x40
==================
--- FAIL: TestStdOutAlerter (0.00s)
    testing.go:1617: race detected during execution of test
FAIL
```

التسابق عند السطر 18: أول فحص لنا، الفحص الذي كنا واثقين منه، لأنه "لا شيء ننتظره بعد". وبثبات، في كل تشغيل.

المشكلة أن `Wait()` وحده لا شيء يحدّ من المدى الذي يسمح للساعة الوهمية أن تصل إليه. فلا توجد goroutine أخرى في الفقاعة عند تلك النقطة، ولا موعد نهائي قادم *نحن* ننتظره. ولم يبقَ سوى مؤقت التنبيه البالغ 5 ثوانٍ، الجالس في كومة المؤقتات (timer heap) الخاصة بالـ runtime. لذا يفعل الـ runtime الشيء المفيد الوحيد الممكن: يُطلق المؤقت ليتقدم، فيولّد goroutine رقم 11 لتشغيل دالة الاستدعاء *بينما* ما زال `Wait()` يقرر هل يعود. فالـ goroutine رقم 11 يعمل فعلًا بالتوازي مع فحصنا، لا قبله. وأحيانًا تنتهي تلك الكتابة أولًا؛ وأحيانًا لا. والـ `bytes.Buffer` الذي نلمسه كلانا بلا قفل، فلا شيء يجعل ذلك آمنًا في أي من الحالتين.

يمكننا إصلاح ذلك بالطريقة التي نصلح بها أي تسابق بيانات (data race): نلفّ `out` في `sync.Mutex`، أو نستخدم `atomic.Pointer[T]` من `sync/atomic` كلمسة أخف. لكن انظر إلى ما يُطلب من `StdOutAlerter` فعلًا: أن يقرر الرسالة *و* ينفذ الأثر الجانبي المتمثل في طباعتها. ولا شيء في جدولة تنبيه رهان أعمى في البوكر يستلزم معرفة بـ `io.Writer`؛ فتلك مسألة المستدعي. وهذا شكل التوتر الذي يعود إليه هذا الكتاب دائمًا: [إذا كانت اختباراتك تسبب لك ألمًا، فاستمع إلى تلك الإشارة وفكّر في تصميم كودك](../questions-and-answers/http-handlers-revisited.md). لنجعل المنبّه ينتج الرسالة فقط عندما يحين وقتها، وليقرر المستدعي ما يفعل بها.

## إعادة الهيكلة

عبور حدود goroutine هو بالضبط ما وُجدت الـ channels من أجله. وكما يقول المثل في Go: [لا تتواصل بمشاركة الذاكرة؛ بل شارك الذاكرة بالتواصل](https://go.dev/blog/codelab-share). فبدلًا من الكتابة في `out` مشترك، يمكن لمنبّهنا أن يرسل الرسالة النهائية عبر channel:

```go
func NewAlerter() (BlindAlerterFunc, <-chan string) {
	alerts := make(chan string)

	scheduleAlertAt := func(duration time.Duration, amount int) {
		time.AfterFunc(duration, func() {
			alerts <- fmt.Sprintf("Blind is now %d", amount)
		})
	}

	return scheduleAlertAt, alerts
}
```

يبقى `BlindAlerter` و`BlindAlerterFunc` كما هما بالضبط. أما `StdOutAlerter` وحده (الذي لم يعد له، بشكل مثير للدلالة، أي علاقة بـ stdout) فقد اختفى، وحلّ محله `NewAlerter` الذي يعيد لنا المنبّه وchannel نستقبل منه. وأي شيء يريد طباعة هذه التنبيهات (مثل `main`) يمكنه المرور على ذلك الـ channel وطباعتها؛ فذلك لم يعد مشكلة هذه الحزمة.

يصبح الاختبار أبسط أيضًا:

```go
func TestNewAlerter(t *testing.T) {
	synctest.Test(t, func(t *testing.T) {
		alerter, alerts := NewAlerter()

		alerter.ScheduleAlertAt(5*time.Second, 100)

		select {
		case got := <-alerts:
			t.Fatalf("did not expect an alert yet, got %q", got)
		default:
		}

		got := <-alerts
		want := "Blind is now 100"
		if got != want {
			t.Errorf("got %q, want %q", got, want)
		}
	})
}
```

لاحظ ما اختفى: لا `synctest.Wait()` في أي مكان، ولا نوع مخصص لحراسة قيمة مشتركة، ولا `time.Sleep` إطلاقًا. لم نعد نحتاج أيًا منها.

إن جملة `select` مع حالة `default`، المأخوذة مباشرة من [فصل select](../go-fundamentals/select.md)، لا تحجب أبدًا: فهي إما تأخذ حالة جاهزة أو تمر إلى `default` فورًا. وعند هذه النقطة ما زال الوقت الوهمي عند الصفر، أي ينقصه خمس ثوانٍ (وهمية) كاملة حتى التنبيه، فلا شيء يحتاج إلى مزامنة: ومن الطبيعي ألا يكون قد وصل شيء بعد. و`got := <-alerts` لا يحتاج إلى دفعة أيضًا: فهو استقبال حاجب عادي، لذا تفعل الفقاعة بالضبط ما صُممت لأجله: فبما أن الشيء الوحيد الذي ينتظره أي أحد في الفقاعة هو ذلك المؤقت، يقفز الوقت الوهمي مباشرة إلى لحظة انطلاقه، فيوقظ goroutine الخاص بـ `time.AfterFunc` ويفكّ حجب استقبالنا.

شغّل هذا مع `-race`، مرارًا. ويبقى أخضر: فلم تعد هناك ذاكرة مشتركة ليتسابق عليها.

قارن هذا بأسلوب `SpyBlindAlerter` من الفصل السابق. فذلك ما زال أداة جيدة تمامًا لمهمة مختلفة: إنه يفحص *ما* الذي تتم جدولته (هل جُدولت الكميات الصحيحة عند الإزاحات الصحيحة لعدد معين من اللاعبين؟) دون أن يبالي بالتوقيت الحقيقي إطلاقًا (وهو مفيد عندما تختبر حسابًا، لا توقيتًا). فـ `synctest` لم يستبدل تلك الحاجة؛ بل يملأ فجوة التغطية التي يتركها أي spy صرف من نوع "ماذا طُلب مني أن أفعل": هل تعمل آلية الجدولة نفسها فعلًا؟

## الخلاصة

### ما غطّيناه

* `synctest.Test` وفكرة فقاعة ذات ساعة وهمية معزولة، لا تتقدم إلا عندما يُحجب شيء حجبًا دائمًا، وليس من تلقاء نفسها.
* "محجوب حجبًا دائمًا": الشرط الذي يسمح للوقت الوهمي بالتقدم، ولماذا لا يُحتسب `sync.Mutex` عن قصد (بينما يُحتسب استقبال channel).
* النوم "مدة كافية" ليس مثل المزامنة: فحتى نوم سخي ينجح عمليًا دائمًا يظل تسابق بيانات حقيقيًا إن لم يكن هناك ما يفرض الترتيب.
* `synctest.Wait()` يصلح ذلك لفحص واحد، لكن استدعاءه دون أي شيء آخر في الفقاعة يحدّه قد يسمح للساعة الوهمية بالتقدم أكثر مما قصدت، وهذا بالضبط ما حدث عند اختبار حالة "لم يحدث شيء بعد".
* اختبار يكشف مشكلة في التصميم لا مجرد خطأ، وإصلاح التصميم بدلًا من مدّ اليد إلى قفل.

### مزالق يجب الانتباه إليها

* إذا بقيت goroutine في الفقاعة محجوبة حجبًا دائمًا عند عودة الدالة الجذرية للفقاعة، فإن `synctest.Test` يُفشل الاختبار باعتباره جمودًا بدلًا من أن يتعلّق؛ فتأكد أن goroutines الخلفية تنتهي فعلًا.
* الـ channels والمؤقتات (timers) والعدادات الدورية (tickers) مرتبطة بالفقاعة التي أُنشئت فيها؛ واستخدام أحدها من خارج فقاعته يؤدي إلى panic.
* عمليات إدخال/إخراج الشبكة والملفات الحقيقية ليست حجبًا دائمًا، لذا لا يمكنك قيادتها عبر ساعة `synctest` الوهمية مباشرة؛ فاستعن بشيء مثل `net.Pipe` إن احتجت بديلًا في الذاكرة.
* دالة الاستدعاء الخاصة بـ `time.AfterFunc` تعمل في goroutine خاصة بها، بلا أي من ضمانات المزامنة التي يمنحها الـ channel. فإن كان عليها لمس حالة مشتركة مباشرة، فتلك الحالة تحتاج إلى قفل خاص بها، شأنها شأن أي كود متزامن آخر.

### مواد إضافية

* [مدونة Go: اختبار الكود المتزامن باستخدام testing/synctest](https://go.dev/blog/synctest)
* [توثيق حزمة `testing/synctest`](https://pkg.go.dev/testing/synctest)
* [ملاحظات إصدار Go 1.25](https://go.dev/doc/go1.25)

### ملاحظة حول كيفية كتابة هذا الفصل

هذا أول فصل في الكتاب يُكتب بمساعدة الذكاء الاصطناعي (Claude). أريد أن أكون صريحًا في ذلك، وفي ما عنته "المساعدة" هنا فعلًا، لأنه لم يكن "صِف فصلًا، واحصل على فصل".

بدت العملية شبيهة كثيرًا بحلقة التطوير الموجه بالاختبار التي يعلّمها هذا الكتاب طوال الطريق: ابحث في توثيق `testing/synctest` ومصدره الحقيقيين بدل التخمين، وجرّب برامج صغيرة مؤقتة للتحقق من الادعاءات قبل كتابة كلمة نثر واحدة، وتعامل مع كل ادعاء في التوثيق كشيء يجب التحقق منه بتشغيل `go test -race` حقيقي، لا الثقة به مباشرة. فعدة مزالق في هذا الفصل، وأبرزها تفاعل `Wait()` مع كاشف التسابق، لم توجد هنا إلا لأن اختبارًا فشل فعلًا بطريقة غير متوقعة، عبر جولات عديدة من تشغيله فعلًا، وكان لا بد من التنقيب في السبب قبل تقرير ما نكتب.

تغيّر شكل التصميم نفسه في المنتصف أيضًا. فالمسودة الأولى كان فيها `StdOutAlerter` يكتب مباشرة في `io.Writer`، وهذا بالضبط نوع الشيء الذي لطالما قاومه هذا الكتاب عندما يبدأ الاختبار في الإيلام: [إذا كانت اختباراتك تسبب لك ألمًا، فاستمع إلى تلك الإشارة وفكّر في تصميم كودك](../questions-and-answers/http-handlers-revisited.md). ففعلنا ذلك، وانتهينا إلى النسخة المبنية على الـ channels أعلاه، وإلى فصل أقصر وأفضل بفضل ذلك.

رُوجع كل شيء هنا وحُرّر، واعتُرض عليه أكثر من مرة: عندما لم تكن النبرة سليمة، وعندما كان قسم ما قد تضخم بما يتجاوز ما يستحقه الدرس الفعلي، وعندما لجأ شرح ما إلى المصطلحات الغامضة بينما كان عرض فشل اختبار حقيقي يفي بالغرض أفضل. وإن كان شيء هنا ما زال يُقرأ بشكل غريب، فالمسؤولية مسؤوليتي أنا، لا مسؤولية الأداة.
