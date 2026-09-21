---
title: الوقت
weight: 310
---

# الوقت

**[يمكنك العثور على كل كود هذا الفصل هنا](https://github.com/quii/learn-go-with-tests/tree/main/time)**

يريد منّا مالك المنتج (product owner) أن نوسّع وظائف تطبيق سطر الأوامر الخاص بنا بمساعدة مجموعة من الأشخاص على لعب البوكر تكساس هولدم (Texas-Holdem Poker).

## معلومات بالقدر الكافي عن البوكر

لن تحتاج إلى معرفة الكثير عن البوكر، يكفي أن تعرف أنه في فترات زمنية معينة يجب إبلاغ جميع اللاعبين بقيمة "الرهان الإجباري" (blind) المتزايدة باطراد.

سيساعدنا تطبيقنا على تتبّع موعد ارتفاع قيمة الرهان الإجباري، ومقدارها الجديد.

* عند بدء تشغيله يسأل عن عدد اللاعبين المشاركين. وهذا يحدّد المدة الزمنية قبل أن ترتفع قيمة الرهان الإجباري.
  * توجد مدة زمنية أساسية مقدارها 5 دقائق.
  * وتُضاف دقيقة واحدة لكل لاعب.
  * مثلًا، 6 لاعبين تعني 11 دقيقة للرهان الإجباري.
* بعد انقضاء وقت الرهان الإجباري، ينبغي أن تنبّه اللعبة اللاعبين إلى القيمة الجديدة لرهانهم.
* يبدأ الرهان الإجباري عند 100 شريحة، ثم 200 و400 و600 و1000 و2000، ويستمر في المضاعفة حتى تنتهي اللعبة (وينبغي أن تُنهي وظيفتنا السابقة "Ruth wins" اللعبة أيضًا).

## تذكير بالكود

في الفصل السابق بدأنا العمل على تطبيق سطر الأوامر، وهو يقبل بالفعل أمرًا بالصيغة `{name} wins`. وهذا هو شكل كود `CLI` الحالي، لكن تأكد من التعرّف على بقية الكود أيضًا قبل البدء.

```go
type CLI struct {
	playerStore PlayerStore
	in          *bufio.Scanner
}

func NewCLI(store PlayerStore, in io.Reader) *CLI {
	return &CLI{
		playerStore: store,
		in:          bufio.NewScanner(in),
	}
}

func (cli *CLI) PlayPoker() {
	userInput := cli.readLine()
	cli.playerStore.RecordWin(extractWinner(userInput))
}

func extractWinner(userInput string) string {
	return strings.Replace(userInput, " wins", "", 1)
}

func (cli *CLI) readLine() string {
	cli.in.Scan()
	return cli.in.Text()
}
```

### `time.AfterFunc`

نريد أن نتمكن من جدولة برنامجنا لطباعة قيم الرهان الإجباري عند مدد زمنية معينة حسب عدد اللاعبين.

ولتحديد نطاق ما نحتاج إلى فعله، سنتغاضى الآن عن جزء عدد اللاعبين ونفترض ببساطة أن هناك 5 لاعبين، لذا سنختبر _أن القيمة الجديدة للرهان الإجباري تُطبع كل 10 دقائق_.

وكالعادة، تغطّينا المكتبة القياسية بـ [`func AfterFunc(d Duration, f func()) *Timer`](https://golang.org/pkg/time/#AfterFunc)

> ينتظر `AfterFunc` انقضاء المدة، ثم يستدعي f في goroutine خاص بها. ويعيد `Timer` يمكن استخدامه لإلغاء الاستدعاء عبر method الـ Stop الخاصة به.

### [`time.Duration`](https://golang.org/pkg/time/#Duration)

> يمثّل Duration الوقت المنقضي بين لحظتين كعدد من النانوثانية من النوع int64.

تحتوي حزمة time على عدد من الثوابت التي تتيح لك ضرب تلك النانوثواني لتصبح أكثر قابلية للقراءة في السيناريوهات التي سنتعامل معها.

```
5 * time.Second
```

عندما نستدعي `PlayPoker` سنجدول جميع تنبيهات الرهان.

قد يكون اختبار هذا صعبًا قليلًا. سنريد التحقق من أن كل فترة زمنية مجدولة بقيمة الرهان الإجباري الصحيحة، لكن إن نظرت إلى توقيع `time.AfterFunc` ستجد أن وسيطه الثاني هو الدالة التي سيشغّلها. ولا يمكنك مقارنة الدوال في Go، لذا لن نستطيع اختبار أي دالة أُرسلت. لذلك سنحتاج إلى كتابة نوع من الغلاف (wrapper) حول `time.AfterFunc` يأخذ وقت التشغيل والمقدار المراد طباعته حتى نتمكن من التجسس عليه.

## اكتب الاختبار أولًا

أضف اختبارًا جديدًا إلى مجموعة اختباراتنا.

```go
t.Run("it schedules printing of blind values", func(t *testing.T) {
	in := strings.NewReader("Chris wins\n")
	playerStore := &poker.StubPlayerStore{}
	blindAlerter := &SpyBlindAlerter{}

	cli := poker.NewCLI(playerStore, in, blindAlerter)
	cli.PlayPoker()

	if len(blindAlerter.alerts) != 1 {
		t.Fatal("expected a blind alert to be scheduled")
	}
})
```

ستلاحظ أننا أنشأنا `SpyBlindAlerter` الذي نحاول حقنه في `CLI`، ثم نتحقق من أنه بعد استدعاء `PlayPoker` يكون هناك تنبيه مجدول.

(تذكّر أننا نبدأ بأبسط سيناريو أولًا ثم نكرّر التحسين لاحقًا.)

وهذا هو تعريف `SpyBlindAlerter`

```go
type SpyBlindAlerter struct {
	alerts []struct {
		scheduledAt time.Duration
		amount      int
	}
}

func (s *SpyBlindAlerter) ScheduleAlertAt(duration time.Duration, amount int) {
	s.alerts = append(s.alerts, struct {
		scheduledAt time.Duration
		amount      int
	}{duration, amount})
}
```

## جرّب تشغيل الاختبار

```
./CLI_test.go:32:27: too many arguments in call to poker.NewCLI
	have (*poker.StubPlayerStore, *strings.Reader, *SpyBlindAlerter)
	want (poker.PlayerStore, io.Reader)
```

## اكتب أصغر قدر من الكود ليعمل الاختبار وافحص مخرجاته الفاشلة

أضفنا وسيطًا جديدًا والمترجم يشتكي. _بالمعنى الدقيق_، أصغر قدر من الكود هو جعل `NewCLI` يقبل `*SpyBlindAlerter`، لكن لنغش قليلًا ونعرّف الاعتمادية كواجهة (interface) مباشرة.

```go
type BlindAlerter interface {
	ScheduleAlertAt(duration time.Duration, amount int)
}
```

ثم نضيفه إلى دالة البناء (constructor)

```go
func NewCLI(store PlayerStore, in io.Reader, alerter BlindAlerter) *CLI
```

ستفشل اختباراتك الأخرى الآن لأنها لا تمرّر `BlindAlerter` إلى `NewCLI`.

التجسس على BlindAlerter ليس مهمًا لبقية الاختبارات، لذا أضف في ملف الاختبار

```go
var dummySpyAlerter = &SpyBlindAlerter{}
```

ثم استخدمه في الاختبارات الأخرى لإصلاح مشكلات الترجمة. وبتسميته "dummy" يتّضح لقارئ الاختبار أنه غير مهم.

[> كائنات dummy تُمرَّر هنا وهناك لكن لا تُستخدم فعليًا. وعادةً لا تُستخدم إلا لملء قوائم الوسائط.](https://martinfowler.com/articles/mocksArentStubs.html)

ينبغي أن تترجم الاختبارات الآن، ويفشل اختبارنا الجديد.

```
=== RUN   TestCLI
=== RUN   TestCLI/it_schedules_printing_of_blind_values
--- FAIL: TestCLI (0.00s)
    --- FAIL: TestCLI/it_schedules_printing_of_blind_values (0.00s)
    	CLI_test.go:38: expected a blind alert to be scheduled
```

## اكتب كودًا كافيًا لنجاح الاختبار

سنحتاج إلى إضافة `BlindAlerter` كحقل في `CLI` لنتمكن من الإشارة إليه في method الـ `PlayPoker`.

```go
type CLI struct {
	playerStore PlayerStore
	in          *bufio.Scanner
	alerter     BlindAlerter
}

func NewCLI(store PlayerStore, in io.Reader, alerter BlindAlerter) *CLI {
	return &CLI{
		playerStore: store,
		in:          bufio.NewScanner(in),
		alerter:     alerter,
	}
}
```

ولنجاح الاختبار، يمكننا استدعاء `BlindAlerter` بأي شيء نريده

```go
func (cli *CLI) PlayPoker() {
	cli.alerter.ScheduleAlertAt(5*time.Second, 100)
	userInput := cli.readLine()
	cli.playerStore.RecordWin(extractWinner(userInput))
}
```

بعدها سنريد التحقق من أنه يجدول كل التنبيهات التي نأملها لـ 5 لاعبين

## اكتب الاختبار أولًا

```go
	t.Run("it schedules printing of blind values", func(t *testing.T) {
		in := strings.NewReader("Chris wins\n")
		playerStore := &poker.StubPlayerStore{}
		blindAlerter := &SpyBlindAlerter{}

		cli := poker.NewCLI(playerStore, in, blindAlerter)
		cli.PlayPoker()

		cases := []struct {
			expectedScheduleTime time.Duration
			expectedAmount       int
		}{
			{0 * time.Second, 100},
			{10 * time.Minute, 200},
			{20 * time.Minute, 300},
			{30 * time.Minute, 400},
			{40 * time.Minute, 500},
			{50 * time.Minute, 600},
			{60 * time.Minute, 800},
			{70 * time.Minute, 1000},
			{80 * time.Minute, 2000},
			{90 * time.Minute, 4000},
			{100 * time.Minute, 8000},
		}

		for i, c := range cases {
			t.Run(fmt.Sprintf("%d scheduled for %v", c.expectedAmount, c.expectedScheduleTime), func(t *testing.T) {

				if len(blindAlerter.alerts) <= i {
					t.Fatalf("alert %d was not scheduled %v", i, blindAlerter.alerts)
				}

				alert := blindAlerter.alerts[i]

				amountGot := alert.amount
				if amountGot != c.expectedAmount {
					t.Errorf("got amount %d, want %d", amountGot, c.expectedAmount)
				}

				gotScheduledTime := alert.scheduledAt
				if gotScheduledTime != c.expectedScheduleTime {
					t.Errorf("got scheduled time of %v, want %v", gotScheduledTime, c.expectedScheduleTime)
				}
			})
		}
	})
```

الاختبار القائم على الجدول (table-based test) مناسب هنا ويوضّح متطلباتنا بوضوح. نمرّ على الجدول ونفحص `SpyBlindAlerter` لنرى إن كان التنبيه قد جُدول بالقيم الصحيحة.

## جرّب تشغيل الاختبار

ينبغي أن يظهر لك الكثير من الإخفاقات بهذا الشكل

```
=== RUN   TestCLI
--- FAIL: TestCLI (0.00s)
=== RUN   TestCLI/it_schedules_printing_of_blind_values
    --- FAIL: TestCLI/it_schedules_printing_of_blind_values (0.00s)
=== RUN   TestCLI/it_schedules_printing_of_blind_values/100_scheduled_for_0s
        --- FAIL: TestCLI/it_schedules_printing_of_blind_values/100_scheduled_for_0s (0.00s)
        	CLI_test.go:71: got scheduled time of 5s, want 0s
=== RUN   TestCLI/it_schedules_printing_of_blind_values/200_scheduled_for_10m0s
        --- FAIL: TestCLI/it_schedules_printing_of_blind_values/200_scheduled_for_10m0s (0.00s)
        	CLI_test.go:59: alert 1 was not scheduled [{5000000000 100}]
```

## اكتب كودًا كافيًا لنجاح الاختبار

```go
func (cli *CLI) PlayPoker() {

	blinds := []int{100, 200, 300, 400, 500, 600, 800, 1000, 2000, 4000, 8000}
	blindTime := 0 * time.Second
	for _, blind := range blinds {
		cli.alerter.ScheduleAlertAt(blindTime, blind)
		blindTime = blindTime + 10*time.Minute
	}

	userInput := cli.readLine()
	cli.playerStore.RecordWin(extractWinner(userInput))
}
```

ليس هذا أكثر تعقيدًا كثيرًا مما كان لدينا. نحن الآن نمرّ على مصفوفة من `blinds` ونستدعي المجدول بوقت `blindTime` متزايد.

## إعادة الهيكلة

يمكننا تغليف تنبيهاتنا المجدولة في method فقط لجعل قراءة `PlayPoker` أوضح قليلًا.

```go
func (cli *CLI) PlayPoker() {
	cli.scheduleBlindAlerts()
	userInput := cli.readLine()
	cli.playerStore.RecordWin(extractWinner(userInput))
}

func (cli *CLI) scheduleBlindAlerts() {
	blinds := []int{100, 200, 300, 400, 500, 600, 800, 1000, 2000, 4000, 8000}
	blindTime := 0 * time.Second
	for _, blind := range blinds {
		cli.alerter.ScheduleAlertAt(blindTime, blind)
		blindTime = blindTime + 10*time.Minute
	}
}
```

أخيرًا، تبدو اختباراتنا ثقيلة قليلًا. لدينا نوعان مجهولان (anonymous) من الـ structs يمثّلان الشيء نفسه، وهو `ScheduledAlert`. لنُعد هيكلته (refactor) إلى نوع جديد ثم نصنع بعض دوال المساعدة لمقارنتهما.

```go
type scheduledAlert struct {
	at     time.Duration
	amount int
}

func (s scheduledAlert) String() string {
	return fmt.Sprintf("%d chips at %v", s.amount, s.at)
}

type SpyBlindAlerter struct {
	alerts []scheduledAlert
}

func (s *SpyBlindAlerter) ScheduleAlertAt(at time.Duration, amount int) {
	s.alerts = append(s.alerts, scheduledAlert{at, amount})
}
```

أضفنا method باسم `String()` إلى نوعنا ليُطبع بشكل جميل إذا فشل الاختبار

حدّث اختبارنا ليستخدم نوعنا الجديد

```go
t.Run("it schedules printing of blind values", func(t *testing.T) {
	in := strings.NewReader("Chris wins\n")
	playerStore := &poker.StubPlayerStore{}
	blindAlerter := &SpyBlindAlerter{}

	cli := poker.NewCLI(playerStore, in, blindAlerter)
	cli.PlayPoker()

	cases := []scheduledAlert{
		{0 * time.Second, 100},
		{10 * time.Minute, 200},
		{20 * time.Minute, 300},
		{30 * time.Minute, 400},
		{40 * time.Minute, 500},
		{50 * time.Minute, 600},
		{60 * time.Minute, 800},
		{70 * time.Minute, 1000},
		{80 * time.Minute, 2000},
		{90 * time.Minute, 4000},
		{100 * time.Minute, 8000},
	}

	for i, want := range cases {
		t.Run(fmt.Sprint(want), func(t *testing.T) {

			if len(blindAlerter.alerts) <= i {
				t.Fatalf("alert %d was not scheduled %v", i, blindAlerter.alerts)
			}

			got := blindAlerter.alerts[i]
			assertScheduledAlert(t, got, want)
		})
	}
})
```

نفّذ `assertScheduledAlert` بنفسك.

أمضينا وقتًا لا بأس به هنا في كتابة الاختبارات، ولم نكن مطيعين تمامًا إذ لم ندمجها في تطبيقنا. لنعالج ذلك قبل أن نكدّس المزيد من المتطلبات.

جرّب تشغيل التطبيق ولن يترجم، وسيشتكي من عدم كفاية الوسائط لـ `NewCLI`.

لننشئ تنفيذًا (implementation) لـ `BlindAlerter` يمكننا استخدامه في تطبيقنا.

أنشئ `blind_alerter.go` وانقل إليه واجهة `BlindAlerter`، ثم أضف الأشياء الجديدة أدناه

```go
package poker

import (
	"fmt"
	"os"
	"time"
)

type BlindAlerter interface {
	ScheduleAlertAt(duration time.Duration, amount int)
}

type BlindAlerterFunc func(duration time.Duration, amount int)

func (a BlindAlerterFunc) ScheduleAlertAt(duration time.Duration, amount int) {
	a(duration, amount)
}

func StdOutAlerter(duration time.Duration, amount int) {
	time.AfterFunc(duration, func() {
		fmt.Fprintf(os.Stdout, "Blind is now %d\n", amount)
	})
}
```

تذكّر أن أي _نوع_ يمكنه تنفيذ واجهة، وليس الـ `structs` فقط. وإذا كنت تصنع مكتبة تكشف واجهة عرّفت فيها دالة واحدة، فمن الأساليب الشائعة أيضًا كشف نوع `MyInterfaceFunc`.

سيكون هذا النوع دالة `func` تنفّذ واجهتك أيضًا. وبهذه الطريقة يصبح لدى مستخدمي واجهتك خيار تنفيذها بدالة فقط، بدلًا من إنشاء نوع `struct` فارغ.

ثم ننشئ الدالة `StdOutAlerter` التي تملك التوقيع نفسه، ونستخدم `time.AfterFunc` لجدولتها للطباعة إلى `os.Stdout`.

حدّث `main` حيث ننشئ `NewCLI` لترى ذلك عمليًا

```go
poker.NewCLI(store, os.Stdin, poker.BlindAlerterFunc(poker.StdOutAlerter)).PlayPoker()
```

قبل التشغيل، قد ترغب في تغيير مقدار زيادة `blindTime` في `CLI` ليكون 10 ثوانٍ بدلًا من 10 دقائق، فقط لترى ذلك عمليًا.

ينبغي أن تراه يطبع قيم الرهان كما نتوقع كل 10 ثوانٍ. ولاحظ كيف يمكنك مع ذلك كتابة `Shaun wins` في سطر الأوامر فيوقف البرنامج كما نتوقع.

لن تُلعب اللعبة دائمًا مع 5 أشخاص، لذا نحتاج إلى مطالبة المستخدم بإدخال عدد اللاعبين قبل بدء اللعبة.

## اكتب الاختبار أولًا

للتحقق من أننا نطالب بإدخال عدد اللاعبين، سنريد تسجيل ما يُكتب إلى StdOut. فعلنا ذلك بضع مرات حتى الآن، ونعرف أن `os.Stdout` هو `io.Writer`، لذا يمكننا فحص ما يُكتب إذا استخدمنا حقن الاعتماديات لتمرير `bytes.Buffer` في اختبارنا ورؤية ما سيكتبه كودنا.

لا تهمّنا بقية المتعاونين (collaborators) في هذا الاختبار بعد، لذا أنشأنا بعض الـ dummies في ملف اختبارنا.

ينبغي أن نكون حذرين قليلًا فلدينا الآن 4 اعتماديات لـ `CLI`، ويبدو أن مسؤولياته بدأت تكثر عن اللزوم. لنرضَ بالأمر في الوقت الحالي ونرَ إن كانت إعادة هيكلة ما ستظهر بينما نضيف هذه الوظيفة الجديدة.

```go
var dummyBlindAlerter = &SpyBlindAlerter{}
var dummyPlayerStore = &poker.StubPlayerStore{}
var dummyStdIn = &bytes.Buffer{}
var dummyStdOut = &bytes.Buffer{}
```

وهذا اختبارنا الجديد

```go
t.Run("it prompts the user to enter the number of players", func(t *testing.T) {
	stdout := &bytes.Buffer{}
	cli := poker.NewCLI(dummyPlayerStore, dummyStdIn, stdout, dummyBlindAlerter)
	cli.PlayPoker()

	got := stdout.String()
	want := "Please enter the number of players: "

	if got != want {
		t.Errorf("got %q, want %q", got, want)
	}
})
```

نمرّر ما سيكون `os.Stdout` في `main` ونرى ما يُكتب.

## جرّب تشغيل الاختبار

```
./CLI_test.go:38:27: too many arguments in call to poker.NewCLI
	have (*poker.StubPlayerStore, *bytes.Buffer, *bytes.Buffer, *SpyBlindAlerter)
	want (poker.PlayerStore, io.Reader, poker.BlindAlerter)
```

## اكتب أصغر قدر من الكود ليعمل الاختبار وافحص مخرجاته الفاشلة

لدينا اعتمادية جديدة لذا سيتعيّن علينا تحديث `NewCLI`

```go
func NewCLI(store PlayerStore, in io.Reader, out io.Writer, alerter BlindAlerter) *CLI
```

الآن سيفشل ترجمة الاختبارات _الأخرى_ لأنها لا تمرّر `io.Writer` إلى `NewCLI`.

أضف `dummyStdOut` لبقية الاختبارات.

ينبغي أن يفشل الاختبار الجديد هكذا

```
=== RUN   TestCLI
--- FAIL: TestCLI (0.00s)
=== RUN   TestCLI/it_prompts_the_user_to_enter_the_number_of_players
    --- FAIL: TestCLI/it_prompts_the_user_to_enter_the_number_of_players (0.00s)
    	CLI_test.go:46: got '', want 'Please enter the number of players: '
FAIL
```

## اكتب كودًا كافيًا لنجاح الاختبار

نحتاج إلى إضافة اعتماديتنا الجديدة إلى `CLI` لنتمكن من الإشارة إليها في `PlayPoker`

```go
type CLI struct {
	playerStore PlayerStore
	in          *bufio.Scanner
	out         io.Writer
	alerter     BlindAlerter
}

func NewCLI(store PlayerStore, in io.Reader, out io.Writer, alerter BlindAlerter) *CLI {
	return &CLI{
		playerStore: store,
		in:          bufio.NewScanner(in),
		out:         out,
		alerter:     alerter,
	}
}
```

ثم أخيرًا يمكننا كتابة رسالة المطالبة في بداية اللعبة

```go
func (cli *CLI) PlayPoker() {
	fmt.Fprint(cli.out, "Please enter the number of players: ")
	cli.scheduleBlindAlerts()
	userInput := cli.readLine()
	cli.playerStore.RecordWin(extractWinner(userInput))
}
```

## إعادة الهيكلة

لدينا نص مكرر لرسالة المطالبة، وينبغي استخراجه إلى ثابت

```go
const PlayerPrompt = "Please enter the number of players: "
```

استخدمه في كود الاختبار وفي `CLI` معًا.

الآن نحتاج إلى إرسال رقم واستخراجه. والطريقة الوحيدة لمعرفة إن كان قد أحدث الأثر المطلوب هي رؤية التنبيهات التي جُدولت.

## اكتب الاختبار أولًا

```go
t.Run("it prompts the user to enter the number of players", func(t *testing.T) {
	stdout := &bytes.Buffer{}
	in := strings.NewReader("7\n")
	blindAlerter := &SpyBlindAlerter{}

	cli := poker.NewCLI(dummyPlayerStore, in, stdout, blindAlerter)
	cli.PlayPoker()

	got := stdout.String()
	want := poker.PlayerPrompt

	if got != want {
		t.Errorf("got %q, want %q", got, want)
	}

	cases := []scheduledAlert{
		{0 * time.Second, 100},
		{12 * time.Minute, 200},
		{24 * time.Minute, 300},
		{36 * time.Minute, 400},
	}

	for i, want := range cases {
		t.Run(fmt.Sprint(want), func(t *testing.T) {

			if len(blindAlerter.alerts) <= i {
				t.Fatalf("alert %d was not scheduled %v", i, blindAlerter.alerts)
			}

			got := blindAlerter.alerts[i]
			assertScheduledAlert(t, got, want)
		})
	}
})
```

أخ! تغييرات كثيرة.

* أزلنا الـ dummy الخاص بـ StdIn وأرسلنا بدلًا منه نسخة mock تمثّل إدخال المستخدم للرقم 7
* وأزلنا أيضًا الـ dummy الخاص بمُنبّه الرهان لنرى أن عدد اللاعبين أثّر على الجدولة
* ونختبر التنبيهات التي جُدولت

## جرّب تشغيل الاختبار

ينبغي أن يظل الاختبار يترجم ويفشل مخبرًا بأن الأوقات المجدولة خاطئة، لأننا كتبنا الكود بترميز ثابت (hard-coded) ليعتمد على وجود 5 لاعبين

```
=== RUN   TestCLI
--- FAIL: TestCLI (0.00s)
=== RUN   TestCLI/it_prompts_the_user_to_enter_the_number_of_players
    --- FAIL: TestCLI/it_prompts_the_user_to_enter_the_number_of_players (0.00s)
=== RUN   TestCLI/it_prompts_the_user_to_enter_the_number_of_players/100_chips_at_0s
        --- PASS: TestCLI/it_prompts_the_user_to_enter_the_number_of_players/100_chips_at_0s (0.00s)
=== RUN   TestCLI/it_prompts_the_user_to_enter_the_number_of_players/200_chips_at_12m0s
```

## اكتب كودًا كافيًا لنجاح الاختبار

تذكّر أننا أحرار في اقتراف ما يلزم من الذنوب لنجعل هذا يعمل. وفور أن يصبح لدينا برنامج يعمل، يمكننا العمل على إعادة هيكلة الفوضى التي نحن على وشك صنعها!

```go
func (cli *CLI) PlayPoker() {
	fmt.Fprint(cli.out, PlayerPrompt)

	numberOfPlayers, _ := strconv.Atoi(cli.readLine())

	cli.scheduleBlindAlerts(numberOfPlayers)

	userInput := cli.readLine()
	cli.playerStore.RecordWin(extractWinner(userInput))
}

func (cli *CLI) scheduleBlindAlerts(numberOfPlayers int) {
	blindIncrement := time.Duration(5+numberOfPlayers) * time.Minute

	blinds := []int{100, 200, 300, 400, 500, 600, 800, 1000, 2000, 4000, 8000}
	blindTime := 0 * time.Second
	for _, blind := range blinds {
		cli.alerter.ScheduleAlertAt(blindTime, blind)
		blindTime = blindTime + blindIncrement
	}
}
```

* نقرأ `numberOfPlayersInput` في نص
* نستخدم `cli.readLine()` للحصول على مدخلات المستخدم ثم نستدعي `Atoi` لتحويلها إلى عدد صحيح، مع تجاهل أي سيناريوهات خطأ. وسنحتاج إلى كتابة اختبار لذلك السيناريو لاحقًا.
* ومن هنا نغيّر `scheduleBlindAlerts` ليقبل عدد اللاعبين. ثم نحسب زمن `blindIncrement` لنضيفه إلى `blindTime` أثناء مرورنا على مقادير الرهان

وبينما أُصلح اختبارنا الجديد، فشل كثير من الاختبارات الأخرى لأن نظامنا الآن لا يعمل إلا إذا بدأت اللعبة بإدخال المستخدم لرقم. ستحتاج إلى إصلاح الاختبارات بتغيير مدخلات المستخدم بحيث يُضاف رقم متبوع بسطر جديد (وهذا يبرز عيوبًا أخرى في أسلوبنا الحالي).

## إعادة الهيكلة

كل هذا يبدو سيئًا بعض الشيء، أليس كذلك؟ لنُنصت إلى **اختباراتنا**.

* لتتمكن من اختبار أننا نجدول بعض التنبيهات، أعددنا 4 اعتماديات مختلفة. ومتى كان لديك اعتماديات كثيرة على _شيء_ في نظامك، فهذا يعني أنه يفعل أكثر مما ينبغي. ونرى ذلك بصريًا في مدى تكدّس اختبارنا.
* وهو يبدو لي كما لو أننا **بحاجة إلى تجريد أنظف (abstraction) بين قراءة مدخلات المستخدم ومنطق العمل (business logic) الذي نريد تنفيذه**.
* وسيكون الاختبار الأفضل: _بالنظر إلى مدخلات المستخدم هذه، هل نستدعي نوعًا جديدًا `Game` بعدد اللاعبين الصحيح_.
* ثم نستخرج اختبار الجدولة إلى اختبارات النوع الجديد `Game`.

يمكننا إعادة الهيكلة نحو `Game` أولًا، وينبغي أن يظل اختبارنا ناجحًا. وبعد أن نُجري التغييرات البنيوية التي نريدها، يمكننا التفكير في كيفية إعادة هيكلة الاختبارات لتعكس فصل الاهتمامات (separation of concerns) الجديد

تذكّر عند إجراء تغييرات في إعادة الهيكلة أن تبقيها أصغر ما يمكن وأن تعيد تشغيل الاختبارات باستمرار.

جرّبه بنفسك أولًا. فكّر في الحدود التي سيقدّمها `Game` وما ينبغي أن يفعله `CLI`.

و**لا** تغيّر الآن الواجهة الخارجية لـ `NewCLI`، لأننا لا نريد تغيير كود الاختبار وكود العميل في الوقت نفسه، فهذا أكثر مما نستطيع التعامل معه وقد ننتهي بكسر الأشياء.

وهذا ما توصلت إليه:

```go
// game.go
type Game struct {
	alerter BlindAlerter
	store   PlayerStore
}

func (p *Game) Start(numberOfPlayers int) {
	blindIncrement := time.Duration(5+numberOfPlayers) * time.Minute

	blinds := []int{100, 200, 300, 400, 500, 600, 800, 1000, 2000, 4000, 8000}
	blindTime := 0 * time.Second
	for _, blind := range blinds {
		p.alerter.ScheduleAlertAt(blindTime, blind)
		blindTime = blindTime + blindIncrement
	}
}

func (p *Game) Finish(winner string) {
	p.store.RecordWin(winner)
}

// cli.go
type CLI struct {
	in   *bufio.Scanner
	out  io.Writer
	game *Game
}

func NewCLI(store PlayerStore, in io.Reader, out io.Writer, alerter BlindAlerter) *CLI {
	return &CLI{
		in:  bufio.NewScanner(in),
		out: out,
		game: &Game{
			alerter: alerter,
			store:   store,
		},
	}
}

const PlayerPrompt = "Please enter the number of players: "

func (cli *CLI) PlayPoker() {
	fmt.Fprint(cli.out, PlayerPrompt)

	numberOfPlayersInput := cli.readLine()
	numberOfPlayers, _ := strconv.Atoi(strings.Trim(numberOfPlayersInput, "\n"))

	cli.game.Start(numberOfPlayers)

	winnerInput := cli.readLine()
	winner := extractWinner(winnerInput)

	cli.game.Finish(winner)
}

func extractWinner(userInput string) string {
	return strings.Replace(userInput, " wins\n", "", 1)
}

func (cli *CLI) readLine() string {
	cli.in.Scan()
	return cli.in.Text()
}
```

من منظور "المجال":

* نريد أن نبدأ `Game`، مع تحديد عدد المشاركين
* ونريد أن ننهي `Game`، معلنين الفائز

ويغلّف لنا نوع `Game` الجديد ذلك.

بهذا التغيير مرّرنا `BlindAlerter` و`PlayerStore` إلى `Game` لأنه أصبح مسؤولًا عن التنبيه وتخزين النتائج.

أصبح `CLI` الآن مسؤولًا فقط عن:

* بناء `Game` باعتمادياته الحالية (وسنعيد هيكلة ذلك بعد قليل)
* تفسير مدخلات المستخدم كاستدعاءات لـ methods في `Game`

نريد أن نتجنب إجراء إعادات هيكلة "كبيرة" تُبقينا في حالة اختبارات فاشلة لفترات طويلة، لأن ذلك يزيد احتمالات الأخطاء. (وإن كنت تعمل في فريق كبير أو موزّع فهذا أمر مهم بشكل خاص)

أول شيء سنفعله هو إعادة هيكلة `Game` لنحقنه في `CLI`. وسنُجري أصغر التغييرات في اختباراتنا لتسهيل ذلك، ثم نرى كيف يمكن تقسيم الاختبارات إلى محورين: تحليل مدخلات المستخدم وإدارة اللعبة.

كل ما نحتاج إلى فعله الآن هو تغيير `NewCLI`

```go
func NewCLI(in io.Reader, out io.Writer, game *Game) *CLI {
	return &CLI{
		in:   bufio.NewScanner(in),
		out:  out,
		game: game,
	}
}
```

يبدو هذا تحسينًا بالفعل. أصبح لدينا اعتماديات أقل، و_قائمة اعتمادياتنا تعكّس هدفنا التصميمي العام_: أن يهتم CLI بالإدخال والإخراج ويفوّض الإجراءات الخاصة باللعبة إلى `Game`.

إذا جرّبت الترجمة ستجد مشكلات. ينبغي أن تكون قادرًا على إصلاحها بنفسك. ولا تقلق بشأن صنع أي mocks لـ `Game` الآن؛ فقط هيّئ كائنات `Game` _حقيقية_ ليعمل كل شيء وتنجح الاختبارات.

ولتفعل ذلك ستحتاج إلى إنشاء دالة بناء

```go
func NewGame(alerter BlindAlerter, store PlayerStore) *Game {
	return &Game{
		alerter: alerter,
		store:   store,
	}
}
```

وهذا مثال لأحد إعدادات الاختبارات التي نصلحها

```go
stdout := &bytes.Buffer{}
in := strings.NewReader("7\n")
blindAlerter := &SpyBlindAlerter{}
game := poker.NewGame(blindAlerter, dummyPlayerStore)

cli := poker.NewCLI(in, stdout, game)
cli.PlayPoker()
```

لن يتطلب إصلاح الاختبارات جهدًا كبيرًا لتعود كلها ناجحة (وهذا هو المقصد!)، لكن تأكد من إصلاح `main.go` أيضًا قبل المرحلة التالية.

```go
// main.go
game := poker.NewGame(poker.BlindAlerterFunc(poker.StdOutAlerter), store)
cli := poker.NewCLI(os.Stdin, os.Stdout, game)
cli.PlayPoker()
```

وبعد أن استخرجنا `Game`، ينبغي أن ننقل التحققات الخاصة باللعبة إلى اختبارات منفصلة عن CLI.

وهذا مجرد تدريب على نسخ اختبارات `CLI` لكن باعتماديات أقل

```go
func TestGame_Start(t *testing.T) {
	t.Run("schedules alerts on game start for 5 players", func(t *testing.T) {
		blindAlerter := &poker.SpyBlindAlerter{}
		game := poker.NewGame(blindAlerter, dummyPlayerStore)

		game.Start(5)

		cases := []poker.ScheduledAlert{
			{At: 0 * time.Second, Amount: 100},
			{At: 10 * time.Minute, Amount: 200},
			{At: 20 * time.Minute, Amount: 300},
			{At: 30 * time.Minute, Amount: 400},
			{At: 40 * time.Minute, Amount: 500},
			{At: 50 * time.Minute, Amount: 600},
			{At: 60 * time.Minute, Amount: 800},
			{At: 70 * time.Minute, Amount: 1000},
			{At: 80 * time.Minute, Amount: 2000},
			{At: 90 * time.Minute, Amount: 4000},
			{At: 100 * time.Minute, Amount: 8000},
		}

		checkSchedulingCases(cases, t, blindAlerter)
	})

	t.Run("schedules alerts on game start for 7 players", func(t *testing.T) {
		blindAlerter := &poker.SpyBlindAlerter{}
		game := poker.NewGame(blindAlerter, dummyPlayerStore)

		game.Start(7)

		cases := []poker.ScheduledAlert{
			{At: 0 * time.Second, Amount: 100},
			{At: 12 * time.Minute, Amount: 200},
			{At: 24 * time.Minute, Amount: 300},
			{At: 36 * time.Minute, Amount: 400},
		}

		checkSchedulingCases(cases, t, blindAlerter)
	})

}

func TestGame_Finish(t *testing.T) {
	store := &poker.StubPlayerStore{}
	game := poker.NewGame(dummyBlindAlerter, store)
	winner := "Ruth"

	game.Finish(winner)
	poker.AssertPlayerWin(t, store, winner)
}
```

أصبحت النية وراء ما يحدث عند بدء لعبة بوكر أوضح بكثير.

وتأكد أيضًا من نقل اختبار نهاية اللعبة.

وفور رضانا عن نقل اختبارات منطق اللعبة، يمكننا تبسيط اختبارات CLI لتعكس مسؤولياته المقصودة بوضوح أكبر

* معالجة مدخلات المستخدم واستدعاء methods الخاصة بـ `Game` عند الاقتضاء
* إرسال المخرجات
* والأهم أنه لا يعرف شيئًا عن الكيفية الفعلية لعمل الألعاب

ولنفعل ذلك سيكون علينا جعل `CLI` لا يعتمد على نوع `Game` الملموس، بل يقبل واجهة فيها `Start(numberOfPlayers)` و`Finish(winner)`. ثم ننشئ spy من ذلك النوع ونتحقق من إجراء الاستدعاءات الصحيحة.

وهنا ندرك أن التسمية محرجة أحيانًا. أعد تسمية `Game` إلى `TexasHoldem` (فهذا هو _نوع_ اللعبة التي نلعبها)، وستُسمى الواجهة الجديدة `Game`. وهذا يبقى وفيًّا لفكرة أن CLI غافل عن اللعبة الفعلية التي نلعبها وعمّا يحدث عند `Start` و`Finish`.

```go
type Game interface {
	Start(numberOfPlayers int)
	Finish(winner string)
}
```

استبدل كل الإشارات إلى `*Game` داخل `CLI` بالواجهة الجديدة `Game`. وكالعادة، أعِد تشغيل الاختبارات باستمرار للتأكد من أن كل شيء ناجح أثناء إعادة الهيكلة.

وبعد أن فصلنا `CLI` عن `TexasHoldem` يمكننا استخدام الـ spies للتحقق من استدعاء `Start` و`Finish` في الوقت الذي نتوقعه وبالوسائط الصحيحة.

أنشئ spy ينفّذ واجهة `Game`

```go
type GameSpy struct {
	StartedWith  int
	FinishedWith string
}

func (g *GameSpy) Start(numberOfPlayers int) {
	g.StartedWith = numberOfPlayers
}

func (g *GameSpy) Finish(winner string) {
	g.FinishedWith = winner
}
```

استبدل أي اختبار لـ `CLI` يختبر منطقًا خاصًا باللعبة بتحققات من كيفية استدعاء `GameSpy`. وبذلك تعكس اختباراتنا مسؤوليات CLI بوضوح.

وهذا مثال لأحد الاختبارات التي نصلحها؛ جرّب إكمال البقية بنفسك وافحص الكود المصدري إن واجهت صعوبة.

```go
	t.Run("it prompts the user to enter the number of players and starts the game", func(t *testing.T) {
		stdout := &bytes.Buffer{}
		in := strings.NewReader("7\n")
		game := &GameSpy{}

		cli := poker.NewCLI(in, stdout, game)
		cli.PlayPoker()

		gotPrompt := stdout.String()
		wantPrompt := poker.PlayerPrompt

		if gotPrompt != wantPrompt {
			t.Errorf("got %q, want %q", gotPrompt, wantPrompt)
		}

		if game.StartedWith != 7 {
			t.Errorf("wanted Start called with 7 but got %d", game.StartedWith)
		}
	})
```

وبعد أن أصبح لدينا فصل واضح للاهتمامات، ينبغي أن يصبح فحص الحالات الحدّية (edge cases) المتعلقة بالإدخال والإخراج في `CLI` أسهل.

نحتاج إلى معالجة سيناريو إدخال المستخدم قيمة غير رقمية عند مطالبتنا له بعدد اللاعبين:

ينبغي ألا يبدأ كودنا اللعبة، وأن يطبع خطأً مفيدًا للمستخدم ثم يخرج.

## اكتب الاختبار أولًا

سنبدأ بالتأكد من أن اللعبة لا تبدأ

```go
t.Run("it prints an error when a non numeric value is entered and does not start the game", func(t *testing.T) {
	stdout := &bytes.Buffer{}
	in := strings.NewReader("Pies\n")
	game := &GameSpy{}

	cli := poker.NewCLI(in, stdout, game)
	cli.PlayPoker()

	if game.StartCalled {
		t.Errorf("game should not have started")
	}
})
```

ستحتاج إلى إضافة حقل `StartCalled` إلى `GameSpy` لا يُضبط إلا إذا استُدعيت `Start`

## جرّب تشغيل الاختبار

```
=== RUN   TestCLI/it_prints_an_error_when_a_non_numeric_value_is_entered_and_does_not_start_the_game
    --- FAIL: TestCLI/it_prints_an_error_when_a_non_numeric_value_is_entered_and_does_not_start_the_game (0.00s)
        CLI_test.go:62: game should not have started
```

## اكتب كودًا كافيًا لنجاح الاختبار

حول مكان استدعائنا لـ `Atoi` نحتاج فقط إلى فحص الخطأ

```go
numberOfPlayers, err := strconv.Atoi(cli.readLine())

if err != nil {
	return
}
```

بعدها نحتاج إلى إخبار المستخدم بما فعله خطأً، لذا سنتحقق من المطبوع إلى `stdout`.

## اكتب الاختبار أولًا

لقد تحققنا سابقًا مما طُبع إلى `stdout`، لذا يمكننا نسخ ذلك الكود الآن

```go
gotPrompt := stdout.String()

wantPrompt := poker.PlayerPrompt + "you're so silly"

if gotPrompt != wantPrompt {
	t.Errorf("got %q, want %q", gotPrompt, wantPrompt)
}
```

نحن نخزّن _كل_ ما يُكتب إلى stdout، لذا ما زلنا نتوقع `poker.PlayerPrompt`. ثم نتحقق فقط من طباعة شيء إضافي. ولا يهمّنا كثيرًا النص الدقيق الآن؛ سنعالجه عند إعادة الهيكلة.

## جرّب تشغيل الاختبار

```
=== RUN   TestCLI/it_prints_an_error_when_a_non_numeric_value_is_entered_and_does_not_start_the_game
    --- FAIL: TestCLI/it_prints_an_error_when_a_non_numeric_value_is_entered_and_does_not_start_the_game (0.00s)
        CLI_test.go:70: got 'Please enter the number of players: ', want 'Please enter the number of players: you're so silly'
```

## اكتب كودًا كافيًا لنجاح الاختبار

غيّر كود معالجة الأخطاء

```go
if err != nil {
	fmt.Fprint(cli.out, "you're so silly")
	return
}
```

## إعادة الهيكلة

الآن أعد هيكلة الرسالة إلى ثابت مثل `PlayerPrompt`

```go
wantPrompt := poker.PlayerPrompt + poker.BadPlayerInputErrMsg
```

ثم ضع رسالة أنسب

```go
const BadPlayerInputErrMsg = "Bad value received for number of players, please try again with a number"
```

أخيرًا، أصبح فحصنا لما أُرسل إلى `stdout` مطوّلًا جدًا، لنكتب دالة تحقق لتنظيفه.

```go
func assertMessagesSentToUser(t testing.TB, stdout *bytes.Buffer, messages ...string) {
	t.Helper()
	want := strings.Join(messages, "")
	got := stdout.String()
	if got != want {
		t.Errorf("got %q sent to stdout but expected %+v", got, messages)
	}
}
```

استخدام صياغة الوسائط المتغيرة (`...string`) مفيد هنا لأننا نحتاج إلى التحقق من أعداد متغيّرة من الرسائل.

استخدم دالة المساعدة هذه في كلا الاختبارين حيث نتحقق من الرسائل المرسلة إلى المستخدم.

هناك عدد من الاختبارات التي قد تستفيد من بعض دوال `assertX`، لذا تدرّب على إعادة الهيكلة بتنظيف اختباراتنا لتُقرأ بشكل جميل.

خُذ وقتك وفكّر في قيمة بعض الاختبارات التي أنتجناها. تذكّر أننا لا نريد اختبارات أكثر من اللازم؛ هل يمكنك إعادة هيكلة بعضها أو حذفها _وأن تبقى واثقًا أن كل شيء يعمل_؟

وهذا ما توصلت إليه

```go
func TestCLI(t *testing.T) {

	t.Run("start game with 3 players and finish game with 'Chris' as winner", func(t *testing.T) {
		game := &GameSpy{}
		stdout := &bytes.Buffer{}

		in := userSends("3", "Chris wins")
		cli := poker.NewCLI(in, stdout, game)

		cli.PlayPoker()

		assertMessagesSentToUser(t, stdout, poker.PlayerPrompt)
		assertGameStartedWith(t, game, 3)
		assertFinishCalledWith(t, game, "Chris")
	})

	t.Run("start game with 8 players and record 'Cleo' as winner", func(t *testing.T) {
		game := &GameSpy{}

		in := userSends("8", "Cleo wins")
		cli := poker.NewCLI(in, dummyStdOut, game)

		cli.PlayPoker()

		assertGameStartedWith(t, game, 8)
		assertFinishCalledWith(t, game, "Cleo")
	})

	t.Run("it prints an error when a non numeric value is entered and does not start the game", func(t *testing.T) {
		game := &GameSpy{}

		stdout := &bytes.Buffer{}
		in := userSends("pies")

		cli := poker.NewCLI(in, stdout, game)
		cli.PlayPoker()

		assertGameNotStarted(t, game)
		assertMessagesSentToUser(t, stdout, poker.PlayerPrompt, poker.BadPlayerInputErrMsg)
	})
}
```

تعكس الاختبارات الآن القدرات الرئيسية لـ CLI؛ فهو قادر على قراءة مدخلات المستخدم من حيث عدد اللاعبين ومن فاز، ويتعامل مع إدخال قيمة سيئة لعدد اللاعبين. وبهذا يتضح للقارئ ما يفعله `CLI` وما لا يفعله أيضًا.

ماذا يحدث إذا أدخل المستخدم `Lloyd is a killer` بدلًا من `Ruth wins`؟

أنهِ هذا الفصل بكتابة اختبار لهذا السيناريو وجعله ينجح.

## الخلاصة

### مراجعة سريعة للمشروع

على مدى الفصول الخمسة الماضية طوّرنا بالاختبارات (TDD) قدرًا لا بأس به من الكود، خطوة خطوة

* لدينا تطبيقان: تطبيق سطر أوامر وخادم ويب.
* ويعتمد هذان التطبيقان على `PlayerStore` لتسجيل الفائزين
* ويستطيع خادم الويب أيضًا عرض جدول ترتيب بمن يفوز بأكثر عدد من الألعاب
* ويساعد تطبيق سطر الأوامر اللاعبين على لعب البوكر بتتبّع قيمة الرهان الإجباري الحالية.

### time.Afterfunc

طريقة مفيدة جدًا لجدولة استدعاء دالة بعد مدة محددة. ويستحق الأمر أن تستثمر وقتك في [الاطلاع على توثيق `time`](https://golang.org/pkg/time/) ففيه الكثير من الدوال والـ methods التي توفّر عليك الوقت.

ومن المفضلة لدي

* تُرجع `time.After(duration)` قناة `chan Time` عند انقضاء المدة. فإن أردت فعل شيء _بعد_ وقت محدد، فقد يفيدك هذا.
* تُرجع `time.NewTicker(duration)` كائن `Ticker` يشبه ما سبق في أنه يعيد قناة، لكن هذا "يدق" كل مدة، لا مرة واحدة فقط. وهذا مفيد جدًا إذا أردت تنفيذ كود كل `N duration`.

### أمثلة أخرى على الفصل الجيد للاهتمامات

_عمومًا_، من الممارسات الجيدة فصل مسؤوليات التعامل مع مدخلات المستخدم واستجاباته عن كود المجال. وترى ذلك هنا في تطبيق سطر الأوامر وكذلك في خادم الويب.

أصبحت اختباراتنا فوضوية. كان لدينا تحققات كثيرة جدًا (افحص هذا المدخل، وجدول هذه التنبيهات، إلخ) واعتماديات كثيرة جدًا. وقد رأينا بأعيننا أنها مكدّسة؛ وإنه **لمهم جدًا أن تنصت إلى اختباراتك**.

* إن بدت اختباراتك فوضوية، جرّب إعادة هيكلتها.
* وإن فعلت ذلك وظلّت فوضوية، فالمرجّح جدًا أنها تشير إلى خلل في تصميمك
* وهذه إحدى نقاط القوة الحقيقية للاختبارات.

ورغم أن الاختبارات وكود الإنتاج كانا مكدّسين قليلًا، فقد استطعنا إعادة الهيكلة بحرية مدعومين باختباراتنا.

تذكّر أن تأخذ خطوات صغيرة دائمًا عند الوقوع في هذه المواقف، وأن تعيد تشغيل الاختبارات بعد كل تغيير.

كان سيكون خطيرًا إعادة هيكلة كود الاختبار _وكود الإنتاج_ في الوقت نفسه، لذا أعدنا هيكلة كود الإنتاج أولًا (ففي الحالة الراهنة لم نكن نستطيع تحسين الاختبارات كثيرًا) دون تغيير واجهته لنعتمد على اختباراتنا قدر الإمكان أثناء التغيير. _ثم_ أعدنا هيكلة الاختبارات بعد أن تحسّن التصميم.

وبعد إعادة الهيكلة عكست قائمة الاعتماديات هدفنا التصميمي. وهذه فائدة أخرى لحقن الاعتماديات إذ يوثّق النية غالبًا. فعندما تعتمد على متغيرات عامة (global variables) تصبح المسؤوليات غير واضحة أبدًا.

## مثال على دالة تنفّذ واجهة

عندما تعرّف واجهة فيها method واحدة، قد ترغب في التفكير في تعريف نوع `MyInterfaceFunc` مكمّل لها ليتمكن المستخدمون من تنفيذ واجهتك بدالة فقط.

```go
type BlindAlerter interface {
	ScheduleAlertAt(duration time.Duration, amount int)
}

// BlindAlerterFunc allows you to implement BlindAlerter with a function
type BlindAlerterFunc func(duration time.Duration, amount int)

// ScheduleAlertAt is BlindAlerterFunc implementation of BlindAlerter
func (a BlindAlerterFunc) ScheduleAlertAt(duration time.Duration, amount int) {
	a(duration, amount)
}
```

وبفعل ذلك، يستطيع مستخدمو مكتبتك تنفيذ واجهتك بدالة فقط. ويمكنهم استخدام [تحويل النوع (Type Conversion)](https://go.dev/tour/basics/13) لتحويل دالتهم إلى `BlindAlerterFunc` ثم استخدامها كـ BlindAlerter (لأن `BlindAlerterFunc` ينفّذ `BlindAlerter`).

```go
game := poker.NewTexasHoldem(poker.BlindAlerterFunc(poker.StdOutAlerter), store)
```

والنقطة الأعم هنا هي أنه في Go يمكنك إضافة methods إلى _الأنواع_، وليس إلى الـ structs فقط. وهذه ميزة قوية جدًا، ويمكنك استخدامها لتنفيذ الواجهات بطرق أكثر ملاءمة.

وتأمّل أنك لا تستطيع تعريف أنواع من الدوال فحسب، بل يمكنك أيضًا تعريف أنواع حول أنواع أخرى لتضيف إليها methods.

```go
type Blog map[string]string

func (b Blog) ServeHTTP(w http.ResponseWriter, r *http.Request) {
	fmt.Fprintln(w, b[r.URL.Path])
}
```

هنا أنشأنا معالج HTTP ينفّذ "مدونة" بسيطة جدًا، إذ يستخدم مسارات URL كمفاتيح لمنشورات مخزّنة في map.
