---
title: JSON والتوجيه والدمج
weight: 280
---

# JSON والتوجيه والدمج

**[يمكنك العثور على كل كود هذا الفصل هنا](https://github.com/quii/learn-go-with-tests/tree/main/json)**

[في الفصل السابق](http-server.md) أنشأنا خادم ويب لتخزين عدد المباريات التي فاز بها اللاعبون.

ولمالك المنتج (product owner) متطلب جديد: نقطة نهاية (endpoint) جديدة اسمها `/league` تُرجع قائمة بكل اللاعبين المخزَّنين. وتريد أن تُعاد هذه القائمة على هيئة JSON.

## إليك الكود الذي وصلنا إليه حتى الآن

```go
// server.go
package main

import (
	"fmt"
	"net/http"
	"strings"
)

type PlayerStore interface {
	GetPlayerScore(name string) int
	RecordWin(name string)
}

type PlayerServer struct {
	store PlayerStore
}

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

```go
// in_memory_player_store.go
package main

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

يمكنك العثور على الاختبارات المقابلة في الرابط أعلى الفصل.

سنبدأ بإنشاء نقطة نهاية جدول الترتيب (league table).

## اكتب الاختبار أولًا

سنوسّع مجموعة الاختبارات الحالية، فلدينا بعض دوال الاختبار المفيدة و`PlayerStore` وهمي (fake) يمكن استخدامه.

```go
//server_test.go
func TestLeague(t *testing.T) {
	store := StubPlayerStore{}
	server := &PlayerServer{&store}

	t.Run("it returns 200 on /league", func(t *testing.T) {
		request, _ := http.NewRequest(http.MethodGet, "/league", nil)
		response := httptest.NewRecorder()

		server.ServeHTTP(response, request)

		assertStatus(t, response.Code, http.StatusOK)
	})
}
```

قبل أن نقلق بشأن النقاط الفعلية و JSON، سنحاول إبقاء التغييرات صغيرة مع خطة للتقدّم تدريجيًا نحو هدفنا. وأبسط بداية هي التحقق من أننا نستطيع الوصول إلى `/league` والحصول على استجابة `OK`.

## جرّب تشغيل الاختبار

```
    --- FAIL: TestLeague/it_returns_200_on_/league (0.00s)
        server_test.go:101: status code is wrong: got 404, want 200
FAIL
FAIL	playerstore	0.221s
FAIL
```

الـ `PlayerServer` لدينا يُرجع `404 Not Found`، وكأننا نحاول جلب انتصارات لاعب غير معروف. وبالنظر إلى كيفية تنفيذ `ServeHTTP` في `server.go`، ندرك أنه يفترض دائمًا أن يُستدعى بعنوان URL يشير إلى لاعب بعينه:

```go
player := strings.TrimPrefix(r.URL.Path, "/players/")
```

ذكرنا في الفصل السابق أن هذه طريقة ساذجة نوعًا ما للتوجيه (routing). ويخبرنا اختبارنا على نحو صحيح بأننا نحتاج إلى مفهوم للتعامل مع مسارات الطلبات المختلفة.

## اكتب كودًا كافيًا لنجاح الاختبار

تمتلك Go آلية توجيه مدمجة اسمها [`ServeMux`](https://golang.org/pkg/net/http/#ServeMux) (موزّع الطلبات، request multiplexer) تتيح لك ربط معالجات `http.Handler` بمسارات طلبات معيّنة.

لنقترف بعض الذنوب البرمجية ونجعل الاختبارات تنجح بأسرع طريقة ممكنة، مع علمنا أننا نستطيع إعادة الهيكلة بأمان بمجرد التأكد من نجاحها.

```go
//server.go
func (p *PlayerServer) ServeHTTP(w http.ResponseWriter, r *http.Request) {

	router := http.NewServeMux()

	router.Handle("/league", http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		w.WriteHeader(http.StatusOK)
	}))

	router.Handle("/players/", http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		player := strings.TrimPrefix(r.URL.Path, "/players/")

		switch r.Method {
		case http.MethodPost:
			p.processWin(w, player)
		case http.MethodGet:
			p.showScore(w, player)
		}
	}))

	router.ServeHTTP(w, r)
}
```

- عند بدء الطلب ننشئ موجّهًا (router) ثم نخبره بأن يستخدم المعالج `y` للمسار `x`.
- ولكي ينجح اختبارنا الجديد، نستخدم `http.HandlerFunc` و_دالة مجهولة (anonymous function)_ لتنفيذ `w.WriteHeader(http.StatusOK)` عند طلب `/league`.
- أما مسار `/players/` فقد اكتفينا بنسخ كودنا ولصقه داخل `http.HandlerFunc` آخر.
- وأخيرًا، نعالج الطلب الوارد باستدعاء `ServeHTTP` الخاص بالموجّه الجديد (هل لاحظت أن `ServeMux` هو _أيضًا_ `http.Handler`؟)

من المفترض أن تنجح الاختبارات الآن.

## إعادة الهيكلة

أصبح `ServeHTTP` كبيرًا نوعًا ما، ويمكننا فصل الأمور قليلًا بإعادة هيكلة المعالجات إلى methods منفصلة.

```go
//server.go
func (p *PlayerServer) ServeHTTP(w http.ResponseWriter, r *http.Request) {

	router := http.NewServeMux()
	router.Handle("/league", http.HandlerFunc(p.leagueHandler))
	router.Handle("/players/", http.HandlerFunc(p.playersHandler))

	router.ServeHTTP(w, r)
}

func (p *PlayerServer) leagueHandler(w http.ResponseWriter, r *http.Request) {
	w.WriteHeader(http.StatusOK)
}

func (p *PlayerServer) playersHandler(w http.ResponseWriter, r *http.Request) {
	player := strings.TrimPrefix(r.URL.Path, "/players/")

	switch r.Method {
	case http.MethodPost:
		p.processWin(w, player)
	case http.MethodGet:
		p.showScore(w, player)
	}
}
```

من الغريب (وغير الفعّال) أن نُعدّ موجّهًا مع كل طلب وارد ثم نستدعيه. وما نريده حقًا هو وجود دالة مثل `NewPlayerServer` تأخذ اعتمادياتنا وتقوم بإعداد الموجّه مرة واحدة. وبعد ذلك يمكن لكل طلب أن يستخدم نسخة الموجّه الواحدة تلك.

```go
//server.go
type PlayerServer struct {
	store  PlayerStore
	router *http.ServeMux
}

func NewPlayerServer(store PlayerStore) *PlayerServer {
	p := &PlayerServer{
		store,
		http.NewServeMux(),
	}

	p.router.Handle("/league", http.HandlerFunc(p.leagueHandler))
	p.router.Handle("/players/", http.HandlerFunc(p.playersHandler))

	return p
}

func (p *PlayerServer) ServeHTTP(w http.ResponseWriter, r *http.Request) {
	p.router.ServeHTTP(w, r)
}
```

- أصبح `PlayerServer` الآن بحاجة إلى تخزين موجّه.
- نقلنا إنشاء التوجيه من داخل `ServeHTTP` إلى `NewPlayerServer` بحيث يحدث مرة واحدة فقط، لا مع كل طلب.
- ستحتاج إلى تحديث كل كود الاختبار وكود الإنتاج حيث كنا نكتب `PlayerServer{&store}` لتصبح `NewPlayerServer(&store)`.

### إعادة هيكلة أخيرة

جرّب تغيير الكود إلى ما يلي.

```go
type PlayerServer struct {
	store PlayerStore
	http.Handler
}

func NewPlayerServer(store PlayerStore) *PlayerServer {
	p := new(PlayerServer)

	p.store = store

	router := http.NewServeMux()
	router.Handle("/league", http.HandlerFunc(p.leagueHandler))
	router.Handle("/players/", http.HandlerFunc(p.playersHandler))

	p.Handler = router

	return p
}
```

ثم استبدل `server := &PlayerServer{&store}` بـ `server := NewPlayerServer(&store)` في `server_test.go` و`server_integration_test.go` و`main.go`.

وأخيرًا تأكد من **حذف** `func (p *PlayerServer) ServeHTTP(w http.ResponseWriter, r *http.Request)` فقد لم تعد هناك حاجة إليه!

## الدمج (Embedding)

غيّرنا الحقل الثاني في `PlayerServer`، فحذفنا الحقل المسمّى `router http.ServeMux` واستبدلناه بـ `http.Handler`؛ وهذا ما يسمى _الدمج (embedding)_.

> لا تقدّم Go المفهوم النمطي القائم على الأنواع للوراثة (subclassing)، لكنها تملك قدرة "استعارة" أجزاء من تنفيذ ما عبر دمج أنواع داخل struct أو واجهة (interface).

[Effective Go - Embedding](https://golang.org/doc/effective_go.html#embedding)

معنى ذلك أن `PlayerServer` لدينا يملك الآن كل الـ methods التي يملكها `http.Handler`، وهي ليست سوى `ServeHTTP`.

ولـ"تعبئة" الـ `http.Handler` نسنده إلى `router` الذي ننشئه في `NewPlayerServer`. ونستطيع فعل ذلك لأن `http.ServeMux` يملك الـ method `ServeHTTP`.

وهذا يتيح لنا حذف الـ method `ServeHTTP` الخاصة بنا، لأننا نكشف واحدة بالفعل عبر النوع المدمَج.

الدمج ميزة لغوية مثيرة للاهتمام جدًا. ويمكنك استخدامه مع الواجهات (interfaces) لتكوين واجهات جديدة.

```go
type Animal interface {
	Eater
	Sleeper
}
```

ويمكنك استخدامه مع الأنواع الملموسة (concrete types) أيضًا، لا مع الواجهات فقط. وكما تتوقع، إذا دمجت نوعًا ملموسًا فستصل إلى كل الـ methods والحقول (fields) العامة فيه.

### هل هناك عيوب؟

يجب أن تكون حذرًا مع دمج الأنواع، لأنك ستكشف كل الـ methods والحقول العامة للنوع الذي تدمجه. وفي حالتنا لا بأس، لأننا دمجنا فقط _الواجهة_ التي أردنا كشفها (`http.Handler`).

ولو كنا متساهلين ودمجنا `http.ServeMux` بدلًا منها (النوع الملموس) فسيعمل ذلك _لكن_ سيتمكن مستخدمو `PlayerServer` من إضافة مسارات جديدة إلى خادمنا لأن `Handle(path, handler)` سيكون عامًا.

**عند دمج الأنواع، فكّر جيدًا في تأثير ذلك على واجهتك العامة (public API).**

من الأخطاء _الشائعة جدًا_ إساءة استخدام الدمج، بحيث ينتهي بك الأمر إلى تلويث واجهاتك العامة وكشف داخليات نوعك.

وبعد أن أعدنا هيكلة تطبيقنا، يمكننا بسهولة إضافة مسارات جديدة، وأصبح لدينا بداية نقطة النهاية `/league`. والآن نحتاج إلى جعلها تُرجع معلومات مفيدة.

ينبغي أن نُرجع JSON يشبه شيئًا كهذا.

```json
[
   {
      "Name":"Bill",
      "Wins":10
   },
   {
      "Name":"Alice",
      "Wins":15
   }
]
```

## اكتب الاختبار أولًا

سنبدأ بمحاولة تحليل الاستجابة (parse) إلى شيء ذي معنى.

```go
//server_test.go
func TestLeague(t *testing.T) {
	store := StubPlayerStore{}
	server := NewPlayerServer(&store)

	t.Run("it returns 200 on /league", func(t *testing.T) {
		request, _ := http.NewRequest(http.MethodGet, "/league", nil)
		response := httptest.NewRecorder()

		server.ServeHTTP(response, request)

		var got []Player

		err := json.NewDecoder(response.Body).Decode(&got)

		if err != nil {
			t.Fatalf("Unable to parse response from server %q into slice of Player, '%v'", response.Body, err)
		}

		assertStatus(t, response.Code, http.StatusOK)
	})
}
```

### لماذا لا نختبر نص JSON نفسه؟

يمكنك القول إن خطوة أولى أبسط تتمثل في التحقق فقط من أن جسم الاستجابة يحتوي على نص JSON معيّن.

ومن تجربتي، فإن الاختبارات التي تتحقق من نصوص JSON تعاني المشكلات التالية.

- *الهشاشة*. إذا غيّرت نموذج البيانات (data-model) ستفشل اختباراتك.
- *صعوبة التنقيح*. قد يكون من الصعب فهم المشكلة الحقيقية عند مقارنة نصي JSON.
- *ضعف التعبير عن القصد*. فرغم أن الناتج ينبغي أن يكون JSON، فالأهم حقًا هو ما هي البيانات بالضبط، لا كيف تم ترميزها.
- *إعادة اختبار المكتبة القياسية*. لا حاجة إلى اختبار كيفية إخراج المكتبة القياسية لـ JSON، فهي مختبرة سلفًا. لا تختبر كود الآخرين.

وبدلًا من ذلك، ينبغي أن نحاول تحليل JSON إلى بنى بيانات (data structures) تصلح للاختبار بها.

### نمذجة البيانات

بالنظر إلى نموذج بيانات JSON، يبدو أننا نحتاج إلى مصفوفة (array) من `Player` مع بعض الحقول، لذا أنشأنا نوعًا جديدًا للتعبير عن ذلك.

```go
//server.go
type Player struct {
	Name string
	Wins int
}
```

### فك ترميز JSON

```go
//server_test.go
var got []Player
err := json.NewDecoder(response.Body).Decode(&got)
```

لتحليل JSON إلى نموذج بياناتنا ننشئ `Decoder` من حزمة `encoding/json` ثم نستدعي الـ method `Decode` الخاصة به. ولكي ننشئ `Decoder` فإنه يحتاج إلى `io.Reader` يقرأ منه، وهو في حالتنا الحقل `Body` في الـ spy الخاص بالاستجابة (response spy).

تأخذ `Decode` عنوان (address) الشيء الذي نحاول فك الترميز إليه، ولهذا نعرّف شريحة فارغة من `Player` في السطر السابق.

قد يفشل تحليل JSON، لذا يمكن أن تُرجع `Decode` خطأ (`error`). ولا جدوى من متابعة الاختبار إذا فشل ذلك، لذا نتحقق من الخطأ ونوقف الاختبار بـ `t.Fatalf` إذا حدث. ولاحظ أننا نطبع جسم الاستجابة مع الخطأ، لأن من المهم لمن يشغّل الاختبار أن يرى النص الذي تعذّر تحليله.

## جرّب تشغيل الاختبار

```
=== RUN   TestLeague/it_returns_200_on_/league
    --- FAIL: TestLeague/it_returns_200_on_/league (0.00s)
        server_test.go:107: Unable to parse response from server '' into slice of Player, 'unexpected end of JSON input'
```

نقطة النهاية لدينا لا تُرجع جسمًا حاليًا، لذا لا يمكن تحليله إلى JSON.

## اكتب كودًا كافيًا لنجاح الاختبار

```go
//server.go
func (p *PlayerServer) leagueHandler(w http.ResponseWriter, r *http.Request) {
	leagueTable := []Player{
		{"Chris", 20},
	}

	json.NewEncoder(w).Encode(leagueTable)

	w.WriteHeader(http.StatusOK)
}
```

ينجح الاختبار الآن.

### الترميز وفك الترميز

لاحظ التناظر الجميل في المكتبة القياسية.

- لإنشاء `Encoder` تحتاج إلى `io.Writer`، وهو ما تنفذه `http.ResponseWriter`.
- ولإنشاء `Decoder` تحتاج إلى `io.Reader`، وهو ما ينفذه الحقل `Body` في الـ spy الخاص بالاستجابة لدينا.

استخدمنا `io.Writer` في أنحاء هذا الكتاب، وهذا عرض آخر لانتشاره في المكتبة القياسية وسهولة تعامل كثير من المكتبات معه.

## إعادة الهيكلة

سيكون جميلًا أن نفصل الاهتمامات (separation of concern) بين المعالج وبين الحصول على `leagueTable`، لأننا نعلم أننا لن نكتبها مباشرة في الكود قريبًا.

```go
//server.go
func (p *PlayerServer) leagueHandler(w http.ResponseWriter, r *http.Request) {
	json.NewEncoder(w).Encode(p.getLeagueTable())
	w.WriteHeader(http.StatusOK)
}

func (p *PlayerServer) getLeagueTable() []Player {
	return []Player{
		{"Chris", 20},
	}
}
```

بعد ذلك، سنريد توسيع اختبارنا حتى نتحكم بالضبط في البيانات التي نريد استرجاعها.

## اكتب الاختبار أولًا

يمكننا تحديث الاختبار للتحقق من أن جدول الترتيب يحتوي على بعض اللاعبين الذين سنضع لهم stub في مخزننا (store).

حدّث `StubPlayerStore` ليتيح تخزين جدول ترتيب، وهو مجرد شريحة (slice) من `Player`. وسنخزّن فيه بياناتنا المتوقعة.

```go
//server_test.go
type StubPlayerStore struct {
	scores   map[string]int
	winCalls []string
	league   []Player
}
```

بعد ذلك، حدّث اختبارنا الحالي بوضع بعض اللاعبين في خاصية league في الـ stub، وتحقق من أن خادمنا يُرجعهم.

```go
//server_test.go
func TestLeague(t *testing.T) {

	t.Run("it returns the league table as JSON", func(t *testing.T) {
		wantedLeague := []Player{
			{"Cleo", 32},
			{"Chris", 20},
			{"Tiest", 14},
		}

		store := StubPlayerStore{nil, nil, wantedLeague}
		server := NewPlayerServer(&store)

		request, _ := http.NewRequest(http.MethodGet, "/league", nil)
		response := httptest.NewRecorder()

		server.ServeHTTP(response, request)

		var got []Player

		err := json.NewDecoder(response.Body).Decode(&got)

		if err != nil {
			t.Fatalf("Unable to parse response from server %q into slice of Player, '%v'", response.Body, err)
		}

		assertStatus(t, response.Code, http.StatusOK)

		if !reflect.DeepEqual(got, wantedLeague) {
			t.Errorf("got %v want %v", got, wantedLeague)
		}
	})
}
```

## جرّب تشغيل الاختبار

```
./server_test.go:33:3: too few values in struct initializer
./server_test.go:70:3: too few values in struct initializer
```

## اكتب أصغر قدر من الكود ليعمل الاختبار وافحص مخرجاته الفاشلة

ستحتاج إلى تحديث الاختبارات الأخرى لأننا أضفنا حقلًا جديدًا في `StubPlayerStore`؛ فاضبطه على nil في الاختبارات الأخرى.

جرّب تشغيل الاختبارات مرة أخرى، ومن المفترض أن تحصل على

```
=== RUN   TestLeague/it_returns_the_league_table_as_JSON
    --- FAIL: TestLeague/it_returns_the_league_table_as_JSON (0.00s)
        server_test.go:124: got [{Chris 20}] want [{Cleo 32} {Chris 20} {Tiest 14}]
```

## اكتب كودًا كافيًا لنجاح الاختبار

نعلم أن البيانات موجودة في `StubPlayerStore`، وقد جرّدناها في واجهة `PlayerStore`. وعلينا تحديث هذه الواجهة حتى يستطيع أي شخص يمرر لنا `PlayerStore` أن يزوّدنا ببيانات جدول الترتيب.

```go
//server.go
type PlayerStore interface {
	GetPlayerScore(name string) int
	RecordWin(name string)
	GetLeague() []Player
}
```

الآن يمكننا تحديث كود المعالج ليستدعي ذلك بدلًا من إرجاع قائمة مكتوبة مباشرة في الكود. احذف الـ method `getLeagueTable()` ثم حدّث `leagueHandler` ليستدعي `GetLeague()`.

```go
//server.go
func (p *PlayerServer) leagueHandler(w http.ResponseWriter, r *http.Request) {
	json.NewEncoder(w).Encode(p.store.GetLeague())
	w.WriteHeader(http.StatusOK)
}
```

جرّب تشغيل الاختبارات.

```
# github.com/quii/learn-go-with-tests/json-and-io/v4
./main.go:9:50: cannot use NewInMemoryPlayerStore() (type *InMemoryPlayerStore) as type PlayerStore in argument to NewPlayerServer:
    *InMemoryPlayerStore does not implement PlayerStore (missing GetLeague method)
./server_integration_test.go:11:27: cannot use store (type *InMemoryPlayerStore) as type PlayerStore in argument to NewPlayerServer:
    *InMemoryPlayerStore does not implement PlayerStore (missing GetLeague method)
./server_test.go:36:28: cannot use &store (type *StubPlayerStore) as type PlayerStore in argument to NewPlayerServer:
    *StubPlayerStore does not implement PlayerStore (missing GetLeague method)
./server_test.go:74:28: cannot use &store (type *StubPlayerStore) as type PlayerStore in argument to NewPlayerServer:
    *StubPlayerStore does not implement PlayerStore (missing GetLeague method)
./server_test.go:106:29: cannot use &store (type *StubPlayerStore) as type PlayerStore in argument to NewPlayerServer:
    *StubPlayerStore does not implement PlayerStore (missing GetLeague method)
```

يشتكي المترجم لأن `InMemoryPlayerStore` و`StubPlayerStore` لا يملكان الـ method الجديدة التي أضفناها إلى واجهتنا.

بالنسبة إلى `StubPlayerStore` فالأمر سهل جدًا، ما عليك سوى إرجاع حقل `league` الذي أضفناه سابقًا.

```go
//server_test.go
func (s *StubPlayerStore) GetLeague() []Player {
	return s.league
}
```

وهذا تذكير بكيفية تنفيذ `InMemoryStore`.

```go
//in_memory_player_store.go
type InMemoryPlayerStore struct {
	store map[string]int
}
```

ومع أن تنفيذ `GetLeague` "بشكل سليم" بالمرور على الـ map سيكون مباشرًا، تذكّر أننا نحاول فقط _كتابة أصغر قدر من الكود لنجاح الاختبارات_.

لذا لنسعد المترجم الآن فقط، ولنتعايش مع الشعور المزعج بتنفيذ ناقص في `InMemoryStore`.

```go
//in_memory_player_store.go
func (i *InMemoryPlayerStore) GetLeague() []Player {
	return nil
}
```

وما يخبرنا به ذلك حقًا هو أننا سنريد _لاحقًا_ اختبار هذا، لكن لنؤجّل ذلك الآن.

جرّب تشغيل الاختبارات، من المفترض أن يمر المترجم وتنجح الاختبارات!

## إعادة الهيكلة

لا يعبّر كود الاختبار عن قصدنا جيدًا، وفيه الكثير من الكود المتكرر (boilerplate) الذي يمكننا إزالته بإعادة الهيكلة.

```go
//server_test.go
t.Run("it returns the league table as JSON", func(t *testing.T) {
	wantedLeague := []Player{
		{"Cleo", 32},
		{"Chris", 20},
		{"Tiest", 14},
	}

	store := StubPlayerStore{nil, nil, wantedLeague}
	server := NewPlayerServer(&store)

	request := newLeagueRequest()
	response := httptest.NewRecorder()

	server.ServeHTTP(response, request)

	got := getLeagueFromResponse(t, response.Body)
	assertStatus(t, response.Code, http.StatusOK)
	assertLeague(t, got, wantedLeague)
})
```

وإليك دوال المساعدة الجديدة

```go
//server_test.go
func getLeagueFromResponse(t testing.TB, body io.Reader) (league []Player) {
	t.Helper()
	err := json.NewDecoder(body).Decode(&league)

	if err != nil {
		t.Fatalf("Unable to parse response from server %q into slice of Player, '%v'", body, err)
	}

	return
}

func assertLeague(t testing.TB, got, want []Player) {
	t.Helper()
	if !reflect.DeepEqual(got, want) {
		t.Errorf("got %v want %v", got, want)
	}
}

func newLeagueRequest() *http.Request {
	req, _ := http.NewRequest(http.MethodGet, "/league", nil)
	return req
}
```

بقي أمر أخير لنعمل خادمنا، وهو التأكد من إرجاع ترويسة (header) اسمها `content-type` في الاستجابة حتى تتعرف الأجهزة على أننا نُرجع `JSON`.

## اكتب الاختبار أولًا

أضف هذا التحقق (assertion) إلى الاختبار الحالي

```go
//server_test.go
if response.Result().Header.Get("content-type") != "application/json" {
	t.Errorf("response did not have content-type of application/json, got %v", response.Result().Header)
}
```

## جرّب تشغيل الاختبار

```
=== RUN   TestLeague/it_returns_the_league_table_as_JSON
    --- FAIL: TestLeague/it_returns_the_league_table_as_JSON (0.00s)
        server_test.go:124: response did not have content-type of application/json, got map[Content-Type:[text/plain; charset=utf-8]]
```

## اكتب كودًا كافيًا لنجاح الاختبار

حدّث `leagueHandler`

```go
//server.go
func (p *PlayerServer) leagueHandler(w http.ResponseWriter, r *http.Request) {
	w.Header().Set("content-type", "application/json")
	json.NewEncoder(w).Encode(p.store.GetLeague())
}
```

من المفترض أن ينجح الاختبار.

## إعادة الهيكلة

أنشئ ثابتًا (constant) للنص "application/json" واستخدمه في `leagueHandler`

```go
//server.go
const jsonContentType = "application/json"

func (p *PlayerServer) leagueHandler(w http.ResponseWriter, r *http.Request) {
	w.Header().Set("content-type", jsonContentType)
	json.NewEncoder(w).Encode(p.store.GetLeague())
}
```

ثم أضف دالة مساعدة لـ `assertContentType`.

```go
//server_test.go
func assertContentType(t testing.TB, response *httptest.ResponseRecorder, want string) {
	t.Helper()
	if response.Result().Header.Get("content-type") != want {
		t.Errorf("response did not have content-type of %s, got %v", want, response.Result().Header)
	}
}
```

استخدمها في الاختبار.

```go
//server_test.go
assertContentType(t, response, jsonContentType)
```

وبعد أن انتهينا من `PlayerServer` في الوقت الحالي، يمكننا تحويل انتباهنا إلى `InMemoryPlayerStore`، لأنه لو جرّبنا الآن عرض هذا على مالك المنتج فلن تعمل `/league`.

وأسرع طريقة لنكسب بعض الثقة هي أن نضيف إلى اختبار التكامل (integration test) فحصًا نصل فيه إلى نقطة النهاية الجديدة ونتأكد من حصولنا على الاستجابة الصحيحة من `/league`.

## اكتب الاختبار أولًا

يمكننا استخدام `t.Run` لتقسيم هذا الاختبار قليلًا، ويمكننا إعادة استخدام دوال المساعدة من اختبارات خادمنا، وهذا يبيّن مرة أخرى أهمية إعادة هيكلة الاختبارات.

```go
//server_integration_test.go
func TestRecordingWinsAndRetrievingThem(t *testing.T) {
	store := NewInMemoryPlayerStore()
	server := NewPlayerServer(store)
	player := "Pepper"

	server.ServeHTTP(httptest.NewRecorder(), newPostWinRequest(player))
	server.ServeHTTP(httptest.NewRecorder(), newPostWinRequest(player))
	server.ServeHTTP(httptest.NewRecorder(), newPostWinRequest(player))

	t.Run("get score", func(t *testing.T) {
		response := httptest.NewRecorder()
		server.ServeHTTP(response, newGetScoreRequest(player))
		assertStatus(t, response.Code, http.StatusOK)

		assertResponseBody(t, response.Body.String(), "3")
	})

	t.Run("get league", func(t *testing.T) {
		response := httptest.NewRecorder()
		server.ServeHTTP(response, newLeagueRequest())
		assertStatus(t, response.Code, http.StatusOK)

		got := getLeagueFromResponse(t, response.Body)
		want := []Player{
			{"Pepper", 3},
		}
		assertLeague(t, got, want)
	})
}
```

## جرّب تشغيل الاختبار

```
=== RUN   TestRecordingWinsAndRetrievingThem/get_league
    --- FAIL: TestRecordingWinsAndRetrievingThem/get_league (0.00s)
        server_integration_test.go:35: got [] want [{Pepper 3}]
```

## اكتب كودًا كافيًا لنجاح الاختبار

تُرجع `InMemoryPlayerStore` القيمة `nil` عند استدعاء `GetLeague()`، لذا سنحتاج إلى إصلاح ذلك.

```go
//in_memory_player_store.go
func (i *InMemoryPlayerStore) GetLeague() []Player {
	var league []Player
	for name, wins := range i.store {
		league = append(league, Player{name, wins})
	}
	return league
}
```

وكل ما علينا فعله هو المرور على الـ map وتحويل كل مفتاح/قيمة إلى `Player`.

من المفترض أن ينجح الاختبار الآن.

## الخلاصة

واصلنا التقدّم الآمن في برنامجنا باستخدام التطوير الموجه بالاختبار (TDD)، وجعلناه يدعم نقاط نهاية جديدة بطريقة قابلة للصيانة عبر موجّه (router)، وأصبح الآن قادرًا على إرجاع JSON لمستهلكينا. وفي الفصل القادم سنغطي حفظ البيانات وترتيب جدولنا.

وإليك ما غطيناه:

- **التوجيه (Routing)**. توفر لك المكتبة القياسية نوعًا سهل الاستخدام للتوجيه. وهو يتبنى واجهة `http.Handler` بالكامل، حيث تُسند المسارات إلى معالجات `Handler`، والموجّه نفسه هو أيضًا `Handler`. لكنه لا يملك بعض الميزات التي قد تتوقعها مثل متغيرات المسار (path variables) (مثل `/users/{id}`). ويمكنك تحليل هذه المعلومة بنفسك بسهولة، لكن قد ترغب في النظر إلى مكتبات توجيه أخرى إذا صار ذلك عبئًا. وأغلب المكتبات الشائعة تلتزم بفلسفة المكتبة القياسية فتنفّذ `http.Handler` أيضًا.
- **دمج الأنواع (Type embedding)**. مررنا سريعًا على هذه التقنية، لكن يمكنك [معرفة المزيد عنها من Effective Go](https://golang.org/doc/effective_go.html#embedding). وإن كان عليك أن تخرج بشيء واحد منها فهو أنها قد تكون مفيدة للغاية، لكن _فكّر دائمًا في واجهتك العامة، ولا تكشف إلا ما هو مناسب_.
- **فك تسلسل JSON وتسلسله**. تجعل المكتبة القياسية تسلسل بياناتك وفك تسلسلها أمرًا بالغ السهولة. وهي أيضًا قابلة للتهيئة (configuration)، ويمكنك تخصيص كيفية عمل هذه التحويلات على البيانات عند الحاجة.
