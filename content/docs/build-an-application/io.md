---
title: الـ IO والترتيب
weight: 290
---

# الإدخال والإخراج والترتيب

**[يمكنك العثور على كل كود هذا الفصل هنا](https://github.com/quii/learn-go-with-tests/tree/main/io)**

[في الفصل السابق](json.md) واصلنا تطوير تطبيقنا بإضافة نقطة نهاية (endpoint) جديدة هي `/league`. وفي الطريق تعلّمنا كيفية التعامل مع JSON، وتضمين الأنواع (embedding types)، والتوجيه (routing).

مالكة المنتج (product owner) منزعجة بعض الشيء لأن البرنامج يفقد النتائج عند إعادة تشغيل الخادم؛ والسبب أن تنفيذنا للمخزن (store) موجود في الذاكرة. وهي أيضًا غير راضية لأننا لم نستنتج أن نقطة النهاية `/league` ينبغي أن تُرجع اللاعبين مرتبين حسب عدد الانتصارات!

## الكود حتى الآن

```go
// server.go
package main

import (
	"encoding/json"
	"fmt"
	"net/http"
	"strings"
)

// PlayerStore stores score information about players
type PlayerStore interface {
	GetPlayerScore(name string) int
	RecordWin(name string)
	GetLeague() []Player
}

// Player stores a name with a number of wins
type Player struct {
	Name string
	Wins int
}

// PlayerServer is a HTTP interface for player information
type PlayerServer struct {
	store PlayerStore
	http.Handler
}

const jsonContentType = "application/json"

// NewPlayerServer creates a PlayerServer with routing configured
func NewPlayerServer(store PlayerStore) *PlayerServer {
	p := new(PlayerServer)

	p.store = store

	router := http.NewServeMux()
	router.Handle("/league", http.HandlerFunc(p.leagueHandler))
	router.Handle("/players/", http.HandlerFunc(p.playersHandler))

	p.Handler = router

	return p
}

func (p *PlayerServer) leagueHandler(w http.ResponseWriter, r *http.Request) {
	w.Header().Set("content-type", jsonContentType)
	json.NewEncoder(w).Encode(p.store.GetLeague())
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

func (i *InMemoryPlayerStore) GetLeague() []Player {
	var league []Player
	for name, wins := range i.store {
		league = append(league, Player{name, wins})
	}
	return league
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
	server := NewPlayerServer(NewInMemoryPlayerStore())
	log.Fatal(http.ListenAndServe(":5000", server))
}
```

يمكنك العثور على الاختبارات المقابلة في الرابط أعلى الفصل.

## خزّن البيانات

هناك عشرات قواعد البيانات التي يمكننا استخدامها لهذا الغرض، لكننا سنسلك نهجًا بسيطًا جدًا. سنخزّن بيانات هذا التطبيق في ملف بصيغة JSON.

فهذا يجعل البيانات قابلة للنقل جدًا، وتنفيذه بسيط نسبيًا.

لن يتوسّع جيدًا بشكل خاص، لكن بما أن هذا نموذج أولي (prototype) فسيكون جيدًا في الوقت الحالي. وإذا تغيّرت ظروفنا ولم يعد مناسبًا، سيكون من السهل استبداله بشيء مختلف بفضل تجريد `PlayerStore` الذي استخدمناه.

سنُبقي على `InMemoryPlayerStore` في الوقت الحالي حتى تظل اختبارات التكامل (integration tests) ناجحة بينما نطوّر مخزننا الجديد. وبمجرد أن نطمئن أن تنفيذنا الجديد كافٍ لنجاح اختبار التكامل، سنستبدله ثم نحذف `InMemoryPlayerStore`.

## اكتب الاختبار أولًا

بحلول الآن ينبغي أن تكون معتادًا على الواجهات (interfaces) في المكتبة القياسية لقراءة البيانات (`io.Reader`) وكتابة البيانات (`io.Writer`)، وعلى كيفية استخدام المكتبة القياسية لاختبار هذه الدوال دون الحاجة إلى ملفات حقيقية.

لكي يكتمل هذا العمل سنحتاج إلى تنفيذ `PlayerStore`، لذا سنكتب اختبارات لمخزننا تستدعي الـ methods التي نحتاج إلى تنفيذها. سنبدأ بـ `GetLeague`.

```go
//file_system_store_test.go
func TestFileSystemStore(t *testing.T) {

	t.Run("league from a reader", func(t *testing.T) {
		database := strings.NewReader(`[
			{"Name": "Cleo", "Wins": 10},
			{"Name": "Chris", "Wins": 33}]`)

		store := FileSystemPlayerStore{database}

		got := store.GetLeague()

		want := []Player{
			{"Cleo", 10},
			{"Chris", 33},
		}

		assertLeague(t, got, want)
	})
}
```

نستخدم `strings.NewReader` التي ستُرجع لنا `Reader`، وهو ما سيستخدمه `FileSystemPlayerStore` لقراءة البيانات. وفي `main` سنفتح ملفًا، وهو أيضًا `Reader`.

## جرّب تشغيل الاختبار

```
# github.com/quii/learn-go-with-tests/io/v1
./file_system_store_test.go:15:12: undefined: FileSystemPlayerStore
```

## اكتب أصغر قدر من الكود ليعمل الاختبار وافحص مخرجاته الفاشلة

لنعرّف `FileSystemPlayerStore` في ملف جديد

```go
//file_system_store.go
type FileSystemPlayerStore struct{}
```

جرّب مرة أخرى

```
# github.com/quii/learn-go-with-tests/io/v1
./file_system_store_test.go:15:28: too many values in struct initializer
./file_system_store_test.go:17:15: store.GetLeague undefined (type FileSystemPlayerStore has no field or method GetLeague)
```

إنه يشتكي لأننا نمرّر `Reader` دون أن نتوقع واحدًا، ولأن `GetLeague` غير معرّفة بعد.

```go
//file_system_store.go
type FileSystemPlayerStore struct {
	database io.Reader
}

func (f *FileSystemPlayerStore) GetLeague() []Player {
	return nil
}
```

محاولة أخرى...

```
=== RUN   TestFileSystemStore//league_from_a_reader
    --- FAIL: TestFileSystemStore//league_from_a_reader (0.00s)
        file_system_store_test.go:24: got [] want [{Cleo 10} {Chris 33}]
```

## اكتب كودًا كافيًا لنجاح الاختبار

لقد قرأنا JSON من reader من قبل

```go
//file_system_store.go
func (f *FileSystemPlayerStore) GetLeague() []Player {
	var league []Player
	json.NewDecoder(f.database).Decode(&league)
	return league
}
```

ينبغي أن ينجح الاختبار.

## إعادة الهيكلة

لقد فعلنا هذا _من قبل_! فكود اختبارنا للخادم كان عليه فكّ ترميز JSON من الاستجابة (response).

لنحاول إزالة التكرار (DRY) بجعل هذا دالة.

أنشئ ملفًا جديدًا اسمه `league.go` وضع داخله هذا.

```go
//league.go
func NewLeague(rdr io.Reader) ([]Player, error) {
	var league []Player
	err := json.NewDecoder(rdr).Decode(&league)
	if err != nil {
		err = fmt.Errorf("problem parsing league, %v", err)
	}

	return league, err
}
```

استدعِ هذا في تنفيذنا وفي دالة المساعدة في اختبارنا `getLeagueFromResponse` في `server_test.go`

```go
//file_system_store.go
func (f *FileSystemPlayerStore) GetLeague() []Player {
	league, _ := NewLeague(f.database)
	return league
}
```

ليس لدينا بعد استراتيجية للتعامل مع أخطاء التحليل (parsing errors)، لكن لنواصل.

### مشكلات الـ Seek

يوجد خلل في تنفيذنا. أولًا، لنذكّر أنفسنا كيف تُعرَّف `io.Reader`.

```go
type Reader interface {
	Read(p []byte) (n int, err error)
}
```

مع ملفنا، يمكنك تخيّل أنه يقرأ بايتًا بايتًا حتى النهاية. فماذا يحدث إذا حاولت `Read` مرة ثانية؟

أضف ما يلي إلى نهاية اختبارنا الحالي.

```go
//file_system_store_test.go

// read again
got = store.GetLeague()
assertLeague(t, got, want)
```

نريد أن ينجح هذا، لكنه لا ينجح عند تشغيل الاختبار.

المشكلة أن `Reader` وصل إلى النهاية فلم يعد هناك ما يُقرأ. نحتاج إلى طريقة لنطلب منه العودة إلى البداية.

[ReadSeeker](https://golang.org/pkg/io/#ReadSeeker) واجهة أخرى في المكتبة القياسية يمكنها المساعدة.

```go
type ReadSeeker interface {
	Reader
	Seeker
}
```

أتذكر التضمين؟ هذه واجهة مكوّنة من `Reader` و[`Seeker`](https://golang.org/pkg/io/#Seeker)

```go
type Seeker interface {
	Seek(offset int64, whence int) (int64, error)
}
```

يبدو هذا جيدًا؛ فهل يمكننا تغيير `FileSystemPlayerStore` ليأخذ هذه الواجهة بدلًا منها؟

```go
//file_system_store.go
type FileSystemPlayerStore struct {
	database io.ReadSeeker
}

func (f *FileSystemPlayerStore) GetLeague() []Player {
	f.database.Seek(0, io.SeekStart)
	league, _ := NewLeague(f.database)
	return league
}
```

جرّب تشغيل الاختبار، إنه ينجح الآن! ولحسن حظنا فإن `strings.NewReader` التي استخدمناها في اختبارنا تنفّذ أيضًا `ReadSeeker`، فلم نضطر إلى أي تغييرات أخرى.

بعد ذلك سننفّذ `GetPlayerScore`.

## اكتب الاختبار أولًا

```go
//file_system_store_test.go
t.Run("get player score", func(t *testing.T) {
	database := strings.NewReader(`[
		{"Name": "Cleo", "Wins": 10},
		{"Name": "Chris", "Wins": 33}]`)

	store := FileSystemPlayerStore{database}

	got := store.GetPlayerScore("Chris")

	want := 33

	if got != want {
		t.Errorf("got %d want %d", got, want)
	}
})
```

## جرّب تشغيل الاختبار

```
./file_system_store_test.go:38:15: store.GetPlayerScore undefined (type FileSystemPlayerStore has no field or method GetPlayerScore)
```

## اكتب أصغر قدر من الكود ليعمل الاختبار وافحص مخرجاته الفاشلة

علينا إضافة الـ method إلى نوعنا الجديد ليجتاز الاختبار مرحلة الترجمة.

```go
//file_system_store.go
func (f *FileSystemPlayerStore) GetPlayerScore(name string) int {
	return 0
}
```

الآن يترجم الكود ويفشل الاختبار

```
=== RUN   TestFileSystemStore/get_player_score
    --- FAIL: TestFileSystemStore//get_player_score (0.00s)
        file_system_store_test.go:43: got 0 want 33
```

## اكتب كودًا كافيًا لنجاح الاختبار

يمكننا المرور على الـ league للعثور على اللاعب وإرجاع نتيجته

```go
//file_system_store.go
func (f *FileSystemPlayerStore) GetPlayerScore(name string) int {

	var wins int

	for _, player := range f.GetLeague() {
		if player.Name == name {
			wins = player.Wins
			break
		}
	}

	return wins
}
```

## إعادة الهيكلة

ستكون قد رأيت العشرات من عمليات إعادة هيكلة دوال المساعدة في الاختبارات، لذا سأترك لك إتمامها

```go
//file_system_store_test.go
t.Run("get player score", func(t *testing.T) {
	database := strings.NewReader(`[
		{"Name": "Cleo", "Wins": 10},
		{"Name": "Chris", "Wins": 33}]`)

	store := FileSystemPlayerStore{database}

	got := store.GetPlayerScore("Chris")
	want := 33
	assertScoreEquals(t, got, want)
})
```

أخيرًا، علينا أن نبدأ تسجيل النتائج باستخدام `RecordWin`.

## اكتب الاختبار أولًا

نهجنا في الكتابة ضيّق الأفق نوعًا ما. فلا يمكننا (بسهولة) تحديث "صف" واحد فقط من JSON في ملف. سنحتاج إلى تخزين التمثيل الجديد _الكامل_ لقاعدة بياناتنا في كل عملية كتابة.

كيف نكتب؟ نستخدم عادةً `Writer`، لكن لدينا بالفعل `ReadSeeker`. وقد يكون لدينا اعتماديتان، لكن المكتبة القياسية توفّر لنا بالفعل واجهة `ReadWriteSeeker` التي تتيح لنا كل ما سنحتاج إلى فعله مع ملف.

لنحدّث نوعنا

```go
//file_system_store.go
type FileSystemPlayerStore struct {
	database io.ReadWriteSeeker
}
```

انظر هل يترجم

```
./file_system_store_test.go:15:34: cannot use database (type *strings.Reader) as type io.ReadWriteSeeker in field value:
    *strings.Reader does not implement io.ReadWriteSeeker (missing Write method)
./file_system_store_test.go:36:34: cannot use database (type *strings.Reader) as type io.ReadWriteSeeker in field value:
    *strings.Reader does not implement io.ReadWriteSeeker (missing Write method)
```

ليس مفاجئًا كثيرًا ألا تنفّذ `strings.Reader` الواجهة `ReadWriteSeeker`، فماذا نفعل؟

أمامنا خياران

- إنشاء ملف مؤقت لكل اختبار. فـ `*os.File` تنفّذ `ReadWriteSeeker`. ومن مزاياه أن يصبح أقرب إلى اختبار تكامل، فنحن نقرأ ونكتب فعلًا من نظام الملفات، ما يمنحنا مستوى ثقة عاليًا جدًا. أما عيوبه فأننا نفضّل اختبارات الوحدة لأنها أسرع وأبسط عمومًا. كما سنحتاج إلى عمل إضافي لإنشاء الملفات المؤقتة ثم التأكد من حذفها بعد الاختبار.
- يمكننا استخدام مكتبة خارجية. فقد كتب [Mattetti](https://github.com/mattetti) مكتبة [filebuffer](https://github.com/mattetti/filebuffer) تنفّذ الواجهة التي نحتاجها ولا تلمس نظام الملفات.

لا أظن أن هناك إجابة خاطئة بشكل خاص هنا، لكن باختياري استخدام مكتبة خارجية سأضطر إلى شرح إدارة الاعتماديات (dependency management)! لذا سنستخدم الملفات بدلًا من ذلك.

قبل إضافة اختبارنا، علينا جعل اختباراتنا الأخرى تترجم باستبدال `strings.Reader` بـ `os.File`.

لننشئ بعض دوال المساعدة التي تُنشئ ملفًا مؤقتًا يحتوي على بعض البيانات، ونجرّد اختبارات النتائج لدينا

```go
//file_system_store_test.go
func createTempFile(t testing.TB, initialData string) (io.ReadWriteSeeker, func()) {
	t.Helper()

	tmpfile, err := os.CreateTemp("", "db")

	if err != nil {
		t.Fatalf("could not create temp file %v", err)
	}

	tmpfile.Write([]byte(initialData))

	removeFile := func() {
		tmpfile.Close()
		os.Remove(tmpfile.Name())
	}

	return tmpfile, removeFile
}

func assertScoreEquals(t testing.TB, got, want int) {
	t.Helper()
	if got != want {
		t.Errorf("got %d want %d", got, want)
	}
}
```

[CreateTemp](https://pkg.go.dev/os#CreateTemp) تُنشئ ملفًا مؤقتًا لنا. والقيمة `"db"` التي مرّرناها بادئة تُضاف إلى اسم ملف عشوائي ستُنشئه. والهدف من ذلك ألا يتعارض مع ملفات أخرى بالمصادفة.

ستلاحظ أننا لا نُرجع `ReadWriteSeeker` (الملف) فحسب، بل نُرجع أيضًا دالة. فعلينا التأكد من حذف الملف بعد انتهاء الاختبار. ولا نريد تسريب تفاصيل الملفات إلى الاختبار لأن ذلك عرضة للخطأ وغير مثير للقارئ. وبإرجاع دالة `removeFile` نعتني بالتفاصيل في دالة المساعدة، وكل ما على المستدعي فعله هو تشغيل `defer removeFile()`.

```go
//file_system_store_test.go
func TestFileSystemStore(t *testing.T) {

	t.Run("league from a reader", func(t *testing.T) {
		database, removeFile := createTempFile(t, `[
			{"Name": "Cleo", "Wins": 10},
			{"Name": "Chris", "Wins": 33}]`)
		defer removeFile()

		store := FileSystemPlayerStore{database}

		got := store.GetLeague()

		want := []Player{
			{"Cleo", 10},
			{"Chris", 33},
		}

		assertLeague(t, got, want)

		// read again
		got = store.GetLeague()
		assertLeague(t, got, want)
	})

	t.Run("get player score", func(t *testing.T) {
		database, removeFile := createTempFile(t, `[
			{"Name": "Cleo", "Wins": 10},
			{"Name": "Chris", "Wins": 33}]`)
		defer removeFile()

		store := FileSystemPlayerStore{database}

		got := store.GetPlayerScore("Chris")
		want := 33
		assertScoreEquals(t, got, want)
	})
}
```

شغّل الاختبارات وينبغي أن تنجح! كانت هناك تغييرات كثيرة، لكن يبدو الآن أن تعريف واجهتنا صار مكتملًا، وينبغي أن يكون إضافة اختبارات جديدة سهلًا جدًا من الآن فصاعدًا.

لنبدأ النسخة الأولى من تسجيل فوز لاعب موجود

```go
//file_system_store_test.go
t.Run("store wins for existing players", func(t *testing.T) {
	database, removeFile := createTempFile(t, `[
		{"Name": "Cleo", "Wins": 10},
		{"Name": "Chris", "Wins": 33}]`)
	defer removeFile()

	store := FileSystemPlayerStore{database}

	store.RecordWin("Chris")

	got := store.GetPlayerScore("Chris")
	want := 34
	assertScoreEquals(t, got, want)
})
```

## جرّب تشغيل الاختبار

`./file_system_store_test.go:67:8: store.RecordWin undefined (type FileSystemPlayerStore has no field or method RecordWin)`

## اكتب أصغر قدر من الكود ليعمل الاختبار وافحص مخرجاته الفاشلة

أضف الـ method الجديدة

```go
//file_system_store.go
func (f *FileSystemPlayerStore) RecordWin(name string) {

}
```

```
=== RUN   TestFileSystemStore/store_wins_for_existing_players
    --- FAIL: TestFileSystemStore/store_wins_for_existing_players (0.00s)
        file_system_store_test.go:71: got 33 want 34
```

تنفيذنا فارغ، لذا تُرجع النتيجة القديمة.

## اكتب كودًا كافيًا لنجاح الاختبار

```go
//file_system_store.go
func (f *FileSystemPlayerStore) RecordWin(name string) {
	league := f.GetLeague()

	for i, player := range league {
		if player.Name == name {
			league[i].Wins++
		}
	}

	f.database.Seek(0, io.SeekStart)
	json.NewEncoder(f.database).Encode(league)
}
```

قد تسأل نفسك لماذا أفعل `league[i].Wins++` بدلًا من `player.Wins++`.

عند استخدام `range` على شريحة (slice) تحصل على الفهرس الحالي للحلقة (وهو `i` في حالتنا) وعلى _نسخة_ من العنصر عند ذلك الفهرس. وتغيير قيمة `Wins` في نسخة لن يؤثر على شريحة `league` التي نمر عليها. لهذا السبب نحتاج إلى مرجع القيمة الفعلية عبر `league[i]` ثم تغيير تلك القيمة بدلًا من ذلك.

إذا شغّلت الاختبارات، ينبغي أن تنجح الآن.

## إعادة الهيكلة

في `GetPlayerScore` و`RecordWin` نمر على `[]Player` للعثور على لاعب بالاسم.

يمكننا إعادة هيكلة هذا الكود المشترك داخل `FileSystemStore`، لكنه يبدو لي كودًا مفيدًا قد نرفعه إلى نوع جديد. فالعمل مع "League" كان حتى الآن دائمًا عبر `[]Player`، لكن يمكننا إنشاء نوع جديد اسمه `League`. سيكون ذلك أسهل على المطورين الآخرين في الفهم، ثم يمكننا إرفاق methods مفيدة بذلك النوع لنستخدمها.

داخل `league.go` أضف ما يلي

```go
//league.go
type League []Player

func (l League) Find(name string) *Player {
	for i, p := range l {
		if p.Name == name {
			return &l[i]
		}
	}
	return nil
}
```

الآن إذا كان لدى أي أحد `League` فسيسهل عليه العثور على لاعب معيّن.

غيّر واجهة `PlayerStore` لدينا لتُرجع `League` بدلًا من `[]Player`. جرّب إعادة تشغيل الاختبارات وستحصل على مشكلة ترجمة لأننا غيّرنا الواجهة، لكن إصلاحها سهل جدًا؛ فقط غيّر نوع الإرجاع من `[]Player` إلى `League`.

يتيح لنا هذا تبسيط الـ methods في `file_system_store`.

```go
//file_system_store.go
func (f *FileSystemPlayerStore) GetPlayerScore(name string) int {

	player := f.GetLeague().Find(name)

	if player != nil {
		return player.Wins
	}

	return 0
}

func (f *FileSystemPlayerStore) RecordWin(name string) {
	league := f.GetLeague()
	player := league.Find(name)

	if player != nil {
		player.Wins++
	}

	f.database.Seek(0, io.SeekStart)
	json.NewEncoder(f.database).Encode(league)
}
```

هذا يبدو أفضل بكثير، ويمكننا أن نرى كيف قد نجد وظائف مفيدة أخرى حول `League` يمكن إعادة هيكلتها.

نحتاج الآن إلى التعامل مع سيناريو تسجيل انتصارات لاعبين جدد.

## اكتب الاختبار أولًا

```go
//file_system_store_test.go
t.Run("store wins for new players", func(t *testing.T) {
	database, removeFile := createTempFile(t, `[
		{"Name": "Cleo", "Wins": 10},
		{"Name": "Chris", "Wins": 33}]`)
	defer removeFile()

	store := FileSystemPlayerStore{database}

	store.RecordWin("Pepper")

	got := store.GetPlayerScore("Pepper")
	want := 1
	assertScoreEquals(t, got, want)
})
```

## جرّب تشغيل الاختبار

```
=== RUN   TestFileSystemStore/store_wins_for_new_players#01
    --- FAIL: TestFileSystemStore/store_wins_for_new_players#01 (0.00s)
        file_system_store_test.go:86: got 0 want 1
```

## اكتب كودًا كافيًا لنجاح الاختبار

كل ما علينا فعله هو التعامل مع سيناريو إرجاع `Find` للقيمة `nil` لأنها لم تجد اللاعب.

```go
//file_system_store.go
func (f *FileSystemPlayerStore) RecordWin(name string) {
	league := f.GetLeague()
	player := league.Find(name)

	if player != nil {
		player.Wins++
	} else {
		league = append(league, Player{name, 1})
	}

	f.database.Seek(0, io.SeekStart)
	json.NewEncoder(f.database).Encode(league)
}
```

المسار السعيد يبدو جيدًا، لذا يمكننا الآن تجربة استخدام `Store` الجديد في اختبار التكامل. سيمنحنا ذلك ثقة أكبر بأن البرنامج يعمل، ثم يمكننا حذف `InMemoryPlayerStore` الزائد.

في `TestRecordingWinsAndRetrievingThem` استبدل المخزن القديم.

```go
//server_integration_test.go
database, removeFile := createTempFile(t, "")
defer removeFile()
store := &FileSystemPlayerStore{database}
```

إذا شغّلت الاختبار فينبغي أن ينجح، والآن يمكننا حذف `InMemoryPlayerStore`. وسيصبح لدى `main.go` مشكلات ترجمة تدفعنا الآن إلى استخدام مخزننا الجديد في الكود "الحقيقي".

```go
// main.go
package main

import (
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

	store := &FileSystemPlayerStore{db}
	server := NewPlayerServer(store)

	if err := http.ListenAndServe(":5000", server); err != nil {
		log.Fatalf("could not listen on port 5000 %v", err)
	}
}
```

- نُنشئ ملفًا لقاعدة بياناتنا.
- الوسيط الثاني لـ `os.OpenFile` يتيح لك تحديد أذونات (permissions) فتح الملف؛ وفي حالتنا تعني `O_RDWR` أننا نريد القراءة والكتابة _و_ تعني `os.O_CREATE` إنشاء الملف إن لم يكن موجودًا.
- الوسيط الثالث يحدد أذونات الملف؛ وفي حالتنا يمكن لجميع المستخدمين قراءة الملف وكتابته. [(انظر superuser.com لشرح أكثر تفصيلًا)](https://superuser.com/questions/295591/what-is-the-meaning-of-chmod-666).

تشغيل البرنامج الآن يحفظ البيانات في ملف بين عمليات إعادة التشغيل، مرحى!

## مزيد من إعادة الهيكلة وهواجس الأداء

في كل مرة يستدعي فيها أحد `GetLeague()` أو `GetPlayerScore()` نقرأ الملف بالكامل ونحلّله إلى JSON. ولا ينبغي أن نضطر إلى ذلك، لأن `FileSystemStore` مسؤولة بالكامل عن حالة الـ league؛ فينبغي أن تحتاج إلى قراءة الملف فقط عند بدء البرنامج، وإلى تحديث الملف فقط عند تغيّر البيانات.

يمكننا إنشاء دالة إنشاء تقوم ببعض هذه التهيئة نيابةً عنا وتخزّن الـ league كقيمة في `FileSystemStore` لتُستخدم في القراءات بدلًا من ذلك.

```go
//file_system_store.go
type FileSystemPlayerStore struct {
	database io.ReadWriteSeeker
	league   League
}

func NewFileSystemPlayerStore(database io.ReadWriteSeeker) *FileSystemPlayerStore {
	database.Seek(0, io.SeekStart)
	league, _ := NewLeague(database)
	return &FileSystemPlayerStore{
		database: database,
		league:   league,
	}
}
```

بهذه الطريقة لن نقرأ من القرص إلا مرة واحدة. ويمكننا الآن استبدال كل استدعاءاتنا السابقة لجلب الـ league من القرص واستخدام `f.league` بدلًا من ذلك.

```go
//file_system_store.go
func (f *FileSystemPlayerStore) GetLeague() League {
	return f.league
}

func (f *FileSystemPlayerStore) GetPlayerScore(name string) int {

	player := f.league.Find(name)

	if player != nil {
		return player.Wins
	}

	return 0
}

func (f *FileSystemPlayerStore) RecordWin(name string) {
	player := f.league.Find(name)

	if player != nil {
		player.Wins++
	} else {
		f.league = append(f.league, Player{name, 1})
	}

	f.database.Seek(0, io.SeekStart)
	json.NewEncoder(f.database).Encode(f.league)
}
```

إذا حاولت تشغيل الاختبارات فستشتكي الآن من تهيئة `FileSystemPlayerStore`، فأصلحها فقط باستدعاء دالة الإنشاء الجديدة.

### مشكلة أخرى

لا يزال في طريقة تعاملنا مع الملفات بعض السذاجة التي _قد_ تُنشئ خطأً سيئًا جدًا لاحقًا.

عندما ننفّذ `RecordWin` نعود بـ `Seek` إلى بداية الملف ثم نكتب البيانات الجديدة — لكن ماذا لو كانت البيانات الجديدة أصغر مما كان موجودًا قبلها؟

في حالتنا الحالية هذا مستحيل؛ فنحن لا نعدّل النتائج ولا نحذفها، لذا لا يمكن للبيانات إلا أن تكبر. ومع ذلك، سيكون من غير المسؤول أن نترك الكود هكذا؛ فليس من المستبعد أن تظهر حالة حذف.

لكن كيف سنختبر هذا؟ ما علينا فعله أولًا هو إعادة هيكلة كودنا بحيث نفصل بين الاهتمام بـ _نوع البيانات التي نكتبها وبين عملية الكتابة_. ثم يمكننا اختبار ذلك بشكل منفصل للتأكد من أنه يعمل كما نأمل.

سننشئ نوعًا جديدًا يغلّف وظيفة "عند الكتابة نبدأ من البداية". سأسميه `Tape`. أنشئ ملفًا جديدًا بهذا المحتوى:

```go
// tape.go
package main

import "io"

type tape struct {
	file io.ReadWriteSeeker
}

func (t *tape) Write(p []byte) (n int, err error) {
	t.file.Seek(0, io.SeekStart)
	return t.file.Write(p)
}
```

لاحظ أننا ننفّذ الآن `Write` فقط، لأنها تغلّف جزء `Seek`. هذا يعني أن `FileSystemStore` يمكنه أن يمتلك مرجعًا إلى `Writer` فقط بدلًا من ذلك.

```go
//file_system_store.go
type FileSystemPlayerStore struct {
	database io.Writer
	league   League
}
```

حدّث دالة الإنشاء لتستخدم `Tape`

```go
//file_system_store.go
func NewFileSystemPlayerStore(database io.ReadWriteSeeker) *FileSystemPlayerStore {
	database.Seek(0, io.SeekStart)
	league, _ := NewLeague(database)

	return &FileSystemPlayerStore{
		database: &tape{database},
		league:   league,
	}
}
```

أخيرًا، يمكننا نيل المكافأة الرائعة التي أردناها بحذف استدعاء `Seek` من `RecordWin`. نعم، لا يبدو الأمر كبيرًا، لكنه على الأقل يعني أننا إن أجرينا أي نوع آخر من الكتابات يمكننا الاعتماد على `Write` لتتصرّف كما نحتاج. كما سيتيح لنا الآن اختبار الكود الذي قد يسبب مشكلات بشكل منفصل وإصلاحه.

لنكتب الاختبار الذي نريد فيه تحديث محتوى ملف بالكامل بشيء أصغر من المحتوى الأصلي.

## اكتب الاختبار أولًا

سيُنشئ اختبارنا ملفًا ببعض المحتوى، ويحاول الكتابة إليه باستخدام `tape`، ثم يقرؤه كاملًا مرة أخرى ليرى ما في الملف. في `tape_test.go`:

```go
//tape_test.go
func TestTape_Write(t *testing.T) {
	file, clean := createTempFile(t, "12345")
	defer clean()

	tape := &tape{file}

	tape.Write([]byte("abc"))

	file.Seek(0, io.SeekStart)
	newFileContents, _ := io.ReadAll(file)

	got := string(newFileContents)
	want := "abc"

	if got != want {
		t.Errorf("got %q want %q", got, want)
	}
}
```

## جرّب تشغيل الاختبار

```
=== RUN   TestTape_Write
--- FAIL: TestTape_Write (0.00s)
    tape_test.go:23: got 'abc45' want 'abc'
```

كما توقعنا! إنه يكتب البيانات التي نريد، لكنه يترك بقية البيانات الأصلية.

## اكتب كودًا كافيًا لنجاح الاختبار

لدى `os.File` دالة truncate تتيح لنا إفراغ الملف فعليًا. وينبغي أن يكفي استدعاؤها للحصول على ما نريد.

غيّر `tape` إلى ما يلي:

```go
//tape.go
type tape struct {
	file *os.File
}

func (t *tape) Write(p []byte) (n int, err error) {
	t.file.Truncate(0)
	t.file.Seek(0, io.SeekStart)
	return t.file.Write(p)
}
```

سيفشل المترجم في عدة مواضع نتوقع فيها `io.ReadWriteSeeker` لكننا نمرّر `*os.File`. ينبغي أن تكون قادرًا على إصلاح هذه المشكلات بنفسك الآن، لكن إن واجهت صعوبة فراجع الكود المصدري.

وبعد إتمام إعادة الهيكلة ينبغي أن ينجح اختبار `TestTape_Write`!

### إعادة هيكلة صغيرة أخرى

في `RecordWin` لدينا السطر `json.NewEncoder(f.database).Encode(f.league)`.

لا نحتاج إلى إنشاء encoder جديد في كل مرة نكتب؛ يمكننا تهيئة واحد في دالة الإنشاء واستخدامه بدلًا من ذلك.

خزّن مرجعًا إلى `Encoder` في نوعنا واهيّئه في دالة الإنشاء:

```go
//file_system_store.go
type FileSystemPlayerStore struct {
	database *json.Encoder
	league   League
}

func NewFileSystemPlayerStore(file *os.File) *FileSystemPlayerStore {
	file.Seek(0, io.SeekStart)
	league, _ := NewLeague(file)

	return &FileSystemPlayerStore{
		database: json.NewEncoder(&tape{file}),
		league:   league,
	}
}
```

تنتقل `tape.Write` إلى بداية الملف في _كل_ استدعاء، لذا يجدر التوقف للتحقق من أن هذا التركيب آمن فعلًا: لو استدعت `Encode` الدالة `Write` أكثر من مرة للاستدعاء نفسه، فستعود كل كتابة تالية إلى البداية وتطمس ما قبلها، ما يُفسد المخرجات. لكنها لا تفعل ذلك — فـ `Encoder.Encode` تحوّل القيمة كاملة إلى الذاكرة أولًا ثم تمرّرها إلى `Writer` الأساسي في استدعاء `Write` واحد، لذا لا تنتقل `tape` إلا مرة واحدة لكل `Encode`، وهو بالضبط سلوك "الكتابة من البداية دائمًا" الذي نريده.

استخدمه في `RecordWin`.

```go
func (f *FileSystemPlayerStore) RecordWin(name string) {
	player := f.league.Find(name)

	if player != nil {
		player.Wins++
	} else {
		f.league = append(f.league, Player{name, 1})
	}

	f.database.Encode(f.league)
}
```

## ألم نكسر بعض القواعد للتو؟ اختبار الأشياء الخاصة؟ بلا واجهات؟

### عن اختبار الأنواع الخاصة

صحيح أنه _عمومًا_ ينبغي أن تفضّل عدم اختبار الأشياء الخاصة، لأن ذلك قد يجعل اختباراتك مرتبطة بالتنفيذ ارتباطًا وثيقًا أحيانًا، ما قد يعيق إعادة الهيكلة مستقبلًا.

لكن علينا ألا ننسى أن الاختبارات ينبغي أن تمنحنا _الثقة_.

لم نكن واثقين أن تنفيذنا سيعمل إذا أضفنا أي نوع من وظائف التعديل أو الحذف. ولم نرد ترك الكود على هذه الحال، خصوصًا لو كان يعمل عليه أكثر من شخص قد لا يكون على علم بمآخذ نهجنا الأول.

وأخيرًا، إنه اختبار واحد فقط! فإذا قررنا تغيير طريقة عمله فلن يكون حذفه كارثة، لكننا على الأقل وثّقنا المتطلب للمشرفين عليه مستقبلًا.

### الواجهات (Interfaces)

بدأنا الكود باستخدام `io.Reader` لأنها كانت أسهل طريق لاختبار `PlayerStore` الجديد كوحدة. ومع تطور الكود انتقلنا إلى `io.ReadWriter` ثم إلى `io.ReadWriteSeeker`. ثم اكتشفنا أنه لا يوجد شيء في المكتبة القياسية ينفّذ تلك الواجهة فعلًا سوى `*os.File`. وكان يمكننا أن نقرر كتابة واحدة بأنفسنا أو استخدام واحدة مفتوحة المصدر، لكن بدا عمليًا أن نُعد ملفات مؤقتة للاختبارات فقط.

وأخيرًا احتجنا إلى `Truncate` الموجودة أيضًا في `*os.File`. وكان من الخيارات أن ننشئ واجهتنا الخاصة التي تجسّد هذه المتطلبات.

```go
type ReadWriteSeekTruncate interface {
	io.ReadWriteSeeker
	Truncate(size int64) error
}
```

لكن ما الذي يمنحنا هذا حقًا؟ تذكّر أننا _لا نستخدم mocks_، ومن غير الواقعي أن يأخذ مخزن **نظام ملفات** أي نوع غير `*os.File`، لذا لا نحتاج إلى تعدد الأشكال (polymorphism) الذي تمنحنا إياه الواجهات.

لا تخف من تغيير الأنواع وتبديلها والتجربة كما فعلنا هنا. والميزة الرائعة في استخدام لغة ذات أنواع ثابتة أن المترجم سيساعدك في كل تغيير.

## معالجة الأخطاء

قبل أن نبدأ العمل على الترتيب، علينا أن نتأكد أننا راضون عن كودنا الحالي ونزيل أي دين تقني (technical debt) قد يكون لدينا. فمن المبادئ المهمة الوصول إلى برنامج يعمل بأسرع ما يمكن (وأن تبقى خارج الحالة الحمراء)، لكن هذا لا يعني أن نتجاهل حالات الخطأ!

إذا عدنا إلى `file_system_store.go` نجد `league, _ := NewLeague(file)` في دالة الإنشاء.

يمكن أن تُرجع `NewLeague` خطأً إذا لم تتمكن من تحليل الـ league من `*os.File` الذي نمرّره.

كان من العملي تجاهل ذلك في ذلك الوقت لأن لدينا اختبارات فاشلة بالفعل. ولو حاولنا معالجته في الوقت نفسه لكنا نتلاعب بأمرين معًا.

لنجعل دالة الإنشاء قادرة على إرجاع خطأ.

```go
//file_system_store.go
func NewFileSystemPlayerStore(file *os.File) (*FileSystemPlayerStore, error) {
	file.Seek(0, io.SeekStart)
	league, err := NewLeague(file)

	if err != nil {
		return nil, fmt.Errorf("problem loading player store from file %s, %v", file.Name(), err)
	}

	return &FileSystemPlayerStore{
		database: json.NewEncoder(&tape{file}),
		league:   league,
	}, nil
}
```

تذكّر أنه من المهم جدًا تقديم رسائل خطأ مفيدة (تمامًا كما في اختباراتك). يقول الناس على الإنترنت مازحين إن معظم كود Go هو:

```go
if err != nil {
	return err
}
```

**هذا ليس أسلوبًا اصطلاحيًا (idiomatic) بنسبة 100%.** فإضافة معلومات سياقية (أي ما كنت تفعله وأنت تسبب الخطأ) إلى رسائل الخطأ تجعل تشغيل برنامجك أسهل بكثير.

إذا حاولت الترجمة فستحصل على بعض الأخطاء.

```
./main.go:18:35: multiple-value NewFileSystemPlayerStore() in single-value context
./file_system_store_test.go:35:36: multiple-value NewFileSystemPlayerStore() in single-value context
./file_system_store_test.go:57:36: multiple-value NewFileSystemPlayerStore() in single-value context
./file_system_store_test.go:70:36: multiple-value NewFileSystemPlayerStore() in single-value context
./file_system_store_test.go:85:36: multiple-value NewFileSystemPlayerStore() in single-value context
./server_integration_test.go:12:35: multiple-value NewFileSystemPlayerStore() in single-value context
```

في main سنريد إنهاء البرنامج مع طباعة الخطأ.

```go
//main.go
store, err := NewFileSystemPlayerStore(db)

if err != nil {
	log.Fatalf("problem creating file system player store, %v ", err)
}
```

في الاختبارات ينبغي أن نتحقق من عدم وجود خطأ. ويمكننا إنشاء دالة مساعدة تساعدنا في ذلك.

```go
//file_system_store_test.go
func assertNoError(t testing.TB, err error) {
	t.Helper()
	if err != nil {
		t.Fatalf("didn't expect an error but got one, %v", err)
	}
}
```

عالج مشكلات الترجمة الأخرى باستخدام دالة المساعدة هذه. وأخيرًا، ينبغي أن يكون لديك اختبار فاشل:

```
=== RUN   TestRecordingWinsAndRetrievingThem
--- FAIL: TestRecordingWinsAndRetrievingThem (0.00s)
    server_integration_test.go:14: didn't expect an error but got one, problem loading player store from file /var/folders/nj/r_ccbj5d7flds0sf63yy4vb80000gn/T/db841037437, problem parsing league, EOF
```

لا يمكننا تحليل الـ league لأن الملف فارغ. ولم تكن تظهر لنا أخطاء من قبل لأننا كنا نتجاهلها دائمًا.

لنصلح اختبار التكامل الكبير بوضع بعض JSON الصالح فيه:

```go
//server_integration_test.go
func TestRecordingWinsAndRetrievingThem(t *testing.T) {
	database, removeFile := createTempFile(t, `[]`)
	//etc...
}
```

الآن وقد نجحت كل الاختبارات، علينا التعامل مع سيناريو كون الملف فارغًا.

## اكتب الاختبار أولًا

```go
//file_system_store_test.go
t.Run("works with an empty file", func(t *testing.T) {
	database, removeFile := createTempFile(t, "")
	defer removeFile()

	_, err := NewFileSystemPlayerStore(database)

	assertNoError(t, err)
})
```

## جرّب تشغيل الاختبار

```
=== RUN   TestFileSystemStore/works_with_an_empty_file
    --- FAIL: TestFileSystemStore/works_with_an_empty_file (0.00s)
        file_system_store_test.go:108: didn't expect an error but got one, problem loading player store from file /var/folders/nj/r_ccbj5d7flds0sf63yy4vb80000gn/T/db019548018, problem parsing league, EOF
```

## اكتب كودًا كافيًا لنجاح الاختبار

غيّر دالة الإنشاء إلى ما يلي

```go
//file_system_store.go
func NewFileSystemPlayerStore(file *os.File) (*FileSystemPlayerStore, error) {

	file.Seek(0, io.SeekStart)

	info, err := file.Stat()

	if err != nil {
		return nil, fmt.Errorf("problem getting file info from file %s, %v", file.Name(), err)
	}

	if info.Size() == 0 {
		file.Write([]byte("[]"))
		file.Seek(0, io.SeekStart)
	}

	league, err := NewLeague(file)

	if err != nil {
		return nil, fmt.Errorf("problem loading player store from file %s, %v", file.Name(), err)
	}

	return &FileSystemPlayerStore{
		database: json.NewEncoder(&tape{file}),
		league:   league,
	}, nil
}
```

تُرجع `file.Stat` إحصاءات عن ملفنا، ما يتيح لنا فحص حجم الملف. فإن كان فارغًا نكتب `Write` مصفوفة JSON فارغة ثم نعود بـ `Seek` إلى البداية، استعدادًا لبقية الكود.

## إعادة الهيكلة

دالة الإنشاء صارت فوضوية بعض الشيء الآن، فلنستخرج كود التهيئة إلى دالة:

```go
//file_system_store.go
func initialisePlayerDBFile(file *os.File) error {
	file.Seek(0, io.SeekStart)

	info, err := file.Stat()

	if err != nil {
		return fmt.Errorf("problem getting file info from file %s, %v", file.Name(), err)
	}

	if info.Size() == 0 {
		file.Write([]byte("[]"))
		file.Seek(0, io.SeekStart)
	}

	return nil
}
```

```go
//file_system_store.go
func NewFileSystemPlayerStore(file *os.File) (*FileSystemPlayerStore, error) {

	err := initialisePlayerDBFile(file)

	if err != nil {
		return nil, fmt.Errorf("problem initialising player db file, %v", err)
	}

	league, err := NewLeague(file)

	if err != nil {
		return nil, fmt.Errorf("problem loading player store from file %s, %v", file.Name(), err)
	}

	return &FileSystemPlayerStore{
		database: json.NewEncoder(&tape{file}),
		league:   league,
	}, nil
}
```

## الترتيب

تريد مالكة المنتج أن تُرجع `/league` اللاعبين مرتبين حسب نتائجهم، من الأعلى إلى الأدنى.

القرار الرئيسي هنا هو أين ينبغي أن يحدث هذا في البرنامج. فلو كنا نستخدم قاعدة بيانات "حقيقية" لاستخدمنا أشياء مثل `ORDER BY` ليكون الترتيب فائق السرعة. لهذا السبب يبدو أن تنفيذات `PlayerStore` هي التي ينبغي أن تكون مسؤولة.

## اكتب الاختبار أولًا

يمكننا تحديث التحقق (assertion) في اختبارنا الأول داخل `TestFileSystemStore`:

```go
//file_system_store_test.go
t.Run("league sorted", func(t *testing.T) {
	database, removeFile := createTempFile(t, `[
		{"Name": "Cleo", "Wins": 10},
		{"Name": "Chris", "Wins": 33}]`)
	defer removeFile()

	store, err := NewFileSystemPlayerStore(database)

	assertNoError(t, err)

	got := store.GetLeague()

	want := League{
		{"Chris", 33},
		{"Cleo", 10},
	}

	assertLeague(t, got, want)

	// read again
	got = store.GetLeague()
	assertLeague(t, got, want)
})
```

ترتيب JSON القادم ترتيب خاطئ، وسيتحقق `want` لدينا من إرجاعه إلى المستدعي بالترتيب الصحيح.

## جرّب تشغيل الاختبار

```
=== RUN   TestFileSystemStore/league_from_a_reader,_sorted
    --- FAIL: TestFileSystemStore/league_from_a_reader,_sorted (0.00s)
        file_system_store_test.go:46: got [{Cleo 10} {Chris 33}] want [{Chris 33} {Cleo 10}]
        file_system_store_test.go:51: got [{Cleo 10} {Chris 33}] want [{Chris 33} {Cleo 10}]
```

## اكتب كودًا كافيًا لنجاح الاختبار

```go
func (f *FileSystemPlayerStore) GetLeague() League {
	sort.Slice(f.league, func(i, j int) bool {
		return f.league[i].Wins > f.league[j].Wins
	})
	return f.league
}
```

[`sort.Slice`](https://golang.org/pkg/sort/#Slice)

> ترتّب Slice الشريحة المعطاة وفق دالة المقارنة (less function) المعطاة.

سهل!

## الخلاصة

### ما غطيناه

- واجهة `Seeker` وعلاقتها بـ `Reader` و`Writer`.
- التعامل مع الملفات.
- إنشاء دالة مساعدة سهلة الاستخدام للاختبار بالملفات تُخفي كل الأمور الفوضوية.
- `sort.Slice` لترتيب الشرائح.
- استخدام المترجم لمساعدتنا في إجراء تغييرات هيكلية على التطبيق بأمان.

### كسر القواعد

- معظم القواعد في هندسة البرمجيات ليست قواعد حقًا، بل ممارسات فضلى تعمل 80% من الوقت.
- اكتشفنا سيناريو لم تكن فيه إحدى "قواعدنا" السابقة بعدم اختبار الدوال الداخلية مفيدة لنا، فكسرنا القاعدة.
- من المهم عند كسر القواعد أن تفهم المقايضة (trade-off) التي تقوم بها. وفي حالتنا تقبّلناها لأنها كانت اختبارًا واحدًا فقط، وكان سيكون من الصعب جدًا ممارسة هذا السيناريو بغير ذلك.
- ولكي تستطيع كسر القواعد **عليك أن تفهمها أولًا**. ومن التشبيهات على ذلك تعلّم الغيتار؛ فلا يهم كم ترى نفسك مبدعًا، عليك أن تفهم الأساسيات وتتدرب عليها.

### أين وصلت برمجيتنا

- لدينا واجهة HTTP برمجية (HTTP API) يمكنك فيها إنشاء لاعبين وزيادة نتائجهم.
- يمكننا إرجاع league بنتائج الجميع بصيغة JSON.
- البيانات محفوظة بشكل دائم في ملف JSON.
