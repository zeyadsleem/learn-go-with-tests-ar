---
title: خادم HTTP
weight: 270
---

# خادم HTTP

**[يمكنك العثور على كل كود هذا الفصل هنا](https://github.com/quii/learn-go-with-tests/tree/main/http-server)**

طُلب منك إنشاء خادم ويب يستطيع المستخدمون من خلاله تتبع عدد المباريات التي فاز بها اللاعبون.

-   `GET /players/{name}` ينبغي أن تُرجع رقمًا يدل على إجمالي عدد مرات الفوز
-   `POST /players/{name}` ينبغي أن تسجّل فوزًا لذلك الاسم، ويزيد العدد مع كل `POST` تالٍ

سنتبع نهج التطوير الموجه بالاختبار، فنصل إلى برنامج يعمل بأسرع ما نستطيع، ثم نجري تحسينات صغيرة متكررة حتى نصل إلى الحل. وباتباعنا هذا النهج:

-   نُبقي مساحة المشكلة صغيرة في أي لحظة
-   لا نسقط في جحور الأرانب
-   إن تعثرنا أو تهنا يومًا، فلن يؤدي الرجوع عن التغييرات (revert) إلى خسارة قدر كبير من العمل.

## الأحمر والأخضر وإعادة الهيكلة

شدّدنا في هذا الكتاب كله على عملية التطوير الموجه بالاختبار: اكتب اختبارًا وشاهده يفشل (أحمر)، ثم اكتب _أصغر_ قدر من الكود ليعمل (أخضر)، ثم أعد الهيكلة.

وهذا الانضباط في كتابة أصغر قدر من الكود مهم من ناحية الأمان الذي يمنحك إياه التطوير الموجه بالاختبار. وينبغي أن تسعى إلى الخروج من حالة "الأحمر" بأسرع ما يمكن.

يصفها كِنت بِك هكذا:

> اجعل الاختبار ينجح بسرعة، واقترف في الطريق ما يلزم من الذنوب.

يمكنك اقتراف هذه الذنوب لأنك ستُعيد الهيكلة بعدها مدعومًا بأمان الاختبارات.

### ماذا لو لم تفعل ذلك؟

كلما زادت التغييرات التي تُجريها وأنت في حالة "الأحمر"، زاد احتمال إضافتك مشكلات أكثر لا تغطيها الاختبارات.

الفكرة أن تكتب كودًا مفيدًا بخطوات صغيرة متكررة، مدفوعًا بالاختبارات، حتى لا تسقط في جحر أرنب لساعات.

### الدجاجة والبيضة

كيف نبني هذا بشكل تكراري؟ لا يمكننا تنفيذ `GET` للاعب دون أن نكون قد خزّنّا شيئًا، ويبدو من الصعب معرفة إن كان `POST` قد نجح دون وجود نقطة النهاية (endpoint) الخاصة بـ `GET` أصلًا.

هنا يتألق الـ _mocking_.

-   سيحتاج `GET` إلى _شيء_ اسمه `PlayerStore` لجلب نقاط لاعب. وينبغي أن يكون هذا واجهة (interface) حتى نتمكن عند الاختبار من إنشاء stub بسيط لاختبار كودنا دون الحاجة إلى تنفيذ أي كود تخزين فعلي.
-   أما `POST` فيمكننا _التجسس (spy)_ على استدعاءاته لـ `PlayerStore` للتأكد أنه يخزّن اللاعبين بشكل صحيح. ولن يرتبط تنفيذنا للحفظ بعملية الجلب.
-   وللحصول على برنامج يعمل بسرعة، يمكننا كتابة تنفيذ بسيط جدًا في الذاكرة، ثم لاحقًا ننشئ تنفيذًا مدعومًا بأي آلية تخزين نفضّلها.

## اكتب الاختبار أولًا

يمكننا كتابة اختبار ونجعله ينجح بإرجاع قيمة مكتوبة مباشرة (hard-coded) لننطلق. ويسمّي كِنت بِك هذا "التزوير" (Faking it). وبعد أن يصبح لدينا اختبار يعمل، نكتب اختبارات إضافية تساعدنا على إزالة ذلك الثابت.

وبهذه الخطوة الصغيرة جدًا، نبدأ بداية مهمة بجعل بنية المشروع ككل تعمل بشكل صحيح دون الحاجة إلى القلق كثيرًا بشأن منطق تطبيقنا.

لإنشاء خادم ويب في Go ستستدعي عادةً [ListenAndServe](https://golang.org/pkg/net/http/#ListenAndServe).

```go
func ListenAndServe(addr string, handler Handler) error
```

سيشغّل هذا خادم ويب يستمع على منفذ، وينشئ goroutine لكل طلب ويشغّله مقابل [`Handler`](https://golang.org/pkg/net/http/#Handler).

```go
type Handler interface {
	ServeHTTP(ResponseWriter, *Request)
}
```

يحقق نوعٌ ما واجهة Handler بتنفيذه الـ method `ServeHTTP` التي تتوقع وسيطين: الأول مكان _كتابة استجابتنا_، والثاني هو طلب HTTP الذي أُرسل إلى الخادم.

لننشئ ملفًا اسمه `server_test.go` ونكتب اختبارًا لدالة `PlayerServer` تأخذ هذين الوسيطين. والطلب المُرسل سيكون لجلب نقاط لاعب، ونتوقع أن تكون `"20"`.

```go
func TestGETPlayers(t *testing.T) {
	t.Run("returns Pepper's score", func(t *testing.T) {
		request, _ := http.NewRequest(http.MethodGet, "/players/Pepper", nil)
		response := httptest.NewRecorder()

		PlayerServer(response, request)

		got := response.Body.String()
		want := "20"

		if got != want {
			t.Errorf("got %q, want %q", got, want)
		}
	})
}
```

للاختبار خادمنا، سنحتاج إلى `Request` نُرسله، وسنريد _التجسس_ على ما يكتبه المعالج في `ResponseWriter`.

-   نستخدم `http.NewRequest` لإنشاء طلب. الوسيط الأول هو method الطلب، والثاني هو مسار الطلب. والوسيط `nil` يشير إلى جسم الطلب (body)، وهو ما لا نحتاج إلى ضبطه في هذه الحالة.
-   تحتوي `net/http/httptest` على متجسس جاهز لنا اسمه `ResponseRecorder`، فيمكننا استخدامه. وفيه methods مفيدة كثيرة لفحص ما كُتب في الاستجابة.

## جرّب تشغيل الاختبار

`./server_test.go:13:2: undefined: PlayerServer`

## اكتب أصغر قدر من الكود ليعمل الاختبار وافحص مخرجاته الفاشلة

المترجم هنا لمساعدتك، فاستمع إليه فقط.

أنشئ ملفًا اسمه `server.go` وعرّف فيه `PlayerServer`.

```go
func PlayerServer() {}
```

جرّب مرة أخرى

```
./server_test.go:13:14: too many arguments in call to PlayerServer
    have (*httptest.ResponseRecorder, *http.Request)
    want ()
```

أضف الوسيطين إلى دالتنا

```go
import "net/http"

func PlayerServer(w http.ResponseWriter, r *http.Request) {

}
```

الكود الآن يُترجم والاختبار يفشل

```
=== RUN   TestGETPlayers/returns_Pepper's_score
    --- FAIL: TestGETPlayers/returns_Pepper's_score (0.00s)
        server_test.go:20: got '', want '20'
```

## اكتب كودًا كافيًا لنجاح الاختبار

في فصل حقن الاعتماديات (DI) تناولنا خوادم HTTP بدالة `Greet`. وتعلّمنا أن `ResponseWriter` في net/http يحقق أيضًا `io.Writer`، فيمكننا استخدام `fmt.Fprint` لإرسال نصوص كاستجابات HTTP.

```go
func PlayerServer(w http.ResponseWriter, r *http.Request) {
	fmt.Fprint(w, "20")
}
```

من المفترض أن ينجح الاختبار الآن.

## أكمل الهيكل الأولي

نريد توصيل هذا كله في تطبيق. وهذا مهم لأننا

-   سنحصل على _برنامج يعمل فعلًا_، ولا نريد كتابة اختبارات لمجرد كتابتها، فمن الجيد رؤية الكود أثناء العمل.
-   وبينما نُعيد هيكلة كودنا، يُرجّح أن تتغير بنية البرنامج. ونريد التأكد أن هذا ينعكس على تطبيقنا أيضًا ضمن النهج التكراري.

أنشئ ملف `main.go` جديدًا لتطبيقنا وضع فيه هذا الكود

```go
package main

import (
	"log"
	"net/http"
)

func main() {
	handler := http.HandlerFunc(PlayerServer)
	log.Fatal(http.ListenAndServe(":5000", handler))
}
```

حتى الآن كان كل كود تطبيقنا في ملف واحد، لكن هذه ليست أفضل ممارسة في المشروعات الأكبر التي ستريد فيها فصل الأشياء في ملفات مختلفة.

لتشغيل هذا نفّذ `go build`، وسيأخذ كل ملفات `.go` في المجلد ويبني لك برنامجًا. ثم يمكنك تنفيذه بـ `./myprogram`.

### `http.HandlerFunc`

استكشفنا سابقًا أن واجهة `Handler` هي ما نحتاج إلى تحقيقه لصنع خادم. و_عادةً_ نفعل ذلك بإنشاء `struct` وجعله يحقق الواجهة بتنفيذ method خاصة به اسمها ServeHTTP. لكن استخدامات الـ structs هي الاحتفاظ بالبيانات، و_حاليًا_ ليس لدينا أي حالة (state)، فلا يبدو من المناسب إنشاء واحد.

تتيح لنا [HandlerFunc](https://golang.org/pkg/net/http/#HandlerFunc) تجنب ذلك.

> النوع HandlerFunc محوّل (adapter) يسمح باستخدام دوال عادية كمعالجات HTTP. فإذا كانت f دالة بالتوقيع المناسب، فإن HandlerFunc(f) معالج (Handler) يستدعي f.

```go
type HandlerFunc func(ResponseWriter, *Request)
```

من التوثيق نرى أن النوع `HandlerFunc` قد حقق فعلًا method الـ `ServeHTTP`.
وبتحويلنا دالة `PlayerServer` إليه (type casting)، نكون قد حققنا `Handler` المطلوب.

### `http.ListenAndServe(":5000"...)`

تأخذ `ListenAndServe` منفذًا تستمع عليه و`Handler`. وإذا حدثت مشكلة فسيرجع خادم الويب خطأً، ومن أمثلة ذلك أن يكون المنفذ قيد الاستماع بالفعل. ولهذا السبب نغلّف الاستدعاء بـ `log.Fatal` لتسجيل الخطأ للمستخدم.

ما سنفعله الآن هو كتابة اختبار _آخر_ يدفعنا إلى إجراء تغيير إيجابي لمحاولة الابتعاد عن القيمة المكتوبة مباشرة.

## اكتب الاختبار أولًا

سنضيف اختبارًا فرعيًا آخر إلى مجموعتنا يحاول جلب نقاط لاعب مختلف، وهذا سيكسر نهجنا المكتوب مباشرة.

```go
t.Run("returns Floyd's score", func(t *testing.T) {
	request, _ := http.NewRequest(http.MethodGet, "/players/Floyd", nil)
	response := httptest.NewRecorder()

	PlayerServer(response, request)

	got := response.Body.String()
	want := "10"

	if got != want {
		t.Errorf("got %q, want %q", got, want)
	}
})
```

ربما كنت تفكر

> من المؤكد أننا نحتاج إلى نوع من مفهوم التخزين للتحكم في النقاط التي يحصل عليها كل لاعب. فمن الغريب أن تبدو القيم بهذا القدر من العشوائية في اختباراتنا.

تذكّر أننا نحاول فقط أخذ أصغر الخطوات المعقولة، لذا نحاول الآن كسر الثابت فقط.

## جرّب تشغيل الاختبار

```
=== RUN   TestGETPlayers/returns_Pepper's_score
    --- PASS: TestGETPlayers/returns_Pepper's_score (0.00s)
=== RUN   TestGETPlayers/returns_Floyd's_score
    --- FAIL: TestGETPlayers/returns_Floyd's_score (0.00s)
        server_test.go:34: got '20', want '10'
```

## اكتب كودًا كافيًا لنجاح الاختبار

```go
//server.go
func PlayerServer(w http.ResponseWriter, r *http.Request) {
	player := strings.TrimPrefix(r.URL.Path, "/players/")

	if player == "Pepper" {
		fmt.Fprint(w, "20")
		return
	}

	if player == "Floyd" {
		fmt.Fprint(w, "10")
		return
	}
}
```

أجبرنا هذا الاختبار على النظر فعليًا في عنوان URL للطلب واتخاذ قرار. فبينما ربما كنا في أذهاننا قلقين بشأن مخازن اللاعبين والواجهات، تبدو الخطوة المنطقية التالية متعلقة فعليًا بـ _التوجيه (routing)_.

لو بدأنا بكود المخزن لكان مقدار التغييرات التي سيتعين علينا إجراءها كبيرًا جدًا مقارنةً بهذا. **فهذه خطوة أصغر نحو هدفنا النهائي وقد دفعتنا إليها الاختبارات**.

نقاوم إغراء استخدام أي مكتبات توجيه الآن، ونكتفي بأصغر خطوة لنجاح اختبارنا.

تُرجع `r.URL.Path` مسار الطلب، ويمكننا بعدها استخدام [`strings.TrimPrefix`](https://golang.org/pkg/strings/#TrimPrefix) لاقتطاع `/players/` والحصول على اللاعب المطلوب. هذا ليس متينًا كثيرًا لكنه سيؤدي الغرض الآن.

## إعادة الهيكلة

يمكننا تبسيط `PlayerServer` بفصل جلب النقاط في دالة منفصلة

```go
//server.go
func PlayerServer(w http.ResponseWriter, r *http.Request) {
	player := strings.TrimPrefix(r.URL.Path, "/players/")

	fmt.Fprint(w, GetPlayerScore(player))
}

func GetPlayerScore(name string) string {
	if name == "Pepper" {
		return "20"
	}

	if name == "Floyd" {
		return "10"
	}

	return ""
}
```

ويمكننا إزالة بعض التكرار (DRY) في الاختبارات بعمل بعض الدوال المساعدة

```go
//server_test.go
func TestGETPlayers(t *testing.T) {
	t.Run("returns Pepper's score", func(t *testing.T) {
		request := newGetScoreRequest("Pepper")
		response := httptest.NewRecorder()

		PlayerServer(response, request)

		assertResponseBody(t, response.Body.String(), "20")
	})

	t.Run("returns Floyd's score", func(t *testing.T) {
		request := newGetScoreRequest("Floyd")
		response := httptest.NewRecorder()

		PlayerServer(response, request)

		assertResponseBody(t, response.Body.String(), "10")
	})
}

func newGetScoreRequest(name string) *http.Request {
	req, _ := http.NewRequest(http.MethodGet, fmt.Sprintf("/players/%s", name), nil)
	return req
}

func assertResponseBody(t testing.TB, got, want string) {
	t.Helper()
	if got != want {
		t.Errorf("response body is wrong, got %q want %q", got, want)
	}
}
```

لكن يجب ألا نرضى بعد. لا يبدو صحيحًا أن خادمنا يعرف النقاط.

لقد أوضحت لنا إعادة الهيكلة ما ينبغي فعله.

أخرجنا حساب النقاط من جسم المعالج الرئيسي إلى دالة `GetPlayerScore`. ويبدو هذا هو المكان المناسب لفصل الاهتمامات باستخدام الواجهات.

لنحوّل دالتنا التي أعدنا هيكلتها إلى واجهة بدلًا منها

```go
type PlayerStore interface {
	GetPlayerScore(name string) int
}
```

حتى يستطيع `PlayerServer` استخدام `PlayerStore`، سيحتاج إلى مرجع (reference) إليه. ويبدو الآن الوقت المناسب لتغيير معماريتنا بحيث يصبح `PlayerServer` عبارة عن `struct`.

```go
type PlayerServer struct {
	store PlayerStore
}
```

وأخيرًا سنحقّق واجهة `Handler` بإضافة method إلى struct الجديد ووضع كود المعالج الحالي فيه.

```go
func (p *PlayerServer) ServeHTTP(w http.ResponseWriter, r *http.Request) {
	player := strings.TrimPrefix(r.URL.Path, "/players/")
	fmt.Fprint(w, p.store.GetPlayerScore(player))
}
```

التغيير الآخر الوحيد أننا الآن نستدعي `store.GetPlayerScore` لجلب النقاط، بدلًا من الدالة المحلية التي عرّفناها (ويمكننا الآن حذفها).

إليك القائمة الكاملة لكود خادمنا

```go
//server.go
type PlayerStore interface {
	GetPlayerScore(name string) int
}

type PlayerServer struct {
	store PlayerStore
}

func (p *PlayerServer) ServeHTTP(w http.ResponseWriter, r *http.Request) {
	player := strings.TrimPrefix(r.URL.Path, "/players/")
	fmt.Fprint(w, p.store.GetPlayerScore(player))
}
```

### إصلاح المشكلات

كانت هذه تغييرات كثيرة، ونعلم أن اختباراتنا وتطبيقنا لن يُترجما بعد الآن، لكن استرخِ فقط ودع المترجم يعمل حتى النهاية.

`./main.go:9:58: type PlayerServer is not an expression`

نحتاج إلى تغيير اختباراتنا لتنشئ بدلًا من ذلك نسخة جديدة من `PlayerServer` ثم تستدعي method الـ `ServeHTTP`.

```go
//server_test.go
func TestGETPlayers(t *testing.T) {
	server := &PlayerServer{}

	t.Run("returns Pepper's score", func(t *testing.T) {
		request := newGetScoreRequest("Pepper")
		response := httptest.NewRecorder()

		server.ServeHTTP(response, request)

		assertResponseBody(t, response.Body.String(), "20")
	})

	t.Run("returns Floyd's score", func(t *testing.T) {
		request := newGetScoreRequest("Floyd")
		response := httptest.NewRecorder()

		server.ServeHTTP(response, request)

		assertResponseBody(t, response.Body.String(), "10")
	})
}
```

لاحظ أننا ما زلنا لا نقلق بشأن صنع المخازن _بعد_، فنحن نريد فقط أن يُترجم الكود بأسرع ما يمكن.

ينبغي أن تتعوّد على إعطاء الأولوية لكود يُترجم، ثم لكود ينجح في الاختبارات.

فبإضافة وظائف أكثر (مثل مخازن stub) بينما الكود لا يُترجم، نفتح على أنفسنا باب _مزيد_ من مشكلات الترجمة المحتملة.

الآن لن يُترجم `main.go` للسبب نفسه.

```go
func main() {
	server := &PlayerServer{}
	log.Fatal(http.ListenAndServe(":5000", server))
}
```

أخيرًا، كل شيء يُترجم لكن الاختبارات تفشل

```
=== RUN   TestGETPlayers/returns_the_Pepper's_score
panic: runtime error: invalid memory address or nil pointer dereference [recovered]
    panic: runtime error: invalid memory address or nil pointer dereference
```

هذا لأننا لم نمرّر `PlayerStore` في اختباراتنا. وسنحتاج إلى صنع واحد من نوع stub.

```go
//server_test.go
type StubPlayerStore struct {
	scores map[string]int
}

func (s *StubPlayerStore) GetPlayerScore(name string) int {
	score := s.scores[name]
	return score
}
```

الـ `map` طريقة سريعة وسهلة لصنع مخزن stub بمفاتيح وقيم لاختباراتنا. ولننشئ الآن واحدًا من هذه المخازن لاختباراتنا ونمرّره إلى `PlayerServer`.

```go
//server_test.go
func TestGETPlayers(t *testing.T) {
	store := StubPlayerStore{
		map[string]int{
			"Pepper": 20,
			"Floyd":  10,
		},
	}
	server := &PlayerServer{&store}

	t.Run("returns Pepper's score", func(t *testing.T) {
		request := newGetScoreRequest("Pepper")
		response := httptest.NewRecorder()

		server.ServeHTTP(response, request)

		assertResponseBody(t, response.Body.String(), "20")
	})

	t.Run("returns Floyd's score", func(t *testing.T) {
		request := newGetScoreRequest("Floyd")
		response := httptest.NewRecorder()

		server.ServeHTTP(response, request)

		assertResponseBody(t, response.Body.String(), "10")
	})
}
```

تنجح اختباراتنا الآن وتبدو أفضل. فقد أصبحت _النية_ خلف كودنا أوضح بفضل إدخال المخزن. فنحن نقول للقارئ إنه بسبب امتلاكنا _هذه البيانات في `PlayerStore`_ فإنك عند استخدامها مع `PlayerServer` ينبغي أن تحصل على الاستجابات التالية.

### تشغيل التطبيق

الآن بعد أن تنجح اختباراتنا، آخر ما نحتاج إلى فعله لإكمال إعادة الهيكلة هو التحقق من أن تطبيقنا يعمل. من المفترض أن يبدأ البرنامج، لكنك ستحصل على استجابة مروعة إذا حاولت الوصول إلى الخادم على `http://localhost:5000/players/Pepper`.

والسبب في ذلك أننا لم نمرّر `PlayerStore`.

سنحتاج إلى عمل تنفيذ له، لكن هذا صعب الآن لأننا لا نخزّن أي بيانات ذات معنى، لذا سيكون مكتوبًا مباشرة في الوقت الحالي.

```go
//main.go
type InMemoryPlayerStore struct{}

func (i *InMemoryPlayerStore) GetPlayerScore(name string) int {
	return 123
}

func main() {
	server := &PlayerServer{&InMemoryPlayerStore{}}
	log.Fatal(http.ListenAndServe(":5000", server))
}
```

إذا شغّلت `go build` مرة أخرى ووصلت إلى العنوان نفسه، ينبغي أن تحصل على `"123"`. ليس رائعًا، لكنه أفضل ما نستطيع فعله حتى نخزّن البيانات.
ولم يبدُ جيدًا أيضًا أن تطبيقنا الرئيسي كان يبدأ لكنه لا يعمل فعليًا. فقد اضطررنا إلى الاختبار اليدوي لنرى المشكلة.

لدينا خيارات قليلة لما نفعله بعد ذلك

-   التعامل مع سيناريو عدم وجود اللاعب
-   التعامل مع سيناريو `POST /players/{name}`

وبينما يقرّبنا سيناريو `POST` من "المسار السعيد"، أشعر أن معالجة سيناريو اللاعب المفقود أولًا ستكون أسهل لأننا في هذا السياق بالفعل. وسنصل إلى البقية لاحقًا.

## اكتب الاختبار أولًا

أضف سيناريو لاعب مفقود إلى مجموعتنا الحالية

```go
//server_test.go
t.Run("returns 404 on missing players", func(t *testing.T) {
	request := newGetScoreRequest("Apollo")
	response := httptest.NewRecorder()

	server.ServeHTTP(response, request)

	got := response.Code
	want := http.StatusNotFound

	if got != want {
		t.Errorf("got status %d want %d", got, want)
	}
})
```

## جرّب تشغيل الاختبار

```
=== RUN   TestGETPlayers/returns_404_on_missing_players
    --- FAIL: TestGETPlayers/returns_404_on_missing_players (0.00s)
        server_test.go:56: got status 200 want 404
```

## اكتب كودًا كافيًا لنجاح الاختبار

```go
//server.go
func (p *PlayerServer) ServeHTTP(w http.ResponseWriter, r *http.Request) {
	player := strings.TrimPrefix(r.URL.Path, "/players/")

	w.WriteHeader(http.StatusNotFound)

	fmt.Fprint(w, p.store.GetPlayerScore(player))
}
```

أحيانًا أدوّر عينيّ كثيرًا عندما يقول دعاة التطوير الموجه بالاختبار: "تأكد أن تكتب أصغر قدر من الكود لنجاح الاختبار"، فقد يبدو ذلك متزمتًا جدًا.

لكن هذا السيناريو يوضّح الفكرة جيدًا. لقد فعلت الحد الأدنى (وأنا أعلم أنه غير صحيح)، وهو كتابة `StatusNotFound` في **كل الاستجابات**، ومع ذلك تنجح كل اختباراتنا!

**وبفعل الحد الأدنى لنجاح الاختبارات يمكنك كشف ثغرات في اختباراتك**. في حالتنا، نحن لا نتحقق من أنه ينبغي أن نحصل على `StatusOK` عندما _توجد_ اللاعبون في المخزن.

حدّث الاختبارين الآخرين للتحقق من الحالة (status) وأصلح الكود.

إليك الاختبارات الجديدة

```go
//server_test.go
func TestGETPlayers(t *testing.T) {
	store := StubPlayerStore{
		map[string]int{
			"Pepper": 20,
			"Floyd":  10,
		},
	}
	server := &PlayerServer{&store}

	t.Run("returns Pepper's score", func(t *testing.T) {
		request := newGetScoreRequest("Pepper")
		response := httptest.NewRecorder()

		server.ServeHTTP(response, request)

		assertStatus(t, response.Code, http.StatusOK)
		assertResponseBody(t, response.Body.String(), "20")
	})

	t.Run("returns Floyd's score", func(t *testing.T) {
		request := newGetScoreRequest("Floyd")
		response := httptest.NewRecorder()

		server.ServeHTTP(response, request)

		assertStatus(t, response.Code, http.StatusOK)
		assertResponseBody(t, response.Body.String(), "10")
	})

	t.Run("returns 404 on missing players", func(t *testing.T) {
		request := newGetScoreRequest("Apollo")
		response := httptest.NewRecorder()

		server.ServeHTTP(response, request)

		assertStatus(t, response.Code, http.StatusNotFound)
	})
}

func assertStatus(t testing.TB, got, want int) {
	t.Helper()
	if got != want {
		t.Errorf("did not get correct status, got %d, want %d", got, want)
	}
}

func newGetScoreRequest(name string) *http.Request {
	req, _ := http.NewRequest(http.MethodGet, fmt.Sprintf("/players/%s", name), nil)
	return req
}

func assertResponseBody(t testing.TB, got, want string) {
	t.Helper()
	if got != want {
		t.Errorf("response body is wrong, got %q want %q", got, want)
	}
}
```

نتحقق الآن من الحالة في كل اختباراتنا، لذا صنعت مساعدًا `assertStatus` لتسهيل ذلك.

الآن يفشل أول اختبارين لدينا بسبب 404 بدلًا من 200، فيمكننا إصلاح `PlayerServer` ليرجع "غير موجود" فقط إذا كانت النقاط 0.

```go
//server.go
func (p *PlayerServer) ServeHTTP(w http.ResponseWriter, r *http.Request) {
	player := strings.TrimPrefix(r.URL.Path, "/players/")

	score := p.store.GetPlayerScore(player)

	if score == 0 {
		w.WriteHeader(http.StatusNotFound)
	}

	fmt.Fprint(w, score)
}
```

### تخزين النقاط

الآن بعد أن أصبحنا نجلب النقاط من مخزن، صار من المنطقي أن نستطيع تخزين نقاط جديدة.

## اكتب الاختبار أولًا

```go
//server_test.go
func TestStoreWins(t *testing.T) {
	store := StubPlayerStore{
		map[string]int{},
	}
	server := &PlayerServer{&store}

	t.Run("it returns accepted on POST", func(t *testing.T) {
		request, _ := http.NewRequest(http.MethodPost, "/players/Pepper", nil)
		response := httptest.NewRecorder()

		server.ServeHTTP(response, request)

		assertStatus(t, response.Code, http.StatusAccepted)
	})
}
```

في البداية لنتحقق فقط من أننا نحصل على رمز الحالة الصحيح إذا وصلنا إلى المسار المحدد بـ POST. وهذا يتيح لنا استنباط وظيفة قبول نوع مختلف من الطلبات ومعالجته بشكل مختلف عن `GET /players/{name}`. وبعد أن يعمل هذا، نبدأ في التحقق من تفاعل معالجنا مع المخزن.

## جرّب تشغيل الاختبار

```
=== RUN   TestStoreWins/it_returns_accepted_on_POST
    --- FAIL: TestStoreWins/it_returns_accepted_on_POST (0.00s)
        server_test.go:70: did not get correct status, got 404, want 202
```

## اكتب كودًا كافيًا لنجاح الاختبار

تذكّر أننا نرتكب الذنوب عن قصد، فجملة `if` مبنية على method الطلب ستؤدي الغرض.

```go
//server.go
func (p *PlayerServer) ServeHTTP(w http.ResponseWriter, r *http.Request) {

	if r.Method == http.MethodPost {
		w.WriteHeader(http.StatusAccepted)
		return
	}

	player := strings.TrimPrefix(r.URL.Path, "/players/")

	score := p.store.GetPlayerScore(player)

	if score == 0 {
		w.WriteHeader(http.StatusNotFound)
	}

	fmt.Fprint(w, score)
}
```

## إعادة الهيكلة

أصبح المعالج مشوشًا بعض الشيء الآن. لنقسّم الكود ليكون أسهل في المتابعة، ونعزل الوظائف المختلفة في دوال جديدة.

```go
//server.go
func (p *PlayerServer) ServeHTTP(w http.ResponseWriter, r *http.Request) {

	switch r.Method {
	case http.MethodPost:
		p.processWin(w)
	case http.MethodGet:
		p.showScore(w, r)
	}

}

func (p *PlayerServer) showScore(w http.ResponseWriter, r *http.Request) {
	player := strings.TrimPrefix(r.URL.Path, "/players/")

	score := p.store.GetPlayerScore(player)

	if score == 0 {
		w.WriteHeader(http.StatusNotFound)
	}

	fmt.Fprint(w, score)
}

func (p *PlayerServer) processWin(w http.ResponseWriter) {
	w.WriteHeader(http.StatusAccepted)
}
```

هذا يجعل جانب التوجيه في `ServeHTTP` أوضح قليلًا، ويعني أن تكراراتنا التالية في التخزين يمكن أن تكون داخل `processWin` فقط.

بعدها، نريد التحقق من أننا عندما ننفذ `POST /players/{name}` يُطلب من `PlayerStore` تسجيل الفوز.

## اكتب الاختبار أولًا

يمكننا تحقيق ذلك بتوسيع `StubPlayerStore` بـ method جديد اسمه `RecordWin` ثم التجسس على استدعاءاته.

```go
//server_test.go
type StubPlayerStore struct {
	scores   map[string]int
	winCalls []string
}

func (s *StubPlayerStore) GetPlayerScore(name string) int {
	score := s.scores[name]
	return score
}

func (s *StubPlayerStore) RecordWin(name string) {
	s.winCalls = append(s.winCalls, name)
}
```

لنوسّع الآن اختبارنا للتحقق من عدد الاستدعاءات في البداية

```go
//server_test.go
func TestStoreWins(t *testing.T) {
	store := StubPlayerStore{
		map[string]int{},
	}
	server := &PlayerServer{&store}

	t.Run("it records wins when POST", func(t *testing.T) {
		request := newPostWinRequest("Pepper")
		response := httptest.NewRecorder()

		server.ServeHTTP(response, request)

		assertStatus(t, response.Code, http.StatusAccepted)

		if len(store.winCalls) != 1 {
			t.Errorf("got %d calls to RecordWin want %d", len(store.winCalls), 1)
		}
	})
}

func newPostWinRequest(name string) *http.Request {
	req, _ := http.NewRequest(http.MethodPost, fmt.Sprintf("/players/%s", name), nil)
	return req
}
```

## جرّب تشغيل الاختبار

```
./server_test.go:26:20: too few values in struct initializer
./server_test.go:65:20: too few values in struct initializer
```

## اكتب أصغر قدر من الكود ليعمل الاختبار وافحص مخرجاته الفاشلة

نحتاج إلى تحديث كودنا حيث ننشئ `StubPlayerStore` لأننا أضفنا حقلًا جديدًا

```go
//server_test.go
store := StubPlayerStore{
	map[string]int{},
	nil,
}
```

```
--- FAIL: TestStoreWins (0.00s)
    --- FAIL: TestStoreWins/it_records_wins_when_POST (0.00s)
        server_test.go:80: got 0 calls to RecordWin want 1
```

## اكتب كودًا كافيًا لنجاح الاختبار

لأننا نتحقق فقط من عدد الاستدعاءات لا من القيم المحددة، فهذا يجعل تكرارنا الأولى أصغر قليلًا.

نحتاج إلى تحديث تصوّر `PlayerServer` لمعنى `PlayerStore` بتغيير الواجهة إذا أردنا أن نستطيع استدعاء `RecordWin`.

```go
//server.go
type PlayerStore interface {
	GetPlayerScore(name string) int
	RecordWin(name string)
}
```

وبفعلنا هذا لم يعد `main` يُترجم

```
./main.go:17:46: cannot use InMemoryPlayerStore literal (type *InMemoryPlayerStore) as type PlayerStore in field value:
    *InMemoryPlayerStore does not implement PlayerStore (missing RecordWin method)
```

يخبرنا المترجم بما هو الخطأ. لنحدّث `InMemoryPlayerStore` ليملك هذه الـ method.

```go
//main.go
type InMemoryPlayerStore struct{}

func (i *InMemoryPlayerStore) RecordWin(name string) {}
```

جرّب تشغيل الاختبارات، وينبغي أن نعود إلى كود يُترجم، لكن الاختبار ما زال يفشل.

الآن بعد أن أصبح لدى `PlayerStore` دالة `RecordWin` يمكننا استدعاؤها داخل `PlayerServer`

```go
//server.go
func (p *PlayerServer) processWin(w http.ResponseWriter) {
	p.store.RecordWin("Bob")
	w.WriteHeader(http.StatusAccepted)
}
```

شغّل الاختبارات ومن المفترض أن ينجح! من الواضح أن `"Bob"` ليس بالضبط ما نريد إرساله إلى `RecordWin`، فلنحسّن الاختبار أكثر.

## اكتب الاختبار أولًا

```go
//server_test.go
func TestStoreWins(t *testing.T) {
	store := StubPlayerStore{
		map[string]int{},
		nil,
	}
	server := &PlayerServer{&store}

	t.Run("it records wins on POST", func(t *testing.T) {
		player := "Pepper"

		request := newPostWinRequest(player)
		response := httptest.NewRecorder()

		server.ServeHTTP(response, request)

		assertStatus(t, response.Code, http.StatusAccepted)

		if len(store.winCalls) != 1 {
			t.Fatalf("got %d calls to RecordWin want %d", len(store.winCalls), 1)
		}

		if store.winCalls[0] != player {
			t.Errorf("did not store correct winner got %q want %q", store.winCalls[0], player)
		}
	})
}
```

الآن بعد أن نعرف أن في شريحة `winCalls` عنصرًا واحدًا، يمكننا بأمان الإشارة إلى الأول والتحقق من أنه يساوي `player`.

## جرّب تشغيل الاختبار

```
=== RUN   TestStoreWins/it_records_wins_on_POST
    --- FAIL: TestStoreWins/it_records_wins_on_POST (0.00s)
        server_test.go:86: did not store correct winner got 'Bob' want 'Pepper'
```

## اكتب كودًا كافيًا لنجاح الاختبار

```go
//server.go
func (p *PlayerServer) processWin(w http.ResponseWriter, r *http.Request) {
	player := strings.TrimPrefix(r.URL.Path, "/players/")
	p.store.RecordWin(player)
	w.WriteHeader(http.StatusAccepted)
}
```

غيّرنا `processWin` لتأخذ `http.Request` حتى نتمكن من النظر في الـ URL لاستخراج اسم اللاعب. وبعد أن نحصل عليه يمكننا استدعاء `store` بالقيمة الصحيحة لنجاح الاختبار.

## إعادة الهيكلة

يمكننا إزالة بعض التكرار في هذا الكود لأننا نستخرج اسم اللاعب بالطريقة نفسها في مكانين

```go
//server.go
func (p *PlayerServer) ServeHTTP(w http.ResponseWriter, r *http.Request) {
	player := strings.TrimPrefix(r.URL.Path, "/players/")

	switch r.Method {
	case http.MethodPost:
		p.processWin(w, player)
	case http.MethodGet:
		p.showScore(w, player)
	}
}

func (p *PlayerServer) showScore(w http.ResponseWriter, player string) {
	score := p.store.GetPlayerScore(player)

	if score == 0 {
		w.WriteHeader(http.StatusNotFound)
	}

	fmt.Fprint(w, score)
}

func (p *PlayerServer) processWin(w http.ResponseWriter, player string) {
	p.store.RecordWin(player)
	w.WriteHeader(http.StatusAccepted)
}
```

رغم أن اختباراتنا تنجح، فليس لدينا برنامج يعمل فعلًا. فإذا حاولت تشغيل `main` واستخدام البرنامج كما هو مقصود فلن يعمل لأننا لم نصل بعد إلى تنفيذ `PlayerStore` بشكل صحيح. لا بأس في ذلك؛ فبتركيزنا على معالجنا حدّدنا الواجهة التي نحتاجها، بدلًا من محاولة تصميمها مسبقًا.

ي_مكننا_ أن نبدأ بكتابة بعض الاختبارات حول `InMemoryPlayerStore`، لكنه موجود هنا مؤقتًا فقط حتى ننفّذ طريقة أكثر متانة لحفظ نقاط اللاعبين (أي قاعدة بيانات).

ما سنفعله الآن هو كتابة _اختبار تكامل (integration test)_ بين `PlayerServer` و`InMemoryPlayerStore` لإنهاء الوظيفة. وسيتيح لنا هذا الوصول إلى هدفنا المتمثل في الثقة بأن تطبيقنا يعمل، دون الحاجة إلى اختبار `InMemoryPlayerStore` مباشرة. وليس ذلك فقط، بل عندما نصل إلى تنفيذ `PlayerStore` بقاعدة بيانات، سنستطيع اختبار ذلك التنفيذ باختبار التكامل نفسه.

### اختبارات التكامل

يمكن أن تكون اختبارات التكامل مفيدة لاختبار أن مساحات أكبر من نظامك تعمل، لكن يجب أن تضع في اعتبارك:

-   أنها أصعب في الكتابة
-   عندما تفشل، قد يصعب معرفة السبب (وهو غالبًا خلل في أحد مكونات اختبار التكامل)، لذا قد يصعب إصلاحها
-   أنها أبطأ في التشغيل أحيانًا (لأنها تُستخدم غالبًا مع مكونات "حقيقية"، مثل قاعدة بيانات)

ولهذا السبب يُوصى بالبحث في _هرم الاختبارات (The Test Pyramid)_.

## اكتب الاختبار أولًا

حرصًا على الإيجاز، سأعرض عليك اختبار التكامل النهائي بعد إعادة الهيكلة.

```go
// server_integration_test.go
package main

import (
	"net/http"
	"net/http/httptest"
	"testing"
)

func TestRecordingWinsAndRetrievingThem(t *testing.T) {
	store := InMemoryPlayerStore{}
	server := PlayerServer{&store}
	player := "Pepper"

	server.ServeHTTP(httptest.NewRecorder(), newPostWinRequest(player))
	server.ServeHTTP(httptest.NewRecorder(), newPostWinRequest(player))
	server.ServeHTTP(httptest.NewRecorder(), newPostWinRequest(player))

	response := httptest.NewRecorder()
	server.ServeHTTP(response, newGetScoreRequest(player))
	assertStatus(t, response.Code, http.StatusOK)

	assertResponseBody(t, response.Body.String(), "3")
}
```

-   ننشئ مكوّنينا اللذين نحاول دمجهما: `InMemoryPlayerStore` و`PlayerServer`.
-   ثم نُطلق 3 طلبات لتسجيل 3 انتصارات لـ `player`. ولا نقلق كثيرًا بشأن رموز الحالة في هذا الاختبار لأنها غير ذات صلة بجودة التكامل.
-   أما الاستجابة التالية فنحن نهتم بها (لذا نخزّنها في متغير `response`) لأننا سنحاول جلب نقاط `player`.

## جرّب تشغيل الاختبار

```
--- FAIL: TestRecordingWinsAndRetrievingThem (0.00s)
    server_integration_test.go:24: response body is wrong, got '123' want '3'
```

## اكتب كودًا كافيًا لنجاح الاختبار

سآخذ هنا بعض الحريات وأكتب كودًا أكثر مما قد ترتاح إليه دون كتابة اختبار.

_هذا مسموح!_ فلا يزال لدينا اختبار يتحقق من أن الأمور ينبغي أن تعمل، لكنه ليس حول الوحدة المحددة التي نعمل عليها (`InMemoryPlayerStore`).

ولو تعثرت في هذا السيناريو، فسأعيد تغييراتي إلى الاختبار الفاشل ثم أكتب اختبارات وحدة أكثر تحديدًا حول `InMemoryPlayerStore` لمساعدتي على استنباط الحل.

```go
//in_memory_player_store.go
func NewInMemoryPlayerStore() *InMemoryPlayerStore {
	return &InMemoryPlayerStore{map[string]int{}}
}

type InMemoryPlayerStore struct {
	store map[string]int
}

func (i *InMemoryPlayerStore) RecordWin(name string) {
	i.store[name]++
}

func (i *InMemoryPlayerStore) GetPlayerScore(name string) int {
	return i.store[name]
}
```

-   نحتاج إلى تخزين البيانات، لذا أضفت `map[string]int` إلى struct الـ `InMemoryPlayerStore`
-   وللراحة صنعت `NewInMemoryPlayerStore` لتهيئة المخزن، وحدّثت اختبار التكامل ليستخدمه:
    ```go
    //server_integration_test.go
    store := NewInMemoryPlayerStore()
    server := PlayerServer{store}
    ```
-   وبقية الكود مجرد غلاف حول الـ `map`

ينجح اختبار التكامل، والآن نحتاج فقط إلى تغيير `main` ليستخدم `NewInMemoryPlayerStore()`.

```go
// main.go
package main

import (
	"log"
	"net/http"
)

func main() {
	server := &PlayerServer{NewInMemoryPlayerStore()}
	log.Fatal(http.ListenAndServe(":5000", server))
}
```

ابنِه وشغّله ثم استخدم `curl` لتجربته.

-   نفّذ هذا عدة مرات، وغيّر أسماء اللاعبين إن أردت `curl -X POST http://localhost:5000/players/Pepper`
-   افحص النقاط بـ `curl http://localhost:5000/players/Pepper`

رائع! لقد صنعت خدمة قريبة من REST. ولتطويرها أكثر، ستريد اختيار مخزن بيانات يحفظ النقاط مدة أطول من مدة تشغيل البرنامج.

-   اختر مخزنًا (Bolt؟ Mongo؟ Postgres؟ نظام الملفات؟)
-   اجعل `PostgresPlayerStore` يحقق `PlayerStore`
-   طبّق الوظيفة بالتطوير الموجه بالاختبار حتى تتأكد أنها تعمل
-   وصّلها باختبار التكامل، وتأكد أنها ما زالت بخير
-   وأخيرًا وصّلها بـ `main`

## إعادة الهيكلة

كدنا نصل! لنبذل بعض الجهد لمنع أخطاء الـ concurrency مثل هذه

```
fatal error: concurrent map read and map write
```

بإضافة mutexes نفرض أمان الـ concurrency، خصوصًا للعدّاد في دالتنا `RecordWin`. اقرأ المزيد عن mutexes في فصل sync.

## الخلاصة

### `http.Handler`

-   حقّق هذه الواجهة لإنشاء خوادم ويب
-   استخدم `http.HandlerFunc` لتحويل الدوال العادية إلى `http.Handler`
-   استخدم `httptest.NewRecorder` لتمريره كـ `ResponseWriter` يتيح لك التجسس على الاستجابات التي يرسلها معالجك
-   استخدم `http.NewRequest` لبناء الطلبات التي تتوقع وصولها إلى نظامك

### الواجهات والـ mocking وحقن الاعتماديات

-   تتيح لك بناء النظام بشكل تكراري في أجزاء أصغر
-   تسمح لك بتطوير معالج يحتاج إلى تخزين دون الحاجة إلى تخزين فعلي
-   استخدم التطوير الموجه بالاختبار لاستنباط الواجهات التي تحتاجها

### اقترف الذنوب ثم أعد الهيكلة (ثم سجّل في نظام التحكم في الإصدارات)

-   عليك أن تتعامل مع وجود ترجمة فاشلة أو اختبارات فاشلة كحالة حمراء تحتاج إلى الخروج منها بأسرع ما يمكن.
-   اكتب فقط الكود اللازم للوصول إلى هناك. _ثم_ أعد الهيكلة واجعل الكود جميلًا.
-   محاولة إجراء تغييرات كثيرة جدًا بينما الكود لا يُترجم أو الاختبارات تفشل تعرّضك لخطر مضاعفة المشكلات.
-   الالتزام بهذا النهج يدفعك إلى كتابة اختبارات صغيرة، ما يعني تغييرات صغيرة، ما يساعد على إبقاء العمل على الأنظمة المعقدة قابلًا للإدارة.
