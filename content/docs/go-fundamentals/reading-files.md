---
title: قراءة الملفات
weight: 180
---

# قراءة الملفات

* [**يمكنك العثور على كل كود هذا الفصل هنا**](https://github.com/quii/learn-go-with-tests/tree/main/reading-files)
* [هذا فيديو لي وأنا أعمل على المسألة وأجيب عن أسئلة بث Twitch](https://www.youtube.com/watch?v=nXts4dEJnkU)

في هذا الفصل سنتعلم كيف نقرأ بعض الملفات، ونستخرج منها بعض البيانات، ونفعل شيئًا مفيدًا.

تخيّل أنك تعمل مع صديقك على بناء برمجيات تدوين. الفكرة أن يكتب الكاتب منشوراته بصيغة markdown، مع بعض البيانات الوصفية (metadata) في أعلى الملف. وعند بدء التشغيل، سيقرأ خادم الويب مجلدًا ليبني بعض `Post`s، ثم تستخدم دالة `NewHandler` منفصلة تلك `Post`s كمصدر بيانات لخادم المدونة.

طُلب منا إنشاء الحزمة التي تحوّل مجلدًا معطى من ملفات المنشورات إلى مجموعة من `Post`s.

### مثال على البيانات

hello world.md

```markdown
Title: Hello, TDD world!
Description: First post on our wonderful blog
Tags: tdd, go
---
Hello world!

The body of posts starts after the `---`
```

### البيانات المتوقعة

```go
type Post struct {
	Title, Description, Body string
	Tags                     []string
}
```

## التطوير التكراري الموجه بالاختبار

سنتّبع مقاربة تكرارية نتقدم فيها دائمًا بخطوات بسيطة وآمنة نحو هدفنا.

وهذا يتطلب منا تقسيم عملنا، لكن علينا أن نحترس من الوقوع في فخ اتباع مقاربة ["من الأسفل إلى الأعلى"](https://en.wikipedia.org/wiki/Top-down_and_bottom-up_design).

لا ينبغي أن نثق بخيالاتنا المفرطة النشاط عندما نبدأ العمل. فقد نُغرى ببناء نوع من التجريد لا تتحقق صحته إلا بعد أن نجمع كل شيء معًا، مثل `BlogPostFileParser` ما.

هذا _ليس_ تكراريًا، ويُفوّت علينا حلقات التغذية الراجعة المحكمة التي يُفترض أن يمنحنا إياها التطوير الموجه بالاختبار.

يقول Kent Beck:

> التفاؤل مخاطرة مهنية ملازمة للبرمجة. والتغذية الراجعة هي العلاج.

بل ينبغي أن تسعى مقاربتنا إلى تقديم قيمة _حقيقية_ للمستهلك بأسرع ما يمكن (وهو ما يُسمى غالبًا "المسار السعيد"). وعندما نقدّم قدرًا صغيرًا من القيمة للمستهلك من البداية إلى النهاية، تصبح بقية المتطلبات عادةً يسيرة في التكرارات التالية.

## التفكير في نوع الاختبار الذي نريد رؤيته

لنذكّر أنفسنا بعقليتنا وأهدافنا عند البدء:

* **اكتب الاختبار الذي تريد رؤيته**. فكّر كيف نحب أن نستخدم الكود الذي سنكتبه من وجهة نظر المستهلك.
* ركّز على _ماذا_ و_لماذا_، ولا تتشتت بـ _كيف_.

تحتاج حزمتنا إلى تقديم دالة يمكن توجيهها إلى مجلد، فتُعيد لنا بعض المنشورات.

```go
var posts []blogposts.Post
posts = blogposts.NewPostsFromFS("some-folder")
```

ولكتابة اختبار حول هذا، سنحتاج إلى نوع من مجلد اختبارات فيه بعض المنشورات النموذجية. _لا شيء خطأ في هذا على الإطلاق_، لكنك تقبل بعض المقايضات:

* لكل اختبار قد تحتاج إلى إنشاء ملفات جديدة لاختبار سلوك بعينه
* بعض السلوكيات سيكون اختبارها صعبًا، مثل الفشل في تحميل الملفات
* ستعمل الاختبارات أبطأ قليلًا لأنها ستحتاج إلى الوصول إلى نظام الملفات

كما أننا نربط أنفسنا بلا داعٍ بتطبيق بعينه لنظام الملفات.

### تجريدات نظام الملفات التي جاءت في Go 1.16

قدّمت Go 1.16 تجريدًا لأنظمة الملفات؛ إنها حزمة [io/fs](https://golang.org/pkg/io/fs/).

> تعرّف حزمة fs واجهات أساسية لنظام ملفات. ويمكن أن يوفّر نظام الملفات نظام التشغيل المضيف، ويمكن أن توفّره حزم أخرى أيضًا.

هذا يتيح لنا إرخاء ارتباطنا بنظام ملفات بعينه، ما يسمح لنا بعدها بحقن تطبيقات مختلفة حسب احتياجاتنا.

> [في جانب المنتِج من الواجهة، يحقق النوع الجديد embed.FS الواجهة fs.FS، وكذلك يفعل zip.Reader. وتوفّر الدالة الجديدة os.DirFS تطبيقًا للواجهة fs.FS مدعومًا بشجرة من ملفات نظام التشغيل.](https://golang.org/doc/go1.16#fs)

إذا استخدمنا هذه الواجهة، سيحصل مستخدمو حزمتنا على عدد من الخيارات المدمجة في المكتبة القياسية. وتعلّم الاستفادة من الواجهات المعرّفة في مكتبة Go القياسية (مثل `io.fs` و[`io.Reader`](https://golang.org/pkg/io/#Reader) و[`io.Writer`](https://golang.org/pkg/io/#Writer)) أمر حيوي لكتابة حزم متراخية الارتباط. وبعد ذلك يمكن إعادة استخدام هذه الحزم في سياقات مختلفة عن التي تخيلتها، بأقل عناء من مستهلكيك.

وفي حالتنا، ربما يريد المستهلك أن تُضمَّن المنشورات داخل ملف Go التنفيذي بدلًا من ملفات في نظام ملفات "حقيقي"؟ في الحالتين، _لا يحتاج كودنا إلى الاهتمام_.

ولاختباراتنا، توفّر لنا حزمة [testing/fstest](https://golang.org/pkg/testing/fstest/) تطبيقًا للواجهة [io/FS](https://golang.org/pkg/io/fs/#FS) لنستخدمه، شبيهًا بالأدوات التي نعرفها في [net/http/httptest](https://golang.org/pkg/net/http/httptest/).

بالنظر إلى هذه المعلومات، تبدو المقاربة التالية أفضل،

```go
var posts []blogposts.Post
posts = blogposts.NewPostsFromFS(someFS)
```

## اكتب الاختبار أولًا

ينبغي أن نبقي النطاق صغيرًا ومفيدًا قدر الإمكان. فإذا أثبتنا أننا نستطيع قراءة كل الملفات في مجلد، فستكون تلك بداية جيدة. وسيمنحنا ذلك ثقة في البرمجيات التي نكتبها. يمكننا التحقق من أن عدد `[]Post` المُرجَع يساوي عدد الملفات في نظام الملفات الوهمي.

أنشئ مشروعًا جديدًا للعمل على هذا الفصل.

* `mkdir blogposts`
* `cd blogposts`
* `go mod init github.com/{your-name}/blogposts`
* `touch blogposts_test.go`

```go
package blogposts_test

import (
	"testing"
	"testing/fstest"
)

func TestNewBlogPosts(t *testing.T) {
	fs := fstest.MapFS{
		"hello world.md":  {Data: []byte("hi")},
		"hello-world2.md": {Data: []byte("hola")},
	}

	posts := blogposts.NewPostsFromFS(fs)

	if len(posts) != len(fs) {
		t.Errorf("got %d posts, wanted %d posts", len(posts), len(fs))
	}
}
```

لاحظ أن حزمة اختبارنا هي `blogposts_test`. وتذكّر، عندما نمارس التطوير الموجه بالاختبار جيدًا فإننا نتبنى مقاربة _مدفوعة بالمستهلك_: لا نريد اختبار التفاصيل الداخلية لأن _المستهلكين_ لا يهتمون بها. وبإضافة `_test` إلى اسم حزمتنا المقصودة، لا نصل إلا إلى الأعضاء المُصدَّرة من حزمتنا — تمامًا كمستخدم حقيقي لحزمتنا.

استوردنا [`testing/fstest`](https://golang.org/pkg/testing/fstest/) التي تتيح لنا الوصول إلى النوع [`fstest.MapFS`](https://golang.org/pkg/testing/fstest/#MapFS). فسيمرّر نظام ملفاتنا الوهمي `fstest.MapFS` إلى حزمتنا.

> إن MapFS نظام ملفات بسيط في الذاكرة للاستخدام في الاختبارات، ويُمثَّل كخريطة من أسماء المسارات (الوسائط الممرَّرة إلى Open) إلى معلومات عن الملفات أو المجلدات التي تمثلها.

هذا يبدو أبسط من صيانة مجلد من ملفات الاختبار، وسيُنفَّذ أسرع.

وأخيرًا، وثّقنا استخدام واجهتنا (API) من وجهة نظر المستهلك، ثم فحصنا ما إذا كانت تُنشئ العدد الصحيح من المنشورات.

## جرّب تشغيل الاختبار

```
./blogpost_test.go:15:12: undefined: blogposts
```

## اكتب أصغر قدر من الكود ليعمل الاختبار و_افحص مخرجاته الفاشلة_

الحزمة غير موجودة. أنشئ ملفًا جديدًا اسمه `blogposts.go` وضع داخله `package blogposts`. وستحتاج بعد ذلك إلى استيراد تلك الحزمة في اختباراتك. بالنسبة لي، تبدو الاستيرادات الآن هكذا:

```go
import (
	blogposts "github.com/quii/learn-go-with-tests/reading-files"
	"testing"
	"testing/fstest"
)
```

الآن لن تُترجم الاختبارات لأن حزمتنا الجديدة لا تحتوي على دالة `NewPostsFromFS` تُرجع نوعًا من المجموعات.

```
./blogpost_test.go:16:12: undefined: blogposts.NewPostsFromFS
```

وهذا يدفعنا إلى إنشاء هيكل دالتنا ليعمل الاختبار. تذكّر ألا تفرط في التفكير في الكود عند هذه النقطة؛ فكل ما نحاول فعله هو الحصول على اختبار يعمل، والتأكد أنه يفشل كما نتوقع. وإذا تخطينا هذه الخطوة فربما تخطينا بعض الافتراضات، وكتبنا اختبارًا غير مفيد.

```go
package blogposts

import "testing/fstest"

type Post struct {
}

func NewPostsFromFS(fileSystem fstest.MapFS) []Post {
	return nil
}
```

ينبغي أن يفشل الاختبار الآن بشكل صحيح

```
=== RUN   TestNewBlogPosts
    blogposts_test.go:48: got 0 posts, wanted 2 posts
```

## اكتب كودًا كافيًا لنجاح الاختبار

كان _بإمكاننا_ استخدام ["slime"](https://deniseyu.github.io/leveling-up-tdd/) لجعله ينجح:

```go
func NewPostsFromFS(fileSystem fstest.MapFS) []Post {
	return []Post{{}, {}}
}
```

لكن، كما كتبت Denise Yu:

> الـ sliming مفيد لمنح كائنك "هيكلًا". فتصميم واجهة وتنفيذ منطق هما اهتمامان منفصلان، واستخدام اختبارات الـ sliming بشكل مدروس يتيح لك التركيز على واحد منهما في كل مرة.

لدينا هيكلنا بالفعل. فما الذي نفعله بدلًا من ذلك؟

بما أننا قلّصنا النطاق، كل ما علينا فعله هو قراءة المجلد وإنشاء منشور لكل ملف نصادفه. ولا داعي للقلق بعد بشأن فتح الملفات وتحليلها.

```go
func NewPostsFromFS(fileSystem fstest.MapFS) []Post {
	dir, _ := fs.ReadDir(fileSystem, ".")
	var posts []Post
	for range dir {
		posts = append(posts, Post{})
	}
	return posts
}
```

تقرأ [`fs.ReadDir`](https://golang.org/pkg/io/fs/#ReadDir) مجلدًا داخل `fs.FS` معطى وتُرجع [`[]DirEntry`](https://golang.org/pkg/io/fs/#DirEntry).

لقد تحطمت رؤيتنا المثالية للعالم بالفعل لأن الأخطاء يمكن أن تحدث، لكن تذكّر أن تركيزنا الآن هو _جعل الاختبار ينجح_، لا تغيير التصميم، لذا سنتجاهل الخطأ في الوقت الحالي.

وبقية الكود مباشرة: نمر على المدخلات، وننشئ `Post` لكل واحد منها، ثم نُعيد الشريحة.

## إعادة الهيكلة

رغم أن اختباراتنا تنجح، لا نستطيع استخدام حزمتنا الجديدة خارج هذا السياق، لأنها مرتبطة بتطبيق ملموس هو `fstest.MapFS`. لكن لا يجب أن تكون كذلك. غيّر وسيط دالة `NewPostsFromFS` ليقبل الواجهة من المكتبة القياسية.

```go
func NewPostsFromFS(fileSystem fs.FS) []Post {
	dir, _ := fs.ReadDir(fileSystem, ".")
	var posts []Post
	for range dir {
		posts = append(posts, Post{})
	}
	return posts
}
```

أعد تشغيل الاختبارات: ينبغي أن يعمل كل شيء.

### معالجة الأخطاء

أخّرنا معالجة الأخطاء سابقًا عندما ركزنا على إنجاح المسار السعيد. وقبل أن نواصل التكرار على الوظائف، يجدر بنا أن نقرّ بأن الأخطاء يمكن أن تحدث عند التعامل مع الملفات. فإلى جانب قراءة المجلد، قد نواجه مشكلات عند فتح الملفات فردًا فردًا. لنغيّر واجهتنا (عبر اختباراتنا أولًا، بطبيعة الحال) لتستطيع إرجاع `error`.

```go
func TestNewBlogPosts(t *testing.T) {
	fs := fstest.MapFS{
		"hello world.md":  {Data: []byte("hi")},
		"hello-world2.md": {Data: []byte("hola")},
	}

	posts, err := blogposts.NewPostsFromFS(fs)

	if err != nil {
		t.Fatal(err)
	}

	if len(posts) != len(fs) {
		t.Errorf("got %d posts, wanted %d posts", len(posts), len(fs))
	}
}
```

شغّل الاختبار: ينبغي أن يشتكي من العدد الخاطئ للقيم المُرجعة. وإصلاح الكود مباشر.

```go
func NewPostsFromFS(fileSystem fs.FS) ([]Post, error) {
	dir, err := fs.ReadDir(fileSystem, ".")
	if err != nil {
		return nil, err
	}
	var posts []Post
	for range dir {
		posts = append(posts, Post{})
	}
	return posts, nil
}
```

سيجعل هذا الاختبار ينجح. وربما ينزعج ممارس التطوير الموجه بالاختبار بداخلك لأننا لم نرَ اختبارًا فاشلًا قبل كتابة كود نشر الخطأ من `fs.ReadDir`. وللقيام بذلك "بشكل صحيح"، سنحتاج إلى اختبار جديد نحقن فيه بديلًا اختباريًا (test-double) فاشلًا للواجهة `fs.FS` ليجعل `fs.ReadDir` تُرجع `error`.

```go
type StubFailingFS struct {
}

func (s StubFailingFS) Open(name string) (fs.File, error) {
	return nil, errors.New("oh no, i always fail")
}
```

```go
// later
_, err := blogposts.NewPostsFromFS(StubFailingFS{})
```

ينبغي أن يمنحك هذا ثقة في مقاربتنا. فالواجهة التي نستخدمها لها method واحدة، ما يجعل إنشاء بدائل اختبارية لاختبار سيناريوهات مختلفة أمرًا بالغ السهولة.

في بعض الحالات، يكون اختبار معالجة الأخطاء هو الأمر العملي، لكننا في حالتنا لا نفعل شيئًا _مثيرًا للاهتمام_ بالخطأ، بل ننشره فقط، لذا لا يستحق الأمر عناء كتابة اختبار جديد.

منطقيًا، ستتمحور تكراراتنا التالية حول توسيع نوع `Post` ليكون فيه بعض البيانات المفيدة.

## اكتب الاختبار أولًا

سنبدأ بالسطر الأول في مخطط المنشور المقترح، وهو حقل العنوان.

نحتاج إلى تغيير محتوى ملفات الاختبار لتطابق ما حُدّد، ثم يمكننا التحقق من أنه يُحلَّل بشكل صحيح.

```go
func TestNewBlogPosts(t *testing.T) {
	fs := fstest.MapFS{
		"hello world.md":  {Data: []byte("Title: Post 1")},
		"hello-world2.md": {Data: []byte("Title: Post 2")},
	}

	// rest of test code cut for brevity
	got := posts[0]
	want := blogposts.Post{Title: "Post 1"}

	if !reflect.DeepEqual(got, want) {
		t.Errorf("got %+v, want %+v", got, want)
	}
}
```

## جرّب تشغيل الاختبار

```
./blogpost_test.go:58:26: unknown field 'Title' in struct literal of type blogposts.Post
```

## اكتب أصغر قدر من الكود ليعمل الاختبار وافحص مخرجاته الفاشلة

أضف الحقل الجديد إلى نوع `Post` ليعمل الاختبار

```go
type Post struct {
	Title string
}
```

أعد تشغيل الاختبار، وينبغي أن تحصل على اختبار فاشل واضح

```
=== RUN   TestNewBlogPosts
=== RUN   TestNewBlogPosts/parses_the_post
    blogpost_test.go:61: got {Title:}, want {Title:Post 1}
```

## اكتب كودًا كافيًا لنجاح الاختبار

سنحتاج إلى فتح كل ملف ثم استخراج العنوان

```go
func NewPostsFromFS(fileSystem fs.FS) ([]Post, error) {
	dir, err := fs.ReadDir(fileSystem, ".")
	if err != nil {
		return nil, err
	}
	var posts []Post
	for _, f := range dir {
		post, err := getPost(fileSystem, f)
		if err != nil {
			return nil, err //todo: needs clarification, should we totally fail if one file fails? or just ignore?
		}
		posts = append(posts, post)
	}
	return posts, nil
}

func getPost(fileSystem fs.FS, f fs.DirEntry) (Post, error) {
	postFile, err := fileSystem.Open(f.Name())
	if err != nil {
		return Post{}, err
	}
	defer postFile.Close()

	postData, err := io.ReadAll(postFile)
	if err != nil {
		return Post{}, err
	}

	post := Post{Title: string(postData)[7:]}
	return post, nil
}
```

تذكّر أن تركيزنا عند هذه النقطة ليس كتابة كود أنيق، بل الوصول إلى نقطة يكون لدينا فيها برمجية تعمل.

ورغم أن هذا يبدو خطوة صغيرة إلى الأمام، فقد تطلب منا كتابة قدر لا بأس به من الكود ووضع بعض الافتراضات بشأن معالجة الأخطاء. وستكون هذه نقطة من المناسب أن تتحدث فيها مع زملائك وتقرروا أفضل مقاربة.

منحتنا المقاربة التكرارية تغذية راجعة سريعة بأن فهمنا للمتطلبات غير مكتمل.

تمنحنا `fs.FS` طريقة لفتح ملف بداخلها بالاسم عبر method اسمها `Open`. ومن هناك نقرأ البيانات من الملف، ولا نحتاج الآن إلى أي تحليل متطور، بل فقط قطع نص `Title:` بتقطيع النص.

## إعادة الهيكلة

فصل "كود فتح الملف" عن "كود تحليل محتوى الملف" سيجعل الكود أبسط فهمًا وتعاملًا.

```go
func getPost(fileSystem fs.FS, f fs.DirEntry) (Post, error) {
	postFile, err := fileSystem.Open(f.Name())
	if err != nil {
		return Post{}, err
	}
	defer postFile.Close()
	return newPost(postFile)
}

func newPost(postFile fs.File) (Post, error) {
	postData, err := io.ReadAll(postFile)
	if err != nil {
		return Post{}, err
	}

	post := Post{Title: string(postData)[7:]}
	return post, nil
}
```

عندما تستخرج دوالًا أو methods جديدة بإعادة الهيكلة، اعتنِ بالوسائط وفكّر فيها. فأنت تصمّم هنا، ولك حرية التفكير بعمق فيما هو مناسب لأن اختباراتك تنجح. فكّر في الارتباط (coupling) والتماسك (cohesion). وفي هذه الحالة ينبغي أن تسأل نفسك:

> هل يجب أن تكون `newPost` مرتبطة بـ `fs.File`؟ هل نستخدم كل الـ methods والبيانات من هذا النوع؟ ما الذي نحتاجه _فعلًا_؟

في حالتنا نستخدمه فقط كوسيط لـ `io.ReadAll` التي تحتاج إلى `io.Reader`. لذا ينبغي أن نرخي الارتباط في دالتنا ونطلب `io.Reader`.

```go
func newPost(postFile io.Reader) (Post, error) {
	postData, err := io.ReadAll(postFile)
	if err != nil {
		return Post{}, err
	}

	post := Post{Title: string(postData)[7:]}
	return post, nil
}
```

ويمكنك أن تقول الشيء نفسه عن دالة `getPost`، التي تأخذ وسيطًا من النوع `fs.DirEntry` لكنها تستدعي `Name()` فقط للحصول على اسم الملف. لا نحتاج كل ذلك؛ لنفصل الارتباط عن ذلك النوع ونمرّر اسم الملف كنص. وإليك الكود بعد إعادة الهيكلة كاملة:

```go
func NewPostsFromFS(fileSystem fs.FS) ([]Post, error) {
	dir, err := fs.ReadDir(fileSystem, ".")
	if err != nil {
		return nil, err
	}
	var posts []Post
	for _, f := range dir {
		post, err := getPost(fileSystem, f.Name())
		if err != nil {
			return nil, err //todo: needs clarification, should we totally fail if one file fails? or just ignore?
		}
		posts = append(posts, post)
	}
	return posts, nil
}

func getPost(fileSystem fs.FS, fileName string) (Post, error) {
	postFile, err := fileSystem.Open(fileName)
	if err != nil {
		return Post{}, err
	}
	defer postFile.Close()
	return newPost(postFile)
}

func newPost(postFile io.Reader) (Post, error) {
	postData, err := io.ReadAll(postFile)
	if err != nil {
		return Post{}, err
	}

	post := Post{Title: string(postData)[7:]}
	return post, nil
}
```

من الآن فصاعدًا، يمكن أن يتركز معظم جهدنا داخل `newPost` بترتيب أنيق. فقد انتهينا من هموم فتح الملفات والمرور عليها، ويمكننا الآن التركيز على استخراج البيانات لنوع `Post`. ورغم أن ذلك ليس ضروريًا تقنيًا، فالملفات طريقة جميلة لتجميع الأشياء المرتبطة منطقيًا معًا، لذا نقلت نوع `Post` ودالة `newPost` إلى ملف جديد اسمه `post.go`.

### دالة مساعدة للاختبارات

ينبغي أن نعتني باختباراتنا أيضًا. سنقوم بالتحقق من `Posts` كثيرًا، لذا يجدر بنا كتابة بعض الكود الذي يساعدنا في ذلك

```go
func assertPost(t *testing.T, got blogposts.Post, want blogposts.Post) {
	t.Helper()
	if !reflect.DeepEqual(got, want) {
		t.Errorf("got %+v, want %+v", got, want)
	}
}
```

```go
assertPost(t, posts[0], blogposts.Post{Title: "Post 1"})
```

## اكتب الاختبار أولًا

لنوسّع اختبارنا أكثر ليستخرج السطر التالي من الملف، وهو الوصف. وحتى إنجاحه ينبغي أن يكون الآن مريحًا ومألوفًا.

```go
func TestNewBlogPosts(t *testing.T) {
	const (
		firstBody = `Title: Post 1
Description: Description 1`
		secondBody = `Title: Post 2
Description: Description 2`
	)

	fs := fstest.MapFS{
		"hello world.md":  {Data: []byte(firstBody)},
		"hello-world2.md": {Data: []byte(secondBody)},
	}

	// rest of test code cut for brevity
	assertPost(t, posts[0], blogposts.Post{
		Title:       "Post 1",
		Description: "Description 1",
	})

}
```

## جرّب تشغيل الاختبار

```
./blogpost_test.go:47:58: unknown field 'Description' in struct literal of type blogposts.Post
```

## اكتب أصغر قدر من الكود ليعمل الاختبار وافحص مخرجاته الفاشلة

أضف الحقل الجديد إلى `Post`.

```go
type Post struct {
	Title       string
	Description string
}
```

ينبغي أن تترجم الاختبارات الآن، وأن تفشل.

```
=== RUN   TestNewBlogPosts
    blogpost_test.go:47: got {Title:Post 1
        Description: Description 1 Description:}, want {Title:Post 1 Description:Description 1}
```

## اكتب كودًا كافيًا لنجاح الاختبار

تحتوي المكتبة القياسية على حزمة مفيدة تساعدك على المرور على البيانات سطرًا سطرًا؛ إنها [`bufio.Scanner`](https://golang.org/pkg/bufio/#Scanner)

> توفّر Scanner واجهة مريحة لقراءة البيانات مثل ملف من أسطر نصية مفصولة بمحارف السطر الجديد.

```go
func newPost(postFile io.Reader) (Post, error) {
	scanner := bufio.NewScanner(postFile)

	scanner.Scan()
	titleLine := scanner.Text()

	scanner.Scan()
	descriptionLine := scanner.Text()

	return Post{Title: titleLine[7:], Description: descriptionLine[13:]}, nil
}
```

ومن المفيد أنها تأخذ أيضًا `io.Reader` لتقرأ منه (شكرًا مرة أخرى، أيها الارتباط المتراخي)، فلسنا بحاجة إلى تغيير وسائط دالتنا.

استدعِ `Scan` لقراءة سطر، ثم استخرج البيانات باستخدام `Text`.

لا يمكن لهذه الدالة أن تُرجع `error` أبدًا. وسيكون مغريًا عند هذه النقطة إزالته من نوع الإرجاع، لكننا نعلم أننا سنضطر إلى معالجة بنى ملفات غير صالحة لاحقًا، لذا قد نتركه كما هو.

## إعادة الهيكلة

لدينا تكرار حول مسح سطر ثم قراءة النص. ونعلم أننا سنقوم بهذه العملية مرة أخرى على الأقل، وهي إعادة هيكلة بسيطة لتطبيق DRY فلنبدأ بها.

```go
func newPost(postFile io.Reader) (Post, error) {
	scanner := bufio.NewScanner(postFile)

	readLine := func() string {
		scanner.Scan()
		return scanner.Text()
	}

	title := readLine()[7:]
	description := readLine()[13:]

	return Post{Title: title, Description: description}, nil
}
```

هذا لم يوفّر إلا أسطرًا قليلة من الكود، لكن هذا نادرًا ما يكون هدف إعادة الهيكلة. ما أحاول فعله هنا هو فصل _ماذا_ عن _كيف_ في قراءة الأسطر ليكون الكود أكثر وصفية بقليل للقارئ.

ورغم أن الرقمين السحريين 7 و13 ينجزان المهمة، فهما ليسا وصفيين كثيرًا.

```go
const (
	titleSeparator       = "Title: "
	descriptionSeparator = "Description: "
)

func newPost(postFile io.Reader) (Post, error) {
	scanner := bufio.NewScanner(postFile)

	readLine := func() string {
		scanner.Scan()
		return scanner.Text()
	}

	title := readLine()[len(titleSeparator):]
	description := readLine()[len(descriptionSeparator):]

	return Post{Title: title, Description: description}, nil
}
```

الآن وقد حدّقت في الكود بعقلية إعادة الهيكلة الإبداعية، أود أن أحاول جعل دالة readLine تتولى إزالة الوسم. وهناك أيضًا طريقة أكثر قابلية للقراءة لإزالة بادئة من نص عبر الدالة `strings.TrimPrefix`.

```go
func newPost(postBody io.Reader) (Post, error) {
	scanner := bufio.NewScanner(postBody)

	readMetaLine := func(tagName string) string {
		scanner.Scan()
		return strings.TrimPrefix(scanner.Text(), tagName)
	}

	return Post{
		Title:       readMetaLine(titleSeparator),
		Description: readMetaLine(descriptionSeparator),
	}, nil
}
```

قد تعجبك هذه الفكرة أو لا تعجبك، لكنها تعجبني. والفكرة أنه في مرحلة إعادة الهيكلة نكون أحرارًا في العبث بالتفاصيل الداخلية، ويمكنك مواصلة تشغيل اختباراتك لتتأكد أن الأمور ما زالت تسير بشكل صحيح. ويمكننا دائمًا العودة إلى الحالات السابقة إن لم نكن راضين. فمقاربة التطوير الموجه بالاختبار تمنحنا هذا الترخيص لتجربة الأفكار كثيرًا، لتصبح لدينا فرص أكثر لكتابة كود رائع.

المتطلب التالي هو استخراج وسوم المنشور (tags). وإذا كنت تتابع معنا، أنصحك بأن تحاول تنفيذه بنفسك قبل مواصلة القراءة. ينبغي أن تكون لديك الآن إيقاع تكراري جيد وأن تشعر بالثقة لاستخراج السطر التالي وتحليل البيانات.

وللاختصار، لن أستعرض خطوات التطوير الموجه بالاختبار، لكن إليك الاختبار بعد إضافة الوسوم.

```go
func TestNewBlogPosts(t *testing.T) {
	const (
		firstBody = `Title: Post 1
Description: Description 1
Tags: tdd, go`
		secondBody = `Title: Post 2
Description: Description 2
Tags: rust, borrow-checker`
	)

	// rest of test code cut for brevity
	assertPost(t, posts[0], blogposts.Post{
		Title:       "Post 1",
		Description: "Description 1",
		Tags:        []string{"tdd", "go"},
	})
}
```

أنت تغش نفسك فقط إذا نسخت ولصقت ما أكتبه. وللتأكد أننا جميعًا على الصفحة نفسها، إليك كودي الذي يتضمن استخراج الوسوم.

```go
const (
	titleSeparator       = "Title: "
	descriptionSeparator = "Description: "
	tagsSeparator        = "Tags: "
)

func newPost(postBody io.Reader) (Post, error) {
	scanner := bufio.NewScanner(postBody)

	readMetaLine := func(tagName string) string {
		scanner.Scan()
		return strings.TrimPrefix(scanner.Text(), tagName)
	}

	return Post{
		Title:       readMetaLine(titleSeparator),
		Description: readMetaLine(descriptionSeparator),
		Tags:        strings.Split(readMetaLine(tagsSeparator), ", "),
	}, nil
}
```

آمل ألا تكون هناك مفاجآت هنا. فقد استطعنا إعادة استخدام `readMetaLine` للحصول على السطر التالي الخاص بالوسوم، ثم تقسيمها باستخدام `strings.Split`.

والتكرار الأخير على مسارنا السعيد هو استخراج المتن (body).

وإليك تذكيرًا بصيغة الملف المقترحة.

```markdown
Title: Hello, TDD world!
Description: First post on our wonderful blog
Tags: tdd, go
---
Hello world!

The body of posts starts after the `---`
```

لقد قرأنا الأسطر الثلاثة الأولى بالفعل. ثم نحتاج إلى قراءة سطر واحد آخر، وإهماله، وبعدها يحتوي بقية الملف على متن المنشور.

## اكتب الاختبار أولًا

غيّر بيانات الاختبار لتحتوي على الفاصل، ومتنًا فيه بضعة أسطر جديدة لنفحص أننا نلتقط كل المحتوى.

```go
	const (
		firstBody = `Title: Post 1
Description: Description 1
Tags: tdd, go
---
Hello
World`
		secondBody = `Title: Post 2
Description: Description 2
Tags: rust, borrow-checker
---
B
L
M`
	)
```

أضف إلى تحققنا مثل البقية

```go
	assertPost(t, posts[0], blogposts.Post{
		Title:       "Post 1",
		Description: "Description 1",
		Tags:        []string{"tdd", "go"},
		Body: `Hello
World`,
	})
```

## جرّب تشغيل الاختبار

```
./blogpost_test.go:60:3: unknown field 'Body' in struct literal of type blogposts.Post
```

كما توقعنا.

## اكتب أصغر قدر من الكود ليعمل الاختبار وافحص مخرجاته الفاشلة

أضف `Body` إلى `Post` وسينبغي أن يفشل الاختبار.

```
=== RUN   TestNewBlogPosts
    blogposts_test.go:38: got {Title:Post 1 Description:Description 1 Tags:[tdd go] Body:}, want {Title:Post 1 Description:Description 1 Tags:[tdd go] Body:Hello
        World}
```

## اكتب كودًا كافيًا لنجاح الاختبار

1. امسح السطر التالي لتتجاهل الفاصل `---`.
2. واصل المسح حتى لا يبقى شيء لمسحه.

```go
func newPost(postBody io.Reader) (Post, error) {
	scanner := bufio.NewScanner(postBody)

	readMetaLine := func(tagName string) string {
		scanner.Scan()
		return strings.TrimPrefix(scanner.Text(), tagName)
	}

	title := readMetaLine(titleSeparator)
	description := readMetaLine(descriptionSeparator)
	tags := strings.Split(readMetaLine(tagsSeparator), ", ")

	scanner.Scan() // ignore a line

	var b strings.Builder
	for scanner.Scan() {
		fmt.Fprintln(&b, scanner.Text())
	}
	body := strings.TrimSuffix(b.String(), "\n")

	return Post{
		Title:       title,
		Description: description,
		Tags:        tags,
		Body:        body,
	}, nil
}
```

* تُرجع `scanner.Scan()` قيمة `bool` تدل على وجود بيانات أخرى لمسحها، لذا يمكننا استخدامها مع حلقة `for` لمواصلة القراءة حتى النهاية.
* بعد كل `Scan()` نكتب البيانات في الـ buffer باستخدام `fmt.Fprintln`. ونستخدم النسخة التي تضيف سطرًا جديدًا لأن الـ scanner يزيل الأسطر الجديدة من كل سطر، لكننا نحتاج إلى الحفاظ عليها.
* وبسبب ما سبق، نحتاج إلى إزالة سطر جديد الأخير، حتى لا يبقى لدينا سطر جديد زائد في النهاية.

## إعادة الهيكلة

تغليف فكرة الحصول على بقية البيانات في دالة سيساعد القراء مستقبلًا على فهم _ما_ يحدث في `newPost` بسرعة، دون أن يشغلوا أنفسهم بتفاصيل التنفيذ.

```go
func newPost(postBody io.Reader) (Post, error) {
	scanner := bufio.NewScanner(postBody)

	readMetaLine := func(tagName string) string {
		scanner.Scan()
		return strings.TrimPrefix(scanner.Text(), tagName)
	}

	return Post{
		Title:       readMetaLine(titleSeparator),
		Description: readMetaLine(descriptionSeparator),
		Tags:        strings.Split(readMetaLine(tagsSeparator), ", "),
		Body:        readBody(scanner),
	}, nil
}

func readBody(scanner *bufio.Scanner) string {
	scanner.Scan() // ignore a line
	var b strings.Builder
	for scanner.Scan() {
		fmt.Fprintln(&b, scanner.Text())
	}
	return strings.TrimSuffix(b.String(), "\n")
}
```

## مواصلة التكرار

لقد أنجزنا "خيطنا الفولاذي" من الوظائف، بأخذ أقصر طريق للوصول إلى مسارنا السعيد، لكن من الواضح أن هناك مسافة يجب قطعها قبل أن يصبح جاهزًا للإنتاج.

لم نعالج:

* عندما تكون صيغة الملف غير صحيحة
* عندما لا يكون الملف بامتداد `.md`
* ماذا لو كان ترتيب حقول البيانات الوصفية مختلفًا؟ هل ينبغي السماح بذلك؟ وهل ينبغي أن نستطيع معالجته؟

لكن الأهم أن لدينا برمجية تعمل، وعرّفنا واجهتنا. وما سبق ليس إلا تكرارات إضافية، واختبارات أكثر نكتبها لتقود سلوكنا. ولدعم أي من ذلك لا ينبغي أن نغيّر _تصميمنا_، بل تفاصيل التنفيذ فقط.

وإبقاء التركيز على الهدف يعني أننا اتخذنا القرارات المهمة، وتحقّقنا من صحتها مقابل السلوك المطلوب، بدلًا من الغرق في أمور لن تؤثر على التصميم العام.

## الخلاصة

تمنحنا `fs.FS` وبقية التغييرات في Go 1.16 طرقًا أنيقة لقراءة البيانات من أنظمة الملفات واختبارها ببساطة.

إذا أردت تجربة الكود "على الحقيقي":

* أنشئ مجلدًا اسمه `cmd` داخل المشروع، وأضف ملفًا اسمه `main.go`
* أضف الكود التالي

```go
import (
	blogposts "github.com/quii/fstest-spike"
	"log"
	"os"
)

func main() {
	posts, err := blogposts.NewPostsFromFS(os.DirFS("posts"))
	if err != nil {
		log.Fatal(err)
	}
	log.Println(posts)
}
```

* أضف بعض ملفات markdown إلى مجلد `posts` وشغّل البرنامج!

لاحظ التناظر بين كود الإنتاج

```go
posts, err := blogposts.NewPostsFromFS(os.DirFS("posts"))
```

والاختبارات

```go
posts, err := blogposts.NewPostsFromFS(fs)
```

هنا يبدو التطوير الموجه بالاختبار المدفوع بالمستهلك ومن الأعلى إلى الأسفل _صحيحًا_.

يستطيع مستخدم حزمتنا أن ينظر إلى اختباراتنا ويفهم بسرعة ما يُفترض أن تفعله وكيف يستخدمها. وكمشرفين على الصيانة، يمكننا أن نكون _واثقين أن اختباراتنا مفيدة لأنها من وجهة نظر المستهلك_. فنحن لا نختبر تفاصيل التنفيذ أو غيرها من التفاصيل العارضة، لذا يمكننا أن نكون واثقين بدرجة معقولة أن اختباراتنا ستساعدنا لا تعيقنا عند إعادة الهيكلة.

وبالاعتماد على ممارسات هندسة برمجيات جيدة مثل [**حقن الاعتماديات**](dependency-injection.md) يصبح كودنا بسيطًا في اختباره وإعادة استخدامه.

عندما تُنشئ حزمًا، حتى لو كانت داخلية في مشروعك فقط، فضّل مقاربة من الأعلى إلى الأسفل مدفوعة بالمستهلك. فهذا سيمنعك من الإفراط في تخيّل التصاميم وبناء تجريدات قد لا تحتاجها أصلًا، وسيساعد على ضمان أن اختباراتك مفيدة.

أبقت المقاربة التكرارية كل خطوة صغيرة، وساعدتنا التغذية الراجعة المستمرة على كشف المتطلبات غير الواضحة ربما أسرع من مقاربات أخرى أكثر عشوائية.

### الكتابة؟

من المهم ملاحظة أن هذه الميزات الجديدة تحتوي على عمليات _قراءة_ الملفات فقط. وإذا كان عملك يحتاج إلى الكتابة، فستحتاج إلى البحث في مكان آخر. وتذكّر أن تواصل التفكير فيما توفّره المكتبة القياسية حاليًا؛ فإن كنت تكتب بيانات، يحسن أن تنظر في الاستفادة من واجهات موجودة مثل `io.Writer` ليبقى كودك متراخي الارتباط وقابلًا لإعادة الاستخدام.

### قراءات إضافية

* كانت هذه مقدمة خفيفة عن `io/fs`. وقد كتب [Ben Congdon شرحًا ممتازًا](https://benjamincongdon.me/blog/2021/01/21/A-Tour-of-Go-116s-iofs-package/) كان عونًا كبيرًا في كتابة هذا الفصل.
* [نقاش حول واجهات نظام الملفات](https://github.com/golang/go/issues/41190)
