---
title: الـ WebSockets
weight: 330
---

# الـ WebSockets

**[يمكنك العثور على كل كود هذا الفصل هنا](https://github.com/quii/learn-go-with-tests/tree/main/websockets)**

في هذا الفصل سنتعلّم كيف نستخدم WebSockets لتحسين تطبيقنا.

## مراجعة سريعة للمشروع

لدينا تطبيقان في قاعدة كود البوكر

* _تطبيق سطر الأوامر_. يطلب من المستخدم إدخال عدد اللاعبين في اللعبة. ومنذ تلك اللحظة يبلّغ اللاعبين بقيمة "الرهان الإجباري" (blind bet) التي تزداد بمرور الوقت. ويمكن للمستخدم في أي لحظة إدخال `"{Playername} wins"` لإنهاء اللعبة وتسجيل الفائز في مخزن (store).
* _تطبيق الويب_. يتيح للمستخدمين تسجيل الفائزين في الألعاب ويعرض جدول الترتيب. ويتشارك المخزن نفسه مع تطبيق سطر الأوامر.

## الخطوات التالية

مالكة المنتج (product owner) سعيدة جدًا بتطبيق سطر الأوامر، لكنها تفضّل لو استطعنا نقل تلك الوظائف إلى المتصفح. فهي تتخيّل صفحة ويب بها مربع نص يتيح للمستخدم إدخال عدد اللاعبين، وعندما يرسل النموذج (form) تعرض الصفحة قيمة الرهان الإجباري وتحدّثها تلقائيًا عند الحاجة. وكما في تطبيق سطر الأوامر، يمكن للمستخدم إعلان الفائز فيُحفظ في قاعدة البيانات.

في الظاهر يبدو الأمر بسيطًا، لكن كما نفعل دائمًا يجب أن نؤكد على اتباع منهج _تكراري_ في كتابة البرمجيات.

أولًا سنحتاج إلى تقديم HTML. حتى الآن كانت كل نقاط النهاية (endpoints) لدينا تُرجع نصًا عاديًا أو JSON. ويمكننا _أن_ نستخدم التقنيات نفسها التي نعرفها (فهي في النهاية نصوص في كل الأحوال)، لكن يمكننا أيضًا استخدام حزمة [html/template](https://golang.org/pkg/html/template/) للحصول على حل أكثر نظافة.

نحتاج أيضًا إلى القدرة على إرسال رسائل بشكل غير متزامن إلى المستخدم تقول `The blind is now *y*` دون الحاجة إلى تحديث المتصفح. ويمكننا استخدام [WebSockets](https://en.wikipedia.org/wiki/WebSocket) لتحقيق ذلك.

> الـ WebSocket بروتوكول اتصالات حاسوبية، يوفّر قنوات اتصال ثنائية الاتجاه (full-duplex) عبر اتصال TCP واحد

بما أننا نتبنّى عددًا من التقنيات، فمن الأهم أن ننجز أقل قدر ممكن من العمل المفيد أولًا ثم نكرّر.

ولهذا السبب فإن أول ما سنفعله هو إنشاء صفحة ويب بها نموذج يتيح للمستخدم تسجيل فائز. وبدلًا من استخدام نموذج عادي، سنستخدم WebSockets لإرسال تلك البيانات إلى خادمنا ليسجّلها.

وبعد ذلك سنعمل على تنبيهات الرهان الإجباري، وحينها سيكون لدينا قدر من البنية التحتية جاهزًا.

### ماذا عن اختبارات JavaScript؟

سيُكتب بعض الـ JavaScript للقيام بذلك، لكنني لن أخوض في كتابة اختبارات له.

هذا ممكن بالطبع، لكن حرصًا على الإيجاز لن أضمّن أي شرح له.

أعتذر يا أصدقاء. اضغطوا على O'Reilly ليدفعوا لي مقابل كتابة "Learn JavaScript with tests".

## اكتب الاختبار أولًا

أول ما نحتاج إلى فعله هو تقديم بعض HTML للمستخدمين عندما يزورون `/game`.

وهذا تذكير بالكود ذي الصلة في خادم الويب لدينا

```go
type PlayerServer struct {
	store PlayerStore
	http.Handler
}

const jsonContentType = "application/json"

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

أسهل ما يمكننا فعله الآن هو التحقق من أننا عند `GET /game` نحصل على `200`.

```go
func TestGame(t *testing.T) {
	t.Run("GET /game returns 200", func(t *testing.T) {
		server := NewPlayerServer(&StubPlayerStore{})

		request, _ := http.NewRequest(http.MethodGet, "/game", nil)
		response := httptest.NewRecorder()

		server.ServeHTTP(response, request)

		assertStatus(t, response.Code, http.StatusOK)
	})
}
```

## جرّب تشغيل الاختبار

```
--- FAIL: TestGame (0.00s)
=== RUN   TestGame/GET_/game_returns_200
    --- FAIL: TestGame/GET_/game_returns_200 (0.00s)
    	server_test.go:109: did not get correct status, got 404, want 200
```

## اكتب كودًا كافيًا لنجاح الاختبار

خادمنا يحتوي على موجّه (router) مُهيّأ، لذا إصلاح الأمر سهل نسبيًا.

أضف إلى الموجّه لدينا

```go
router.Handle("/game", http.HandlerFunc(p.game))
```

ثم اكتب الـ method `game`

```go
func (p *PlayerServer) game(w http.ResponseWriter, r *http.Request) {
	w.WriteHeader(http.StatusOK)
}
```

## إعادة الهيكلة

كود الخادم جيد أصلًا، فقد أدخلنا كودًا إضافيًا في الكود القائم المحسّن هيكلته بسهولة تامة.

يمكننا ترتيب الاختبار قليلًا بإضافة دالة مساعدة `newGameRequest` لإنشاء الطلب إلى `/game`. جرّب كتابتها بنفسك.

```go
func TestGame(t *testing.T) {
	t.Run("GET /game returns 200", func(t *testing.T) {
		server := NewPlayerServer(&StubPlayerStore{})

		request := newGameRequest()
		response := httptest.NewRecorder()

		server.ServeHTTP(response, request)

		assertStatus(t, response, http.StatusOK)
	})
}
```

ستلاحظ أيضًا أنني غيّرت `assertStatus` لتقبل `response` بدلًا من `response.Code` لأنني أرى أن قراءتها أفضل.

الآن نحتاج إلى جعل نقطة النهاية تُرجع بعض HTML، وهذا هو

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>Let's play poker</title>
</head>
<body>
<section id="game">
    <div id="declare-winner">
        <label for="winner">Winner</label>
        <input type="text" id="winner"/>
        <button id="winner-button">Declare winner</button>
    </div>
</section>
</body>
<script type="application/javascript">

    const submitWinnerButton = document.getElementById('winner-button')
    const winnerInput = document.getElementById('winner')

    if (window['WebSocket']) {
        const conn = new WebSocket('ws://' + document.location.host + '/ws')

        submitWinnerButton.onclick = event => {
            conn.send(winnerInput.value)
        }
    }
</script>
</html>
```

لدينا صفحة ويب بسيطة جدًا

* حقل نصي ليدخل المستخدم اسم الفائز فيه
* وزر يمكنه النقر عليه لإعلان الفائز.
* وبعض الـ JavaScript لفتح اتصال WebSocket بخادمنا والتعامل مع الضغط على زر الإرسال

الـ `WebSocket` مدمج في أغلب المتصفحات الحديثة، لذا لا داعي للقلق بشأن جلب أي مكتبات. لن تعمل صفحة الويب في المتصفحات القديمة، لكن لا مشكلة لدينا في ذلك في هذا السيناريو.

### كيف نختبر أننا نُرجِع التوصيف الصحيح؟

توجد عدة طرق. وكما أُكّد في الكتاب كله، من المهم أن تكون للاختبارات التي تكتبها قيمة كافية تبرّر التكلفة.

1. اكتب اختبارًا يعمل في المتصفح باستخدام شيء مثل Selenium. هذه الاختبارات هي الأكثر "واقعية" بين كل الأساليب، لأنها تشغّل متصفح ويب فعليًا وتحاكي تفاعل مستخدم معه. ويمكن لهذه الاختبارات أن تمنحك ثقة كبيرة بأن نظامك يعمل، لكن كتابتها أصعب من اختبارات الوحدة (unit tests) وتشغيلها أبطأ بكثير. وبالنسبة لغرض منتجنا، هذا مبالغ فيه.
2. أجرِ مطابقة نصية دقيقة. هذا _قد_ يكون مقبولًا، لكن هذا النوع من الاختبارات ينتهي به الأمر هشًا جدًا. فبمجرد أن يغيّر أحدهم التوصيف (markup) سيفشل اختبار لديك، مع أن شيئًا لم _ينكسر فعليًا_.
3. تحقق من أننا نستدعي القالب الصحيح. سنستخدم مكتبة قوالب من المكتبة القياسية لتقديم HTML (سنناقش ذلك بعد قليل)، ويمكننا حقن _الشيء_ الذي يولّد HTML والتجسس على استدعائه للتحقق من أننا نفعل ذلك بشكل صحيح. سيكون لهذا أثر على تصميم كودنا، لكنه لا يختبر كثيرًا في الحقيقة؛ سوى أننا نستدعيه بملف القالب الصحيح. وبما أنه لن يكون لدينا سوى قالب واحد في مشروعنا، فيبدو احتمال الفشل هنا ضعيفًا.

لذا، ولأول مرة في كتاب "Learn Go with Tests"، لن نكتب اختبارًا.

ضع التوصيف في ملف اسمه `game.html`

ثم غيّر نقطة النهاية التي كتبناها للتو إلى ما يلي

```go
func (p *PlayerServer) game(w http.ResponseWriter, r *http.Request) {
	tmpl, err := template.ParseFiles("game.html")

	if err != nil {
		http.Error(w, fmt.Sprintf("problem loading template %s", err.Error()), http.StatusInternalServerError)
		return
	}

	tmpl.Execute(w, nil)
}
```

حزمة [`html/template`](https://golang.org/pkg/html/template/) هي حزمة Go لإنشاء HTML. في حالتنا نستدعي `template.ParseFiles` ونمرّر مسار ملف html لدينا. وبافتراض عدم وجود خطأ يمكنك بعدها `Execute` القالب، فيكتبه إلى `io.Writer`. وفي حالتنا نريد أن يكتب `Write` إلى الإنترنت، لذا نمرّر له `http.ResponseWriter` الخاص بنا.

بما أننا لم نكتب اختبارًا، من الحكمة أن نختبر خادم الويب يدويًا للتأكد فقط من أن الأمور تعمل كما نأمل. اذهب إلى `cmd/webserver` وشغّل ملف `main.go`. ثم زُر `http://localhost:5000/game`.

ينبغي _أن_ تكون قد حصلت على خطأ بأن القالب غير موجود. يمكنك إما تغيير المسار ليكون نسبيًا إلى مجلدك، أو وضع نسخة من `game.html` في مجلد `cmd/webserver`. وقد اخترت إنشاء رابط رمزي (symlink) بالصياغة (`ln -s ../../game.html game.html`) إلى الملف داخل جذر المشروع، حتى تظهر أي تغييرات أجريها عند تشغيل الخادم.

إذا أجريت هذا التغيير وشغّلت مرة أخرى، ينبغي أن ترى واجهة المستخدم (UI) لدينا.

الآن نحتاج إلى اختبار أنه عندما تصلنا سلسلة نصية عبر اتصال WebSocket بخادمنا فإننا نعلنها فائزًا في لعبة.

## اكتب الاختبار أولًا

لأول مرة سنستخدم مكتبة خارجية حتى نستطيع التعامل مع WebSockets.

شغّل `go get github.com/gorilla/websocket`

سيجلب هذا كود مكتبة [Gorilla WebSocket](https://github.com/gorilla/websocket) الرائعة. والآن يمكننا تحديث اختباراتنا لمتطلبنا الجديد.

```go
t.Run("when we get a message over a websocket it is a winner of a game", func(t *testing.T) {
	store := &StubPlayerStore{}
	winner := "Ruth"
	server := httptest.NewServer(NewPlayerServer(store))
	defer server.Close()

	wsURL := "ws" + strings.TrimPrefix(server.URL, "http") + "/ws"

	ws, _, err := websocket.DefaultDialer.Dial(wsURL, nil)
	if err != nil {
		t.Fatalf("could not open a ws connection on %s %v", wsURL, err)
	}
	defer ws.Close()

	if err := ws.WriteMessage(websocket.TextMessage, []byte(winner)); err != nil {
		t.Fatalf("could not send message over ws connection %v", err)
	}

	AssertPlayerWin(t, store, winner)
})
```

تأكد من وجود import لمكتبة `websocket`. لقد فعلتها بيئة التطوير (IDE) لدي تلقائيًا، وينبغي أن تفعلها بيئتك أيضًا.

لاختبار ما يحدث من المتصفح، علينا فتح اتصال WebSocket خاص بنا والكتابة إليه.

كانت اختباراتنا السابقة حول خادمنا تستدعي methods على الخادم فقط، لكننا الآن نحتاج إلى اتصال دائم بخادمنا. ولعمل ذلك نستخدم `httptest.NewServer` الذي يأخذ `http.Handler` ويشغّله ويستمع للاتصالات.

باستخدام `websocket.DefaultDialer.Dial` نحاول الاتصال بخادمنا، ثم نحاول إرسال رسالة تحتوي على `winner`.

وأخيرًا نتحقق (assert) من مخزن اللاعبين للتأكد من تسجيل الفائز.

## جرّب تشغيل الاختبار

```
=== RUN   TestGame/when_we_get_a_message_over_a_websocket_it_is_a_winner_of_a_game
    --- FAIL: TestGame/when_we_get_a_message_over_a_websocket_it_is_a_winner_of_a_game (0.00s)
        server_test.go:124: could not open a ws connection on ws://127.0.0.1:55838/ws websocket: bad handshake
```

لم نغيّر خادمنا ليقبل اتصالات WebSocket على `/ws`، لذا لم تكتمل المصافحة (handshake) بعد.

## اكتب كودًا كافيًا لنجاح الاختبار

أضف مسارًا آخر إلى الموجّه لدينا

```go
router.Handle("/ws", http.HandlerFunc(p.webSocket))
```

ثم أضف المعالج `webSocket` الجديد

```go
func (p *PlayerServer) webSocket(w http.ResponseWriter, r *http.Request) {
	upgrader := websocket.Upgrader{
		ReadBufferSize:  1024,
		WriteBufferSize: 1024,
	}
	upgrader.Upgrade(w, r, nil)
}
```

لقبول اتصال WebSocket نقوم بـ `Upgrade` للطلب. وإذا أعدت تشغيل الاختبار الآن، ينبغي أن تنتقل إلى الخطأ التالي.

```
=== RUN   TestGame/when_we_get_a_message_over_a_websocket_it_is_a_winner_of_a_game
    --- FAIL: TestGame/when_we_get_a_message_over_a_websocket_it_is_a_winner_of_a_game (0.00s)
        server_test.go:132: got 0 calls to RecordWin want 1
```

وبعد أن أصبح لدينا اتصال مفتوح، سنريد الاستماع إلى رسالة ثم تسجيلها فائزًا.

```go
func (p *PlayerServer) webSocket(w http.ResponseWriter, r *http.Request) {
	upgrader := websocket.Upgrader{
		ReadBufferSize:  1024,
		WriteBufferSize: 1024,
	}
	conn, _ := upgrader.Upgrade(w, r, nil)
	_, winnerMsg, _ := conn.ReadMessage()
	p.store.RecordWin(string(winnerMsg))
}
```

(نعم، نحن نتجاهل الكثير من الأخطاء الآن!)

تنتظر `conn.ReadMessage()` حتى تصل رسالة على الاتصال. وبمجرد وصول واحدة نستخدمها في `RecordWin`. وهذا سيغلق اتصال WebSocket في النهاية.

وإذا حاولت تشغيل الاختبار، فسيظل فاشلًا.

المشكلة في التوقيت. فهناك تأخير بين قراءة اتصال WebSocket للرسالة وتسجيل الفوز، وينتهي اختبارنا قبل حدوث ذلك. يمكنك اختبار هذا بوضع `time.Sleep` قصير قبل التحقق الأخير.

لنمضِ في ذلك الآن، مع الإقرار بأن وضع فترات انتظار عشوائية في الاختبارات **ممارسة سيئة جدًا**.

```go
time.Sleep(10 * time.Millisecond)
AssertPlayerWin(t, store, winner)
```

## إعادة الهيكلة

اقترفنا ذنوبًا كثيرة لجعل هذا الاختبار يعمل في كود الخادم وكود الاختبار معًا، لكن تذكّر أن هذه أسهل طريقة للعمل.

لدينا برمجية بشعة ومقرفة لكنها _تعمل_ ومدعومة باختبار، لذا أصبحنا أحرارًا في تجميلها ونحن نعلم أننا لن نكسر شيئًا عن غير قصد.

لنبدأ بكود الخادم.

يمكننا نقل `upgrader` إلى قيمة خاصة داخل حزمتنا، لأننا لا نحتاج إلى إعادة تعريفه مع كل طلب اتصال WebSocket

```go
var wsUpgrader = websocket.Upgrader{
	ReadBufferSize:  1024,
	WriteBufferSize: 1024,
}

func (p *PlayerServer) webSocket(w http.ResponseWriter, r *http.Request) {
	conn, _ := wsUpgrader.Upgrade(w, r, nil)
	_, winnerMsg, _ := conn.ReadMessage()
	p.store.RecordWin(string(winnerMsg))
}
```

سيُنفَّذ استدعاؤنا لـ `template.ParseFiles("game.html")` مع كل `GET /game`، ما يعني أننا سنصل إلى نظام الملفات في كل طلب رغم أننا لا نحتاج إلى إعادة تحليل القالب. لنعد هيكلة كودنا بحيث نحلّل القالب مرة واحدة في `NewPlayerServer` بدلًا من ذلك. وسنحتاج إلى جعل هذه الدالة قادرة على إرجاع خطأ في حال واجهنا مشكلات في جلب القالب من القرص أو تحليله.

وهذه هي التغييرات ذات الصلة على `PlayerServer`

```go
type PlayerServer struct {
	store PlayerStore
	http.Handler
	template *template.Template
}

const htmlTemplatePath = "game.html"

func NewPlayerServer(store PlayerStore) (*PlayerServer, error) {
	p := new(PlayerServer)

	tmpl, err := template.ParseFiles(htmlTemplatePath)

	if err != nil {
		return nil, fmt.Errorf("problem opening %s %v", htmlTemplatePath, err)
	}

	p.template = tmpl
	p.store = store

	router := http.NewServeMux()
	router.Handle("/league", http.HandlerFunc(p.leagueHandler))
	router.Handle("/players/", http.HandlerFunc(p.playersHandler))
	router.Handle("/game", http.HandlerFunc(p.game))
	router.Handle("/ws", http.HandlerFunc(p.webSocket))

	p.Handler = router

	return p, nil
}

func (p *PlayerServer) game(w http.ResponseWriter, r *http.Request) {
	p.template.Execute(w, nil)
}
```

بتغيير توقيع `NewPlayerServer` أصبح لدينا الآن مشكلات في الترجمة. جرّب إصلاحها بنفسك أو ارجع إلى الكود المصدري إن واجهت صعوبة.

وفي كود الاختبار أنشأت دالة مساعدة اسمها `mustMakePlayerServer(t *testing.T, store PlayerStore) *PlayerServer` حتى أُخفي ضجيج الأخطاء بعيدًا عن الاختبارات.

```go
func mustMakePlayerServer(t *testing.T, store PlayerStore) *PlayerServer {
	server, err := NewPlayerServer(store)
	if err != nil {
		t.Fatal("problem creating player server", err)
	}
	return server
}
```

وبالمثل أنشأت دالة مساعدة أخرى هي `mustDialWS` حتى أُخفي ضجيج الأخطاء المزعج عند إنشاء اتصال WebSocket.

```go
func mustDialWS(t *testing.T, url string) *websocket.Conn {
	ws, _, err := websocket.DefaultDialer.Dial(url, nil)

	if err != nil {
		t.Fatalf("could not open a ws connection on %s %v", url, err)
	}

	return ws
}
```

وأخيرًا، يمكننا في كود الاختبار إنشاء دالة مساعدة لترتيب إرسال الرسائل

```go
func writeWSMessage(t testing.TB, conn *websocket.Conn, message string) {
	t.Helper()
	if err := conn.WriteMessage(websocket.TextMessage, []byte(message)); err != nil {
		t.Fatalf("could not send message over ws connection %v", err)
	}
}
```

وبعد أن أصبحت الاختبارات تنجح، جرّب تشغيل الخادم وأعلن بعض الفائزين في `/game`. ينبغي أن تراهم مسجّلين في `/league`. وتذكّر أننا في كل مرة نستقبل فيها فائزًا _نُغلق الاتصال_، لذا ستحتاج إلى تحديث الصفحة لفتح الاتصال من جديد.

لقد أنشأنا نموذج ويب بسيطًا يتيح للمستخدمين تسجيل فائز في لعبة. لنكرّر التحسين عليه بحيث يستطيع المستخدم بدء لعبة بتقديم عدد اللاعبين، ويدفع الخادم رسائل إلى العميل تخبره بقيمة الرهان الإجباري مع مرور الوقت.

أولًا حدّث `game.html` لتحديث كود جهة العميل (client side) وفق المتطلبات الجديدة

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>Lets play poker</title>
</head>
<body>
<section id="game">
    <div id="game-start">
        <label for="player-count">Number of players</label>
        <input type="number" id="player-count"/>
        <button id="start-game">Start</button>
    </div>

    <div id="declare-winner">
        <label for="winner">Winner</label>
        <input type="text" id="winner"/>
        <button id="winner-button">Declare winner</button>
    </div>

    <div id="blind-value"/>
</section>

<section id="game-end">
    <h1>Another great game of poker everyone!</h1>
    <p><a href="/league">Go check the league table</a></p>
</section>

</body>
<script type="application/javascript">
    const startGame = document.getElementById('game-start')

    const declareWinner = document.getElementById('declare-winner')
    const submitWinnerButton = document.getElementById('winner-button')
    const winnerInput = document.getElementById('winner')

    const blindContainer = document.getElementById('blind-value')

    const gameContainer = document.getElementById('game')
    const gameEndContainer = document.getElementById('game-end')

    declareWinner.hidden = true
    gameEndContainer.hidden = true

    document.getElementById('start-game').addEventListener('click', event => {
        startGame.hidden = true
        declareWinner.hidden = false

        const numberOfPlayers = document.getElementById('player-count').value

        if (window['WebSocket']) {
            const conn = new WebSocket('ws://' + document.location.host + '/ws')

            submitWinnerButton.onclick = event => {
                conn.send(winnerInput.value)
                gameEndContainer.hidden = false
                gameContainer.hidden = true
            }

            conn.onclose = evt => {
                blindContainer.innerText = 'Connection closed'
            }

            conn.onmessage = evt => {
                blindContainer.innerText = evt.data
            }

            conn.onopen = function () {
                conn.send(numberOfPlayers)
            }
        }
    })
</script>
</html>
```

التغيير الرئيسي هو إضافة قسم لإدخال عدد اللاعبين وقسم لعرض قيمة الرهان الإجباري. ولدينا منطق صغير لإظهار/إخفاء واجهة المستخدم حسب مرحلة اللعبة.

أي رسالة تصلنا عبر `conn.onmessage` نفترض أنها تنبيهات الرهان الإجباري، لذا نضبط `blindContainer.innerText` وفقًا لها.

كيف نرسل تنبيهات الرهان الإجباري؟ في الفصل السابق قدّمنا فكرة `Game` حتى يستطيع كود الـ CLI استدعاء `Game` ويتولّى كل ما يلزم، بما في ذلك جدولة تنبيهات الرهان الإجباري. وقد اتضح أن هذا فصل جيد للاهتمامات.

```go
type Game interface {
	Start(numberOfPlayers int)
	Finish(winner string)
}
```

عندما كان يُطلب من المستخدم في الـ CLI إدخال عدد اللاعبين، كان يبدأ `Start` اللعبة فتنطلق تنبيهات الرهان الإجباري، وعندما يعلن المستخدم الفائز كان يُنهيها `Finish`. وهذه هي المتطلبات نفسها التي لدينا الآن، لكن بوسيلة مختلفة للحصول على المدخلات؛ لذا ينبغي أن نسعى إلى إعادة استخدام هذا المفهوم إن استطعنا.

تنفيذنا "الحقيقي" لـ `Game` هو `TexasHoldem`

```go
type TexasHoldem struct {
	alerter BlindAlerter
	store   PlayerStore
}
```

بتمرير `BlindAlerter` يستطيع `TexasHoldem` جدولة إرسال تنبيهات الرهان الإجباري إلى _أي مكان_

```go
type BlindAlerter interface {
	ScheduleAlertAt(duration time.Duration, amount int)
}
```

وللتذكير، هذا هو تنفيذنا لـ `BlindAlerter` الذي نستخدمه في الـ CLI.

```go
func StdOutAlerter(duration time.Duration, amount int) {
	time.AfterFunc(duration, func() {
		fmt.Fprintf(os.Stdout, "Blind is now %d\n", amount)
	})
}
```

هذا يعمل في الـ CLI لأننا _نريد دائمًا إرسال التنبيهات إلى `os.Stdout`_، لكنه لن يعمل في خادم الويب لدينا. فمع كل طلب نحصل على `http.ResponseWriter` جديد نرقّيه بعدها إلى `*websocket.Conn`. لذا لا يمكننا أن نعرف عند بناء اعتمادياتنا إلى أين ينبغي أن تذهب تنبيهاتنا.

ولهذا السبب نحتاج إلى تغيير `BlindAlerter.ScheduleAlertAt` ليأخذ وجهة (destination) للتنبيهات، حتى نتمكن من إعادة استخدامه في خادم الويب.

افتح `blind_alerter.go` وأضف الوسيط من النوع `io.Writer`

```go
type BlindAlerter interface {
	ScheduleAlertAt(duration time.Duration, amount int, to io.Writer)
}

type BlindAlerterFunc func(duration time.Duration, amount int, to io.Writer)

func (a BlindAlerterFunc) ScheduleAlertAt(duration time.Duration, amount int, to io.Writer) {
	a(duration, amount, to)
}
```

لم تعد فكرة `StdoutAlerter` تناسب نموذجنا الجديد، لذا أعد تسميتها إلى `Alerter` فقط

```go
func Alerter(duration time.Duration, amount int, to io.Writer) {
	time.AfterFunc(duration, func() {
		fmt.Fprintf(to, "Blind is now %d\n", amount)
	})
}
```

إذا حاولت الترجمة ستفشل في `TexasHoldem` لأنه يستدعي `ScheduleAlertAt` بلا وجهة. ولتعود الأمور إلى الترجمة _مؤقتًا_، ثبّتها بشكل صريح (hard-code) على `os.Stdout`.

جرّب تشغيل الاختبارات وستفشل لأن `SpyBlindAlerter` لم يعد يحقّق `BlindAlerter`. أصلح ذلك بتحديث توقيع `ScheduleAlertAt`، ثم شغّل الاختبارات وينبغي أن تبقى خضراء.

لا معنى لأن يعرف `TexasHoldem` إلى أين يرسل تنبيهات الرهان الإجباري. لنحدّث الآن `Game` بحيث تحدّد عند بدء اللعبة _إلى أين_ ينبغي أن تذهب التنبيهات.

```go
type Game interface {
	Start(numberOfPlayers int, alertsDestination io.Writer)
	Finish(winner string)
}
```

دع المترجم يخبرك بما تحتاج إلى إصلاحه. التغيير ليس بهذا السوء:

* حدّث `TexasHoldem` ليحقّق `Game` تحقيقًا صحيحًا
* وفي `CLI`، عند بدء اللعبة مرّر خاصية `out` لدينا (`cli.game.Start(numberOfPlayers, cli.out)`)
* وفي اختبار `TexasHoldem` استخدمت `game.Start(5, io.Discard)` لحل مشكلة الترجمة وضبط مخرج التنبيهات ليُهمَل

إذا فعلت كل شيء صحيحًا، فكل شيء ينبغي أن يكون أخضر! والآن يمكننا محاولة استخدام `Game` داخل `Server`.

## اكتب الاختبار أولًا

متطلبات `CLI` و`Server` واحدة! لكن آلية التوصيل مختلفة فقط.

لنلقِ نظرة على اختبار `CLI` لدينا لنستلهم منه.

```go
t.Run("start game with 3 players and finish game with 'Chris' as winner", func(t *testing.T) {
	game := &GameSpy{}

	out := &bytes.Buffer{}
	in := userSends("3", "Chris wins")

	poker.NewCLI(in, out, game).PlayPoker()

	assertMessagesSentToUser(t, out, poker.PlayerPrompt)
	assertGameStartedWith(t, game, 3)
	assertFinishCalledWith(t, game, "Chris")
})
```

يبدو أننا ينبغي أن نستطيع تطوير نتيجة مماثلة بالاختبارات باستخدام `GameSpy`

استبدل اختبار الـ websocket القديم بما يلي

```go
t.Run("start a game with 3 players and declare Ruth the winner", func(t *testing.T) {
	game := &poker.GameSpy{}
	winner := "Ruth"
	server := httptest.NewServer(mustMakePlayerServer(t, dummyPlayerStore, game))
	ws := mustDialWS(t, "ws"+strings.TrimPrefix(server.URL, "http")+"/ws")

	defer server.Close()
	defer ws.Close()

	writeWSMessage(t, ws, "3")
	writeWSMessage(t, ws, winner)

	time.Sleep(10 * time.Millisecond)
	assertGameStartedWith(t, game, 3)
	assertFinishCalledWith(t, game, winner)
})
```

* كما ناقشنا، ننشئ spy من النوع `Game` ونمرّره إلى `mustMakePlayerServer` (تأكد من تحديث الدالة المساعدة لدعم ذلك).
* ثم نرسل رسائل websocket الخاصة بلعبة.
* وأخيرًا نتحقق من أن اللعبة بدأت وانتهت بما نتوقعه.

## جرّب تشغيل الاختبار

ستظهر لك عدة أخطاء في الترجمة حول `mustMakePlayerServer` في اختبارات أخرى. عرّف متغيرًا غير مُصدَّر (unexported) اسمه `dummyGame` واستخدمه في كل الاختبارات التي لا تُترجم

```go
var (
	dummyGame = &GameSpy{}
)
```

الخطأ الأخير هو عندما نحاول تمرير `Game` إلى `NewPlayerServer` لكنه لا يدعمه بعد

```
./server_test.go:21:38: too many arguments in call to "github.com/quii/learn-go-with-tests/WebSockets/v2".NewPlayerServer
	have ("github.com/quii/learn-go-with-tests/WebSockets/v2".PlayerStore, "github.com/quii/learn-go-with-tests/WebSockets/v2".Game)
	want ("github.com/quii/learn-go-with-tests/WebSockets/v2".PlayerStore)
```

## اكتب أصغر قدر من الكود ليعمل الاختبار وافحص مخرجاته الفاشلة

أضفه كوسيط الآن فقط لتشغيل الاختبار

```go
func NewPlayerServer(store PlayerStore, game Game) (*PlayerServer, error)
```

أخيرًا!

```
=== RUN   TestGame/start_a_game_with_3_players_and_declare_Ruth_the_winner
--- FAIL: TestGame (0.01s)
    --- FAIL: TestGame/start_a_game_with_3_players_and_declare_Ruth_the_winner (0.01s)
    	server_test.go:146: wanted Start called with 3 but got 0
    	server_test.go:147: expected finish called with 'Ruth' but got ''
FAIL
```

## اكتب كودًا كافيًا لنجاح الاختبار

نحتاج إلى إضافة `Game` كحقل (field) في `PlayerServer` حتى يستخدمه عندما تصله الطلبات.

```go
type PlayerServer struct {
	store PlayerStore
	http.Handler
	template *template.Template
	game     Game
}
```

(لدينا بالفعل method اسمها `game`، لذا أعد تسميتها إلى `playGame`)

ثم لنُسنده في الدالة البانية (constructor) لدينا

```go
func NewPlayerServer(store PlayerStore, game Game) (*PlayerServer, error) {
	p := new(PlayerServer)

	tmpl, err := template.ParseFiles(htmlTemplatePath)

	if err != nil {
		return nil, fmt.Errorf("problem opening %s %v", htmlTemplatePath, err)
	}

	p.game = game

	// etc
}
```

الآن يمكننا استخدام `Game` لدينا داخل `webSocket`.

```go
func (p *PlayerServer) webSocket(w http.ResponseWriter, r *http.Request) {
	conn, _ := wsUpgrader.Upgrade(w, r, nil)

	_, numberOfPlayersMsg, _ := conn.ReadMessage()
	numberOfPlayers, _ := strconv.Atoi(string(numberOfPlayersMsg))
	p.game.Start(numberOfPlayers, io.Discard) //todo: Don't discard the blinds messages!

	_, winner, _ := conn.ReadMessage()
	p.game.Finish(string(winner))
}
```

مرحى! الاختبارات تنجح.

لن نرسل رسائل الرهان الإجباري إلى أي مكان _في الوقت الحالي_ لأننا نحتاج إلى التفكير في ذلك. فعندما نستدعي `game.Start` نمرّر `io.Discard` الذي سيتجاهل أي رسائل تُكتب إليه فحسب.

شغّل خادم الويب الآن. ستحتاج إلى تحديث `main.go` لتمرير `Game` إلى `PlayerServer`

```go
func main() {
	db, err := os.OpenFile(dbFileName, os.O_RDWR|os.O_CREATE, 0666)

	if err != nil {
		log.Fatalf("problem opening %s %v", dbFileName, err)
	}

	store, err := poker.NewFileSystemPlayerStore(db)

	if err != nil {
		log.Fatalf("problem creating file system player store, %v ", err)
	}

	game := poker.NewTexasHoldem(poker.BlindAlerterFunc(poker.Alerter), store)

	server, err := poker.NewPlayerServer(store, game)

	if err != nil {
		log.Fatalf("problem creating player server %v", err)
	}

	log.Fatal(http.ListenAndServe(":5000", server))
}
```

بغض النظر عن أننا لا نحصل على تنبيهات الرهان الإجباري بعد، فالتطبيق يعمل فعلًا! لقد نجحنا في إعادة استخدام `Game` مع `PlayerServer`، وقد تولّى كل التفاصيل. وبمجرد أن نكتشف كيف نرسل تنبيهات الرهان الإجباري عبر الـ websockets بدلًا من إهمالها، _ينبغي_ أن يعمل كل شيء.

لكن قبل ذلك، لنرتب بعض الكود.

## إعادة الهيكلة

طريقتنا في استخدام WebSockets بدائية نوعًا ما، ومعالجة الأخطاء ساذجة نوعًا ما، لذا أردت تغليف ذلك في نوع لإزالة هذه الفوضى من كود الخادم. وقد نرغب في إعادة النظر فيه لاحقًا، لكنه سيرتّب الأمور قليلًا الآن.

```go
type playerServerWS struct {
	*websocket.Conn
}

func newPlayerServerWS(w http.ResponseWriter, r *http.Request) *playerServerWS {
	conn, err := wsUpgrader.Upgrade(w, r, nil)

	if err != nil {
		log.Printf("problem upgrading connection to WebSockets %v\n", err)
	}

	return &playerServerWS{conn}
}

func (w *playerServerWS) WaitForMsg() string {
	_, msg, err := w.ReadMessage()
	if err != nil {
		log.Printf("error reading from websocket %v\n", err)
	}
	return string(msg)
}
```

الآن أصبح كود الخادم أبسط قليلًا

```go
func (p *PlayerServer) webSocket(w http.ResponseWriter, r *http.Request) {
	ws := newPlayerServerWS(w, r)

	numberOfPlayersMsg := ws.WaitForMsg()
	numberOfPlayers, _ := strconv.Atoi(numberOfPlayersMsg)
	p.game.Start(numberOfPlayers, io.Discard) //todo: Don't discard the blinds messages!

	winner := ws.WaitForMsg()
	p.game.Finish(winner)
}
```

وبمجرد أن نكتشف كيف لا نُهمل رسائل الرهان الإجباري، نكون قد انتهينا.

### دعنا _لا_ نكتب اختبارًا!

أحيانًا عندما لا نكون متأكدين من كيفية فعل شيء، يكون الأفضل أن نعبث ونجرّب الأشياء! تأكد من تسجيل عملك (commit) أولًا، لأننا بمجرد أن نكتشف طريقة ينبغي أن نقودها عبر اختبار.

سطر الكود المشكِل لدينا هو

```go
p.game.Start(numberOfPlayers, io.Discard) //todo: Don't discard the blinds messages!
```

نحتاج إلى تمرير `io.Writer` لتكتب اللعبة تنبيهات الرهان الإجباري إليه.

ألن يكون جميلًا لو استطعنا تمرير `playerServerWS` من قبل؟ فهو غلافنا حول WebSocket، لذا _يبدو_ أنه ينبغي أن نستطيع إرساله إلى `Game` لترسل الرسائل إليه.

جرّب ذلك:

```go
func (p *PlayerServer) webSocket(w http.ResponseWriter, r *http.Request) {
	ws := newPlayerServerWS(w, r)

	numberOfPlayersMsg := ws.WaitForMsg()
	numberOfPlayers, _ := strconv.Atoi(numberOfPlayersMsg)
	p.game.Start(numberOfPlayers, ws)
	//etc...
}
```

المترجم يشتكي

```
./server.go:71:14: cannot use ws (type *playerServerWS) as type io.Writer in argument to p.game.Start:
	*playerServerWS does not implement io.Writer (missing Write method)
```

يبدو أن الشيء البديهي هو أن نجعل `playerServerWS` _يحقّق_ `io.Writer`. ولعمل ذلك نستخدم `*websocket.Conn` الأساسي لاستخدام `WriteMessage` لإرسال الرسالة عبر الـ websocket

```go
func (w *playerServerWS) Write(p []byte) (n int, err error) {
	err = w.WriteMessage(websocket.TextMessage, p)

	if err != nil {
		return 0, err
	}

	return len(p), nil
}
```

يبدو هذا سهلًا جدًا! جرّب تشغيل التطبيق وانظر إن كان يعمل.

وقبل ذلك عدّل `TexasHoldem` ليكون زمن زيادة الرهان الإجباري أقصر حتى تراه يعمل

```go
blindIncrement := time.Duration(5+numberOfPlayers) * time.Second // (rather than a minute)
```

ينبغي أن تراه يعمل! فتزداد قيمة الرهان الإجباري في المتصفح وكأنها سحر.

لنعد الآن الكود كما كان ونفكّر كيف نختبره. كل ما فعلناه _لتنفيذ_ الأمر هو تمرير `playerServerWS` بدلًا من `io.Discard` إلى `StartGame`، وقد يجعلك ذلك تظن أنه ينبغي أن نتجسس على الاستدعاء للتحقق من أنه يعمل.

التجسس رائع ويساعدنا على فحص تفاصيل التنفيذ، لكن ينبغي أن نحاول دائمًا تفضيل اختبار السلوك _الحقيقي_ إن استطعنا، لأنك عند إعادة الهيكلة ستجد غالبًا أن اختبارات التجسس هي التي تبدأ بالفشل، لأنها تفحص عادةً تفاصيل تنفيذ تحاول أنت تغييرها.

يفتح اختبارنا حاليًا اتصال websocket بخادمنا قيد التشغيل ويرسل رسائل ليجعله يفعل أشياء. وبالمثل ينبغي أن نستطيع اختبار الرسائل التي يرسلها خادمنا عائدًا عبر اتصال websocket.

## اكتب الاختبار أولًا

سنعدّل اختبارنا الحالي.

حاليًا لا يرسل `GameSpy` أي بيانات إلى `out` عند استدعاء `Start`. ينبغي أن نغيّره حتى نستطيع ضبطه ليرسل رسالة جاهزة (canned message)، ثم نتحقق من وصول تلك الرسالة إلى الـ websocket. وهذا ينبغي أن يمنحنا ثقة بأننا ضبطنا الأمور بشكل صحيح مع ممارسة السلوك الحقيقي الذي نريده.

```go
type GameSpy struct {
	StartCalled     bool
	StartCalledWith int
	BlindAlert      []byte

	FinishedCalled   bool
	FinishCalledWith string
}
```

أضف الحقل `BlindAlert`.

حدّث `Start` في `GameSpy` ليرسل الرسالة الجاهزة إلى `out`.

```go
func (g *GameSpy) Start(numberOfPlayers int, out io.Writer) {
	g.StartCalled = true
	g.StartCalledWith = numberOfPlayers
	out.Write(g.BlindAlert)
}
```

يعني هذا الآن أننا عندما نمارس `PlayerServer` وهو يحاول `Start` اللعبة، ينبغي أن ينتهي به الأمر بإرسال الرسائل عبر الـ websocket إن كانت الأمور تعمل بشكل صحيح.

وأخيرًا يمكننا تحديث الاختبار

```go
t.Run("start a game with 3 players, send some blind alerts down WS and declare Ruth the winner", func(t *testing.T) {
	wantedBlindAlert := "Blind is 100"
	winner := "Ruth"

	game := &GameSpy{BlindAlert: []byte(wantedBlindAlert)}
	server := httptest.NewServer(mustMakePlayerServer(t, dummyPlayerStore, game))
	ws := mustDialWS(t, "ws"+strings.TrimPrefix(server.URL, "http")+"/ws")

	defer server.Close()
	defer ws.Close()

	writeWSMessage(t, ws, "3")
	writeWSMessage(t, ws, winner)

	time.Sleep(10 * time.Millisecond)
	assertGameStartedWith(t, game, 3)
	assertFinishCalledWith(t, game, winner)

	_, gotBlindAlert, _ := ws.ReadMessage()

	if string(gotBlindAlert) != wantedBlindAlert {
		t.Errorf("got blind alert %q, want %q", string(gotBlindAlert), wantedBlindAlert)
	}
})
```

* أضفنا `wantedBlindAlert` وضبطنا `GameSpy` ليرسلها إلى `out` إذا استُدعي `Start`.
* ونأمل أن تُرسَل عبر اتصال websocket، لذا أضفنا استدعاءً لـ `ws.ReadMessage()` للانتظار حتى تُرسَل رسالة ثم التحقق من أنها الرسالة التي توقعناها.

## جرّب تشغيل الاختبار

ستجد أن الاختبار يتعلّق إلى الأبد. والسبب أن `ws.ReadMessage()` ستحجب التنفيذ حتى تصل رسالة، ولن تصل أبدًا.

## اكتب أصغر قدر من الكود ليعمل الاختبار وافحص مخرجاته الفاشلة

ينبغي ألا تكون لدينا أبدًا اختبارات تعلّق، لذا لنقدّم طريقة للتعامل مع الكود الذي نريد أن تنتهي مهلته (timeout).

```go
func within(t testing.TB, d time.Duration, assert func()) {
	t.Helper()

	done := make(chan struct{}, 1)

	go func() {
		assert()
		done <- struct{}{}
	}()

	select {
	case <-time.After(d):
		t.Error("timed out")
	case <-done:
	}
}
```

ما تفعله `within` هو أنها تأخذ دالة `assert` كوسيط ثم تشغّلها في goroutine. وإذا انتهت الدالة (أو عندما تنتهي) ستشير إلى انتهائها عبر قناة `done`.

وفي أثناء ذلك نستخدم جملة `select` التي تتيح لنا انتظار إرسال رسالة عبر قناة. ومن هنا يصبح الأمر سباقًا بين دالة `assert` و`time.After` التي سترسل إشارة عند انقضاء المدة.

وأخيرًا أنشأت دالة مساعدة للتحقق لدينا لجعل الأمور أكثر أناقة قليلًا

```go
func assertWebsocketGotMsg(t *testing.T, ws *websocket.Conn, want string) {
	_, msg, _ := ws.ReadMessage()
	if string(msg) != want {
		t.Errorf(`got "%s", want "%s"`, string(msg), want)
	}
}
```

وهكذا أصبح شكل الاختبار الآن

```go
t.Run("start a game with 3 players, send some blind alerts down WS and declare Ruth the winner", func(t *testing.T) {
	wantedBlindAlert := "Blind is 100"
	winner := "Ruth"

	game := &GameSpy{BlindAlert: []byte(wantedBlindAlert)}
	server := httptest.NewServer(mustMakePlayerServer(t, dummyPlayerStore, game))
	ws := mustDialWS(t, "ws"+strings.TrimPrefix(server.URL, "http")+"/ws")

	defer server.Close()
	defer ws.Close()

	writeWSMessage(t, ws, "3")
	writeWSMessage(t, ws, winner)

	time.Sleep(tenMS)

	assertGameStartedWith(t, game, 3)
	assertFinishCalledWith(t, game, winner)
	within(t, tenMS, func() { assertWebsocketGotMsg(t, ws, wantedBlindAlert) })
})
```

والآن إذا شغّلت الاختبار...

```
=== RUN   TestGame
=== RUN   TestGame/start_a_game_with_3_players,_send_some_blind_alerts_down_WS_and_declare_Ruth_the_winner
--- FAIL: TestGame (0.02s)
    --- FAIL: TestGame/start_a_game_with_3_players,_send_some_blind_alerts_down_WS_and_declare_Ruth_the_winner (0.02s)
    	server_test.go:143: timed out
    	server_test.go:150: got "", want "Blind is 100"
```

## اكتب كودًا كافيًا لنجاح الاختبار

وأخيرًا يمكننا الآن تغيير كود الخادم بحيث يرسل اتصال WebSocket لدينا إلى اللعبة عند بدئها

```go
func (p *PlayerServer) webSocket(w http.ResponseWriter, r *http.Request) {
	ws := newPlayerServerWS(w, r)

	numberOfPlayersMsg := ws.WaitForMsg()
	numberOfPlayers, _ := strconv.Atoi(numberOfPlayersMsg)
	p.game.Start(numberOfPlayers, ws)

	winner := ws.WaitForMsg()
	p.game.Finish(winner)
}
```

## إعادة الهيكلة

كان كود الخادم تغييرًا صغيرًا جدًا، لذا لا يوجد الكثير لنغيّره هنا، لكن كود الاختبار ما زال يحتوي على استدعاء `time.Sleep` لأننا ننتظر خادمنا حتى يقوم بعمله بشكل غير متزامن.

يمكننا إعادة هيكلة الدالتين المساعدتين `assertGameStartedWith` و`assertFinishCalledWith` بحيث تعيدان محاولة تحقّقيهما لمدة قصيرة قبل الفشل.

وإليك كيف يمكنك فعل ذلك مع `assertFinishCalledWith`، ويمكنك استخدام المنهج نفسه مع الدالة المساعدة الأخرى.

```go
func assertFinishCalledWith(t testing.TB, game *GameSpy, winner string) {
	t.Helper()

	passed := retryUntil(500*time.Millisecond, func() bool {
		return game.FinishCalledWith == winner
	})

	if !passed {
		t.Errorf("expected finish called with %q but got %q", winner, game.FinishCalledWith)
	}
}
```

وهذه هي طريقة تعريف `retryUntil`

```go
func retryUntil(d time.Duration, f func() bool) bool {
	deadline := time.Now().Add(d)
	for time.Now().Before(deadline) {
		if f() {
			return true
		}
	}
	return false
}
```

## الخلاصة

أصبح تطبيقنا الآن مكتملًا. فيمكن بدء لعبة بوكر من متصفح ويب، ويُبلَّغ المستخدمون بقيمة الرهان الإجباري مع مرور الوقت عبر WebSockets. وعندما تنتهي اللعبة يمكنهم تسجيل الفائز الذي يُحفظ باستخدام الكود الذي كتبناه قبل بضعة فصول. ويستطيع اللاعبون معرفة أفضل لاعب بوكر (أو أكثرهم حظًا) عبر نقطة النهاية `/league` في الموقع.

ارتكبنا أخطاء خلال الرحلة، لكن مع تدفق الـ TDD لم نكن يومًا بعيدين كثيرًا عن برمجية تعمل. وقد كنا أحرارًا في مواصلة التكرار والتجريب.

سيستعرض الفصل الأخير المنهج والتصميم الذي وصلنا إليه، ويربط بعض الأطراف المتروكة.

غطّينا بضعة أشياء في هذا الفصل

### WebSockets

* طريقة ملائمة لإرسال الرسائل بين العملاء والخوادم دون حاجة العميل إلى مواصلة الاستعلام (polling) من الخادم. وكود كل من العميل والخادم لدينا بسيط جدًا.
* اختبارها بديهي، لكن عليك الحذر من الطبيعة غير المتزامنة للاختبارات

### التعامل مع الكود في الاختبارات الذي قد يتأخر أو لا ينتهي

* أنشئ دوال مساعدة لإعادة محاولة التحققات وإضافة مهل زمنية (timeouts).
* يمكننا استخدام goroutines لضمان ألا تحجب التحققات أي شيء، ثم استخدام القنوات (channels) لتشير إلى أنها انتهت، أو لا.
* تحتوي حزمة `time` على بعض الدوال المفيدة التي ترسل أيضًا إشارات عبر القنوات عن أحداث زمنية، حتى نتمكن من ضبط مهل زمنية
