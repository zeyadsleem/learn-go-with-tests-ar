---
title: سطر الأوامر وهيكل المشروع
weight: 300
---

# سطر الأوامر وهيكل المشروع

**[يمكنك العثور على كل كود هذا الفصل هنا](https://github.com/quii/learn-go-with-tests/tree/main/command-line)**

يريد مالك المنتج (product owner) الآن أن _يتحوّل_ (pivot) بتقديم تطبيق ثانٍ - تطبيق سطر أوامر.

وفي الوقت الحالي، سيكفي أن يكون قادرًا على تسجيل فوز لاعب عندما يكتب المستخدم `Ruth wins`. والقصد أن يصبح في النهاية أداة تساعد المستخدمين على لعب البوكر.

ويريد مالك المنتج أن تكون قاعدة البيانات مشتركة بين التطبيقين، حتى يتحدّث الدوري وفق الانتصارات المسجّلة في التطبيق الجديد.

## تذكير بالكود

لدينا تطبيق فيه ملف `main.go` يشغّل خادم HTTP. ولن يكون خادم HTTP موضع اهتمامنا في هذا التمرين، لكن التجريد (abstraction) الذي يستخدمه سيكون كذلك. فهو يعتمد على `PlayerStore`.

```go
type PlayerStore interface {
	GetPlayerScore(name string) int
	RecordWin(name string)
	GetLeague() League
}
```

في الفصل السابق أنشأنا `FileSystemPlayerStore` الذي ينفّذ تلك الواجهة (interface). وينبغي أن نستطيع إعادة استخدام جزء من هذا في تطبيقنا الجديد.

## بعض إعادة هيكلة المشروع أولًا

يحتاج مشروعنا الآن إلى إنشاء ملفين تنفيذيين: خادم الويب الموجود لدينا، وتطبيق سطر الأوامر.

وقبل أن نغوص في عملنا الجديد، ينبغي أن ننظّم مشروعنا ليستوعب ذلك.

حتى الآن كان كل الكود يسكن مجلدًا واحدًا، في مسار يشبه هذا

`$GOPATH/src/github.com/your-name/my-app`

لكي تصنع تطبيقًا في Go فأنت تحتاج إلى دالة `main` داخل `package main`. وحتى الآن كان كل كود "المجال" (domain) لدينا يسكن داخل `package main`، ويستطيع `func main` أن يشير إلى كل شيء.

كان هذا جيدًا حتى الآن، ومن الممارسات الجيدة ألا تبالغ في هيكل الحزم. فلو أخذت وقتك وتصفحت المكتبة القياسية (standard library) فلن تجد الكثير من المجلدات والهياكل.

ولحسن الحظ، إضافة الهيكل أمر سهل جدًا _عندما تحتاجه_.

داخل المشروع الحالي أنشئ مجلدًا اسمه `cmd` وبداخله مجلد `webserver` (مثلًا `mkdir -p cmd/webserver`).

يُعد `cmd` عُرفًا شائعًا في Go لاحتواء حزم `main` الخاصة بالتطبيقات التي يبنيها المشروع، لتبقى منفصلة عن كود المكتبة القابل للاستيراد الموجود في جذر المشروع.

انقل `main.go` إلى داخله.

وإذا كان الأمر `tree` مثبتًا عندك فشغّله، وينبغي أن يبدو هيكلك هكذا

```
.
|-- file_system_store.go
|-- file_system_store_test.go
|-- cmd
|   |-- webserver
|       |-- main.go
|-- league.go
|-- server.go
|-- server_integration_test.go
|-- server_test.go
|-- tape.go
|-- tape_test.go
```

أصبح لدينا الآن فصل فعلي بين تطبيقنا وكود المكتبة، لكننا نحتاج الآن إلى تغيير بعض أسماء الحزم. وتذكّر أن حزمة تطبيق Go الذي تبنيه _يجب_ أن تكون `main`.

غيّر بقية الكود ليكون في حزمة اسمها `poker`.

وأخيرًا، نحتاج إلى استيراد هذه الحزمة في `main.go` حتى نتمكن من استخدامها في إنشاء خادم الويب. وبعدها يمكننا استخدام كود مكتبتنا عبر `poker.FunctionName`.

ستختلف المسارات على جهازك، لكنها ينبغي أن تكون مشابهة لهذا:

```go
// cmd/webserver/main.go
package main

import (
	"github.com/quii/learn-go-with-tests/command-line/v1"
	"log"
	"net/http"
	"os"
)

const dbFileName = "game.db.json"

func main() {
	db, err := os.OpenFile(dbFileName, os.O_RDWR|os.O_CREATE, 0666)

	if err != nil {
		log.Fatalf("problem opening %s %v", dbFileName, err)
	}

	store, err := poker.NewFileSystemPlayerStore(db)

	if err != nil {
		log.Fatalf("problem creating file system player store, %v ", err)
	}

	server := poker.NewPlayerServer(store)

	log.Fatal(http.ListenAndServe(":5000", server))
}
```

`dbFileName` مسار نسبي، لذا سيُنشأ `game.db.json` (أو يُقرأ) نسبةً إلى المجلد الذي تشغّل منه الملف التنفيذي الناتج، لا المجلد الذي يسكن فيه الملف التنفيذي. ولأن مالك المنتج يريد أن يتشارك تطبيق سطر الأوامر وخادم الويب قاعدة البيانات نفسها، فهذا مهم: تشغيل كليهما من مجلدين مختلفين سيمنح كل واحد منهما بهدوء ملف `game.db.json` خاصًا به منفصلًا. وسنعود إلى هذا في "الفحوصات الأخيرة" أدناه.

قد يبدو المسار الكامل مربكًا بعض الشيء، لكن هكذا يمكنك استيراد _أي_ مكتبة متاحة للعموم إلى كودك.

وبفصل كود المجال لدينا في حزمة منفصلة ورفعه إلى مستودع عام مثل GitHub، يستطيع أي مطوّر Go كتابة كوده الخاص الذي يستورد تلك الحزمة لتكون الميزات التي كتبناها متاحة له. وفي أول مرة تحاول تشغيله سيشكو من أنه غير موجود، وكل ما عليك فعله هو تشغيل `go get`.

بالإضافة إلى ذلك، يمكن للمستخدمين الاطلاع على [التوثيق على pkg.go.dev](https://pkg.go.dev/github.com/quii/learn-go-with-tests/command-line/v1).

### الفحوصات الأخيرة

- داخل الجذر شغّل `go test` وتأكد أنها ما زالت تنجح
- ادخل إلى `cmd/webserver` ونفّذ `go run main.go`
  - زُر `http://localhost:5000/league` وينبغي أن ترى أنه ما زال يعمل

لاحقًا في هذا الفصل سنبني تطبيقًا ثانيًا هو `cmd/cli`، والمقصود أن يتشارك ملف `game.db.json` نفسه مع خادم الويب. ولأن `dbFileName` يُحلّ نسبةً إلى مجلد العمل الحالي، ستحتاج إلى تشغيل الملفين التنفيذيين _من المجلد نفسه_ ليرى كل منهما تحديثات الآخر، مثل أن تبنيهما أولًا ثم تشغّل الملفين من جذر المشروع (`go build -o webserver ./cmd/webserver && go build -o cli ./cmd/cli` ثم `./webserver` و`./cli`)، بدلًا من استخدام `go run main.go` من داخل كل مجلد فرعي في `cmd`.

### الهيكل العظمي المتحرك (walking skeleton)

قبل أن نغوص في كتابة الاختبارات، لنضف تطبيقًا جديدًا سيبنياه مشروعنا. أنشئ مجلدًا آخر داخل `cmd` اسمه `cli` (واجهة سطر الأوامر) وأضف إليه ملف `main.go` بهذا المحتوى

```go
// cmd/cli/main.go
package main

import "fmt"

func main() {
	fmt.Println("Let's play poker")
}
```

أول متطلب سنتعامل معه هو تسجيل فوز عندما يكتب المستخدم `{PlayerName} wins`.

## اكتب الاختبار أولًا

نعلم أننا نحتاج إلى صنع شيء اسمه `CLI` يتيح لنا `Play` البوكر. وسيحتاج إلى قراءة مدخلات المستخدم ثم تسجيل الانتصارات في `PlayerStore`.

لكن قبل أن نستبق الأمور كثيرًا، لنكتب اختبارًا نتحقق فيه من أنه يتكامل مع `PlayerStore` بالشكل الذي نريده.

داخل `CLI_test.go` (في جذر المشروع، وليس داخل `cmd`)

```go
// CLI_test.go
package poker

import "testing"

func TestCLI(t *testing.T) {
	playerStore := &StubPlayerStore{}
	cli := &CLI{playerStore}
	cli.PlayPoker()

	if len(playerStore.winCalls) != 1 {
		t.Fatal("expected a win call but didn't get any")
	}
}
```

- يمكننا استخدام `StubPlayerStore` من اختبارات أخرى
- نمرّر الاعتمادية (dependency) إلى نوع `CLI` الذي لم يوجد بعد
- نُشغّل اللعبة عبر method غير مكتوبة هي `PlayPoker`
- نتحقق من تسجيل فوز

## جرّب تشغيل الاختبار

```
# github.com/quii/learn-go-with-tests/command-line/v2
./cli_test.go:25:10: undefined: CLI
```

## اكتب أصغر قدر من الكود ليعمل الاختبار وافحص مخرجاته الفاشلة

في هذه المرحلة، ينبغي أن تكون مرتاحًا بما يكفي لإنشاء struct الـ `CLI` الجديدة مع الحقل المناسب لاعتماديتنا وإضافة method.

وينبغي أن ينتهي بك الأمر إلى كود كهذا

```go
// CLI.go
package poker

type CLI struct {
	playerStore PlayerStore
}

func (cli *CLI) PlayPoker() {}
```

تذكّر أننا نحاول فقط تشغيل الاختبار لنفحص أنه يفشل بالشكل الذي نتمناه

```
--- FAIL: TestCLI (0.00s)
    cli_test.go:30: expected a win call but didn't get any
FAIL
```

## اكتب كودًا كافيًا لنجاح الاختبار

```go
//CLI.go
func (cli *CLI) PlayPoker() {
	cli.playerStore.RecordWin("Cleo")
}
```

ينبغي أن يجعل ذلك الاختبار ينجح.

بعد ذلك، نحتاج إلى محاكاة القراءة من `Stdin` (مدخلات المستخدم) حتى نتمكن من تسجيل انتصارات لاعبين محددين.

لنوسّع اختبارنا ليمرّن هذا.

## اكتب الاختبار أولًا

```go
//CLI_test.go
func TestCLI(t *testing.T) {
	in := strings.NewReader("Chris wins\n")
	playerStore := &StubPlayerStore{}

	cli := &CLI{playerStore, in}
	cli.PlayPoker()

	if len(playerStore.winCalls) != 1 {
		t.Fatal("expected a win call but didn't get any")
	}

	got := playerStore.winCalls[0]
	want := "Chris"

	if got != want {
		t.Errorf("didn't record correct winner, got %q, want %q", got, want)
	}
}
```

‏`os.Stdin` هو ما سنستخدمه في `main` لالتقاط مدخلات المستخدم. وهو `*File` من تحت الغطاء، ما يعني أنه ينفّذ `io.Reader` التي صرنا نعرف أنها طريقة ملائمة لالتقاط النص.

وننشئ `io.Reader` في اختبارنا باستخدام `strings.NewReader` الملائمة، ونملؤها بما نتوقع أن يكتبه المستخدم.

## جرّب تشغيل الاختبار

`./CLI_test.go:12:32: too many values in struct initializer`

## اكتب أصغر قدر من الكود ليعمل الاختبار وافحص مخرجاته الفاشلة

نحتاج إلى إضافة اعتماديتنا الجديدة إلى `CLI`.

```go
//CLI.go
type CLI struct {
	playerStore PlayerStore
	in          io.Reader
}
```

```
--- FAIL: TestCLI (0.00s)
    CLI_test.go:23: didn't record the correct winner, got 'Cleo', want 'Chris'
FAIL
```

## اكتب كودًا كافيًا لنجاح الاختبار

تذكّر أن تفعل أسهل شيء ممكن أولًا

```go
func (cli *CLI) PlayPoker() {
	cli.playerStore.RecordWin("Chris")
}
```

ينجح الاختبار. وسنضيف اختبارًا آخر يجبرنا على كتابة كود حقيقي، لكن أولًا لنُعِد الهيكلة.

## إعادة الهيكلة

في `server_test` قمنا سابقًا بفحوصات للتأكد من تسجيل الانتصارات كما نفعل هنا. لنطبّق DRY على ذلك التحقق (assertion) وننقله إلى دالة مساعدة

```go
//server_test.go
func assertPlayerWin(t testing.TB, store *StubPlayerStore, winner string) {
	t.Helper()

	if len(store.winCalls) != 1 {
		t.Fatalf("got %d calls to RecordWin want %d", len(store.winCalls), 1)
	}

	if store.winCalls[0] != winner {
		t.Errorf("did not store correct winner got %q want %q", store.winCalls[0], winner)
	}
}
```

استبدل الآن التحققات في كل من `server_test.go` و`CLI_test.go`.

وينبغي أن يصبح شكل الاختبار هكذا

```go
//CLI_test.go
func TestCLI(t *testing.T) {
	in := strings.NewReader("Chris wins\n")
	playerStore := &StubPlayerStore{}

	cli := &CLI{playerStore, in}
	cli.PlayPoker()

	assertPlayerWin(t, playerStore, "Chris")
}
```

والآن لنكتب اختبارًا _آخر_ بمدخلات مستخدم مختلفة، ليدفعنا إلى قراءتها فعلًا.

## اكتب الاختبار أولًا

```go
//CLI_test.go
func TestCLI(t *testing.T) {

	t.Run("record chris win from user input", func(t *testing.T) {
		in := strings.NewReader("Chris wins\n")
		playerStore := &StubPlayerStore{}

		cli := &CLI{playerStore, in}
		cli.PlayPoker()

		assertPlayerWin(t, playerStore, "Chris")
	})

	t.Run("record cleo win from user input", func(t *testing.T) {
		in := strings.NewReader("Cleo wins\n")
		playerStore := &StubPlayerStore{}

		cli := &CLI{playerStore, in}
		cli.PlayPoker()

		assertPlayerWin(t, playerStore, "Cleo")
	})

}
```

## جرّب تشغيل الاختبار

```
=== RUN   TestCLI
--- FAIL: TestCLI (0.00s)
=== RUN   TestCLI/record_chris_win_from_user_input
    --- PASS: TestCLI/record_chris_win_from_user_input (0.00s)
=== RUN   TestCLI/record_cleo_win_from_user_input
    --- FAIL: TestCLI/record_cleo_win_from_user_input (0.00s)
        CLI_test.go:27: did not store correct winner got 'Chris' want 'Cleo'
FAIL
```

## اكتب كودًا كافيًا لنجاح الاختبار

سنستخدم [`bufio.Scanner`](https://golang.org/pkg/bufio/) لقراءة المدخلات من `io.Reader`.

> تنفّذ حزمة bufio الإدخال والإخراج المُخزَّن (buffered I/O). وهي تُغلّف كائن io.Reader أو io.Writer، فتنشئ كائنًا آخر (Reader أو Writer) ينفّذ الواجهة نفسها لكنه يوفّر التخزين المؤقت وبعض المساعدة في الإدخال والإخراج النصي.

حدّث الكود إلى ما يلي

```go
//CLI.go
type CLI struct {
	playerStore PlayerStore
	in          io.Reader
}

func (cli *CLI) PlayPoker() {
	reader := bufio.NewScanner(cli.in)
	reader.Scan()
	cli.playerStore.RecordWin(extractWinner(reader.Text()))
}

func extractWinner(userInput string) string {
	return strings.Replace(userInput, " wins", "", 1)
}
```

ستنجح الاختبارات الآن.

- ستقرأ `Scanner.Scan()` حتى نهاية سطر جديد.
- ثم نستخدم `Scanner.Text()` لإرجاع الـ `string` التي قرأها الماسح.

وبعد أن أصبحت لدينا اختبارات ناجحة، ينبغي أن نوصل هذا بـ `main`. وتذكّر أننا ينبغي أن نسعى دائمًا إلى الحصول على برمجية عاملة ومتكاملة تمامًا بأسرع ما يمكن.

في `main.go` أضف ما يلي وشغّله. (قد تحتاج إلى تعديل مسار الاعتمادية الثانية ليطابق ما هو موجود على جهازك)

```go
package main

import (
	"fmt"
	"github.com/quii/learn-go-with-tests/command-line/v3"
	"log"
	"os"
)

const dbFileName = "game.db.json"

func main() {
	fmt.Println("Let's play poker")
	fmt.Println("Type {Name} wins to record a win")

	db, err := os.OpenFile(dbFileName, os.O_RDWR|os.O_CREATE, 0666)

	if err != nil {
		log.Fatalf("problem opening %s %v", dbFileName, err)
	}

	store, err := poker.NewFileSystemPlayerStore(db)

	if err != nil {
		log.Fatalf("problem creating file system player store, %v ", err)
	}

	game := poker.CLI{store, os.Stdin}
	game.PlayPoker()
}
```

ينبغي أن تحصل على خطأ

```
command-line/v3/cmd/cli/main.go:32:25: implicit assignment of unexported field 'playerStore' in poker.CLI literal
command-line/v3/cmd/cli/main.go:32:34: implicit assignment of unexported field 'in' in poker.CLI literal
```

ما يحدث هنا سببه أننا نحاول الإسناد إلى الحقلين `playerStore` و`in` في `CLI`. وهذان حقلان غير مُصدَّرين (خاصان). وكان _بإمكاننا_ فعل ذلك في كود اختبارنا لأن اختبارنا في الحزمة نفسها التي فيها `CLI` (`poker`). لكن `main` لدينا في الحزمة `main`، لذا لا يملك صلاحية الوصول.

ويُبرز هذا أهمية _دمج عملك_. فقد جعلنا اعتماديات `CLI` خاصة عن حق (لأننا لا نريد كشفها لمستخدمي `CLI`)، لكننا لم نوفّر طريقة يبنيها بها المستخدمون.

هل توجد طريقة لكشف هذه المشكلة مبكرًا؟

### `package mypackage_test`

في كل الأمثلة الأخرى حتى الآن، عندما ننشئ ملف اختبار نعلنه في الحزمة نفسها التي نختبرها.

وهذا جيد، ويعني أنه في المناسبات النادرة التي نريد فيها اختبار شيء داخلي في الحزمة نستطيع الوصول إلى الأنواع غير المُصدَّرة.

لكن بما أننا نادينا بـ_عدم_ اختبار الأمور الداخلية _عمومًا_، فهل يمكن أن تساعدنا Go على فرض ذلك؟ ماذا لو استطعنا اختبار كودنا حيث لا نملك صلاحية الوصول إلا إلى الأنواع المُصدَّرة (كما يفعل `main` لدينا)؟

عندما تكتب مشروعًا بحزم متعددة، أوصي بشدة أن ينتهي اسم حزمة اختبارك بـ `_test`. وعندما تفعل ذلك لن تستطيع الوصول إلا إلى الأنواع العامة في حزمتك. وسيساعد هذا في هذه الحالة تحديدًا، كما يساعد على فرض الانضباط باختبار واجهات API العامة فقط. وإن كنت ما زلت ترغب في اختبار الأمور الداخلية، فيمكنك إنشاء اختبار منفصل بالحزمة التي تريد اختبارها.

ومن مقولات TDD أنك إن لم تستطع اختبار كودك، فمن المحتمل أن يكون دمج الآخرين معه صعبًا. واستخدام `package foo_test` سيساعد في ذلك بأن يجبرك على اختبار كودك كأنك تستورده كما سيفعل مستخدمو حزمتك.

وقبل إصلاح `main` لنغيّر حزمة اختبارنا داخل `CLI_test.go` إلى `poker_test`.

وإذا كان لديك IDE مضبوط جيدًا فسترى فجأة الكثير من الأحمر! وإذا شغّلت المترجم فستحصل على الأخطاء التالية

```
./CLI_test.go:12:19: undefined: StubPlayerStore
./CLI_test.go:17:3: undefined: assertPlayerWin
./CLI_test.go:22:19: undefined: StubPlayerStore
./CLI_test.go:27:3: undefined: assertPlayerWin
```

لقد تعثّرنا الآن بأسئلة إضافية حول تصميم الحزم. فلكي نختبر برمجيتنا أنشأنا stubs ودوال مساعدة غير مُصدَّرة، ولم تعد متاحة لنا في `CLI_test` لأن الدوال المساعدة معرَّفة في ملفات `_test.go` في حزمة `poker`.

#### هل نريد أن تكون الـ stubs والدوال المساعدة 'عامة'؟

هذا نقاش ذو طابع شخصي. فقد يقول قائل إنك لا تريد تلويث واجهة API حزمتك بكود يسهّل الاختبارات.

وفي العرض التقديمي ["Advanced Testing with Go"](https://speakerdeck.com/mitchellh/advanced-testing-with-go?slide=53) لميتشل هاشيموتو، يُوصف كيف يدعو فريق HashiCorp إلى فعل هذا حتى يتمكن مستخدمو الحزمة من كتابة اختبارات دون إعادة اختراع العجلة بكتابة stubs. وهذا يعني في حالتنا أن أي شخص يستخدم حزمة `poker` لن يحتاج إلى إنشاء `PlayerStore` الـ stub الخاص به إن أراد العمل مع كودنا.

ومن تجربتي الشخصية استخدمت هذه التقنية في حزم مشتركة أخرى، وأثبتت فائدة كبيرة من حيث توفير وقت المستخدمين عند التكامل مع حزمنا.

فلننشئ ملفًا اسمه `testing.go` ونضف إليه الـ stub والدوال المساعدة.

```go
// testing.go
package poker

import "testing"

type StubPlayerStore struct {
	scores   map[string]int
	winCalls []string
	league   []Player
}

func (s *StubPlayerStore) GetPlayerScore(name string) int {
	score := s.scores[name]
	return score
}

func (s *StubPlayerStore) RecordWin(name string) {
	s.winCalls = append(s.winCalls, name)
}

func (s *StubPlayerStore) GetLeague() League {
	return s.league
}

func AssertPlayerWin(t testing.TB, store *StubPlayerStore, winner string) {
	t.Helper()

	if len(store.winCalls) != 1 {
		t.Fatalf("got %d calls to RecordWin want %d", len(store.winCalls), 1)
	}

	if store.winCalls[0] != winner {
		t.Errorf("did not store correct winner got %q want %q", store.winCalls[0], winner)
	}
}

// todo for you - the rest of the helpers
```

ستحتاج إلى جعل الدوال المساعدة عامة (تذكّر أن التصدير يحدث بحرف كبير في البداية) إن أردت كشفها لمن يستوردون حزمتنا.

وفي اختبار `CLI` لدينا ستحتاج إلى استدعاء الكود كما لو كنت تستخدمه من حزمة مختلفة.

```go
//CLI_test.go
func TestCLI(t *testing.T) {

	t.Run("record chris win from user input", func(t *testing.T) {
		in := strings.NewReader("Chris wins\n")
		playerStore := &poker.StubPlayerStore{}

		cli := &poker.CLI{playerStore, in}
		cli.PlayPoker()

		poker.AssertPlayerWin(t, playerStore, "Chris")
	})

	t.Run("record cleo win from user input", func(t *testing.T) {
		in := strings.NewReader("Cleo wins\n")
		playerStore := &poker.StubPlayerStore{}

		cli := &poker.CLI{playerStore, in}
		cli.PlayPoker()

		poker.AssertPlayerWin(t, playerStore, "Cleo")
	})

}
```

سترى الآن أن لدينا المشكلات نفسها التي واجهناها في `main`

```
./CLI_test.go:15:26: implicit assignment of unexported field 'playerStore' in poker.CLI literal
./CLI_test.go:15:39: implicit assignment of unexported field 'in' in poker.CLI literal
./CLI_test.go:25:26: implicit assignment of unexported field 'playerStore' in poker.CLI literal
./CLI_test.go:25:39: implicit assignment of unexported field 'in' in poker.CLI literal
```

وأسهل طريقة للتغلب على هذا هي إنشاء دالة بانية (constructor) كما فعلنا مع أنواع أخرى. وسنغيّر `CLI` أيضًا ليخزّن `bufio.Scanner` بدلًا من الـ reader، لأنها الآن تُغلَّف تلقائيًا وقت البناء.

```go
//CLI.go
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
```

وبفعل هذا يمكننا تبسيط كود القراءة وإعادة هيكلته

```go
//CLI.go
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

غيّر الاختبار ليستخدم الدالة البانية بدلًا من ذلك، وينبغي أن نعود إلى اختبارات ناجحة.

وأخيرًا، يمكننا العودة إلى `main.go` الجديد واستخدام الدالة البانية التي أنشأناها للتو

```go
//cmd/cli/main.go
game := poker.NewCLI(store, os.Stdin)
```

جرّب تشغيله واكتب "Bob wins".

### إعادة الهيكلة

لدينا بعض التكرار في تطبيقينا حيث نفتح ملفًا وننشئ `file_system_store` من محتواه. ويبدو هذا ضعفًا طفيفًا في تصميم حزمتنا، لذا ينبغي أن نصنع فيها دالة تغلّف فتح ملف من مسار وتعيد لك `PlayerStore`.

```go
//file_system_store.go
func FileSystemPlayerStoreFromFile(path string) (*FileSystemPlayerStore, func(), error) {
	db, err := os.OpenFile(path, os.O_RDWR|os.O_CREATE, 0666)

	if err != nil {
		return nil, nil, fmt.Errorf("problem opening %s %v", path, err)
	}

	closeFunc := func() {
		db.Close()
	}

	store, err := NewFileSystemPlayerStore(db)

	if err != nil {
		return nil, nil, fmt.Errorf("problem creating file system player store, %v ", err)
	}

	return store, closeFunc, nil
}
```

الآن أعد هيكلة تطبيقينا كليهما ليستخدما هذه الدالة في إنشاء المخزن.

#### كود تطبيق CLI

```go
// cmd/cli/main.go
package main

import (
	"fmt"
	"github.com/quii/learn-go-with-tests/command-line/v3"
	"log"
	"os"
)

const dbFileName = "game.db.json"

func main() {
	store, close, err := poker.FileSystemPlayerStoreFromFile(dbFileName)

	if err != nil {
		log.Fatal(err)
	}
	defer close()

	fmt.Println("Let's play poker")
	fmt.Println("Type {Name} wins to record a win")
	poker.NewCLI(store, os.Stdin).PlayPoker()
}
```

#### كود تطبيق خادم الويب

```go
// cmd/webserver/main.go
package main

import (
	"github.com/quii/learn-go-with-tests/command-line/v3"
	"log"
	"net/http"
)

const dbFileName = "game.db.json"

func main() {
	store, close, err := poker.FileSystemPlayerStoreFromFile(dbFileName)

	if err != nil {
		log.Fatal(err)
	}
	defer close()

	server := poker.NewPlayerServer(store)

	if err := http.ListenAndServe(":5000", server); err != nil {
		log.Fatalf("could not listen on port 5000 %v", err)
	}
}
```

لاحظ التماثل: فرغم أن واجهتي المستخدم مختلفتان، فإن الإعداد يكاد يكون متطابقًا. ويبدو هذا تأكيدًا جيدًا لتصميمنا حتى الآن.
ولاحظ أيضًا أن `FileSystemPlayerStoreFromFile` تُرجع دالة إغلاق، حتى نتمكن من إغلاق الملف الأساسي بمجرد انتهائنا من استخدام المخزن (Store).

## الخلاصة

### هيكل الحزم

كان هدف هذا الفصل إنشاء تطبيقين يعيدان استخدام كود المجال الذي كتبناه حتى الآن. ولتحقيق ذلك احتجنا إلى تحديث هيكل حزمنا ليكون لدينا مجلدات منفصلة لكل `main` من تطبيقاتنا.

وبفعل ذلك اصطدمنا بمشكلات تكامل بسبب قيم غير مُصدَّرة، ما يوضّح أكثر قيمة العمل على شكل "شرائح" صغيرة والدمج المتكرر.

وتعلّمنا كيف تساعدنا `mypackage_test` على إنشاء بيئة اختبار تمنح التجربة نفسها التي تحصل عليها الحزم الأخرى عند التكامل مع كودك، لتساعدك على كشف مشكلات التكامل ورؤية مدى سهولة (أو صعوبة!) التعامل مع كودك.

### قراءة مدخلات المستخدم

رأينا كيف أن القراءة من `os.Stdin` سهلة جدًا للتعامل معها لأنها تنفّذ `io.Reader`. واستخدمنا `bufio.Scanner` لقراءة مدخلات المستخدم سطرًا سطرًا بسهولة.

### التجريدات البسيطة تؤدي إلى إعادة استخدام أبسط للكود

لم يكد يتطلب أي جهد دمج `PlayerStore` في تطبيقنا الجديد (بعد أن أجرينا تعديلات الحزم)، وكان الاختبار بعدها سهلًا جدًا أيضًا لأننا قررنا كشف نسختنا من الـ stub.
