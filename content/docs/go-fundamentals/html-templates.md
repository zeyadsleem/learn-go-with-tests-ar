---
title: القوالب (Templating)
weight: 190
---

# القوالب (Templating)

**[يمكنك العثور على كل كود هذا الفصل هنا](https://github.com/quii/learn-go-with-tests/tree/main/blogrenderer)**

نعيش في عالم يريد فيه الجميع بناء تطبيقات ويب بأحدث صيحة رائجة من إطارات عمل الواجهات الأمامية، مبنية فوق غيغابايتات من JavaScript المُحوَّلة (transpiled)، وتعمل مع نظام بناء بالغ التعقيد؛ [لكن ربما لا يكون ذلك ضروريًا دائمًا](https://quii.dev/The_Web_I_Want).

وأود القول إن معظم مطوري Go يقدّرون سلسلة أدوات بسيطة ومستقرة وسريعة، لكن عالم الواجهات الأمامية كثيرًا ما يخفق في تقديم ذلك.

كثيرة هي المواقع التي لا تحتاج إلى أن تكون [تطبيقات صفحة واحدة (SPA)](https://en.wikipedia.org/wiki/Single-page_application). **فـ HTML و CSS طريقتان رائعتان لتقديم المحتوى**، ويمكنك استخدام Go لبناء موقع يقدّم HTML.

وإذا كنت ترغب في أن يبقى لديك بعض العناصر الديناميكية، فيمكنك أن ترشّ قليلًا من JavaScript على جهة العميل، أو قد ترغب حتى في التجربة مع [Hotwire](https://hotwired.dev) الذي يتيح لك تقديم تجربة ديناميكية بمقاربة من جهة الخادم.

يمكنك توليد HTML في Go باستخدام مطوّل لـ [`fmt.Fprintf`](https://pkg.go.dev/fmt#Fprintf)، لكنك ستتعلم في هذا الفصل أن المكتبة القياسية في Go تضم بعض الأدوات التي تولّد HTML بطريقة أبسط وأسهل في الصيانة. وستتعلم أيضًا طرقًا أكثر فاعلية لاختبار هذا النوع من الكود ربما لم تصادفها من قبل.

## ما سنبنيه

في فصل [قراءة الملفات](reading-files.md) كتبنا كودًا يأخذ [`fs.FS`](https://pkg.go.dev/io/fs) (نظام ملفات)، ويعيد شريحة (slice) من `Post` لكل ملف markdown يصادفه.

```go
posts, err := blogposts.NewPostsFromFS(os.DirFS("posts"))
```

وهذه طريقة تعريفنا `Post`

```go
type Post struct {
	Title, Description, Body string
	Tags                     []string
}
```

وهذا مثال لأحد ملفات markdown التي يمكن تحليلها.

```markdown
Title: Welcome to my blog
Description: Introduction to my blog
Tags: cooking, family, live-laugh-love
---
# First recipe!
Welcome to my **amazing recipe blog**. I am going to write about my family recipes, and make sure I write a long, irrelevant and boring story about my family before you get to the actual instructions.
```

إذا واصلنا رحلتنا في كتابة برمجيات التدوين، فسنأخذ هذه البيانات ونولّد منها HTML ليُعيده خادم الويب استجابةً لطلبات HTTP.

نريد لمدونتنا أن تولّد نوعين من الصفحات:

1. **عرض المنشور**. يعرض منشورًا بعينه. والحقل `Body` في `Post` نصٌّ يحتوي markdown، لذا ينبغي تحويله إلى HTML.
2. **الفهرس**. يسرد جميع المنشورات، مع روابط تشعبية لعرض المنشور المحدد.

سنريد أيضًا مظهرًا وإحساسًا متناسقين في كل موقعنا، لذا سيكون لكل صفحة الهيكل المعتاد لـ HTML مثل `<html>` و`<head>` يحتوي روابط إلى أوراق أنماط CSS وأي شيء آخر قد نريده.

عندما تبني برمجيات تدوين، أمامك بضعة خيارات في المقاربة التي تتبعها لبناء HTML وإرساله إلى متصفح المستخدم.

سنصمم كودنا بحيث يقبل `io.Writer`. وهذا يعني أن مستدعي كودنا يملك مرونة أن:

- يكتبه إلى [os.File](https://pkg.go.dev/os#File)، فيُقدَّم تقديمًا ساكنًا (statically served)
- يكتب HTML مباشرةً إلى [`http.ResponseWriter`](https://pkg.go.dev/net/http#ResponseWriter)
- أو يكتبه إلى أي شيء فعليًا! فما دام الشيء ينفّذ `io.Writer`، يستطيع المستخدم توليد HTML من `Post`

## اكتب الاختبار أولًا

كما جرت العادة، من المهم التفكير في المتطلبات قبل الانغماس بسرعة كبيرة. فكيف يمكننا أخذ هذه المجموعة الكبيرة نسبيًا من المتطلبات وتقسيمها إلى خطوة صغيرة قابلة للتحقيق نركّز عليها؟

من وجهة نظري، عرض المحتوى فعليًا أولى بالأولوية من صفحة الفهرس. فيمكننا إطلاق هذا المنتج ومشاركة روابط مباشرة إلى محتوانا الرائع. أما صفحة فهرس لا تستطيع الربط بالمحتوى الفعلي فليست مفيدة.

ومع ذلك، يبدو عرض المنشور كما وصفناه سابقًا مهمة كبيرة. فكل هيكل HTML، وتحويل markdown الخاص بالمتن إلى HTML، وسرد الوسوم (tags)... إلخ.

في هذه المرحلة لست مهتمًا كثيرًا بالتوصيف (markup) المحدد، وخطوة أولى سهلة تكون بمجرد التحقق من أننا نستطيع عرض عنوان المنشور داخل `<h1>`. وهذه _تبدو_ أصغر خطوة أولى تدفعنا قليلًا إلى الأمام.

```go
package blogrenderer_test

import (
	"bytes"
	"github.com/quii/learn-go-with-tests/blogrenderer"
	"testing"
)

func TestRender(t *testing.T) {
	var (
		aPost = blogrenderer.Post{
			Title:       "hello world",
			Body:        "This is a post",
			Description: "This is a description",
			Tags:        []string{"go", "tdd"},
		}
	)

	t.Run("it converts a single post into HTML", func(t *testing.T) {
		buf := bytes.Buffer{}
		err := blogrenderer.Render(&buf, aPost)

		if err != nil {
			t.Fatal(err)
		}

		got := buf.String()
		want := `<h1>hello world</h1>`
		if got != want {
			t.Errorf("got '%s' want '%s'", got, want)
		}
	})
}
```

قرارنا بقبول `io.Writer` يجعل الاختبار بسيطًا أيضًا؛ فنحن هنا نكتب إلى [`bytes.Buffer`](https://pkg.go.dev/bytes#Buffer) يمكننا فحص محتواه لاحقًا.

## جرّب تشغيل الاختبار

إن كنت قد قرأت الفصول السابقة من هذا الكتاب، فمن المفترض أنك أصبحت متمرّسًا في هذه الخطوة الآن. لن تستطيع تشغيل الاختبار لأننا لم نعرّف الحزمة ولا الدالة `Render`. جرّب أن تتبع رسائل المترجم بنفسك وصولًا إلى حالة تستطيع فيها تشغيل الاختبار ورؤيته يفشل برسالة واضحة.

من المهم حقًا أن تتمرّن على رؤية اختباراتك تفشل؛ وستشكر نفسك بعد ستة أشهر عندما تُفشل اختبارًا عن غير قصد، لأنك بذلت الجهد _الآن_ للتحقق من أنه يفشل برسالة واضحة.

## اكتب أصغر قدر من الكود ليعمل الاختبار وافحص مخرجاته الفاشلة

هذا هو أصغر كود يجعل الاختبار يعمل

```go
package blogrenderer

// if you're continuing from the read files chapter, you shouldn't redefine this
type Post struct {
	Title, Description, Body string
	Tags                     []string
}

func Render(w io.Writer, p Post) error {
	return nil
}
```

من المفترض أن يشتكي الاختبار من أن النص الفارغ لا يساوي ما نريده.

## اكتب كودًا كافيًا لنجاح الاختبار

```go
func Render(w io.Writer, p Post) error {
	_, err := fmt.Fprintf(w, "<h1>%s</h1>", p.Title)
	return err
}
```

تذكّر أن تطوير البرمجيات نشاط تعلّمي في المقام الأول. ولكي نستكشف ونتعلم أثناء عملنا، نحتاج إلى العمل بطريقة تمنحنا حلقات تغذية راجعة متكررة وعالية الجودة، وأسهل سبيل إلى ذلك هو العمل بخطوات صغيرة.

لذا لا نقلق الآن بشأن استخدام أي مكتبات قوالب (templating). فيمكنك إنشاء HTML بتركيب النصوص "العادي" بلا مشكلة، وبتخطي جزء القوالب نستطيع التحقق من قدر صغير من السلوك المفيد، وقد أنجزنا قليلًا من العمل التصميمي لواجهة حزمتنا (API).

## إعادة الهيكلة

لا يوجد الكثير لنعيد هيكلته بعد، فلننتقل إلى التكرار التالي.

## اكتب الاختبار أولًا

بعد أن أصبح لدينا إصدار أساسي جدًا يعمل، يمكننا الآن تطوير الاختبار لتوسيع الوظيفة. وسنعرض في هذه الحالة معلومات أكثر من `Post`.

```go
	t.Run("it converts a single post into HTML", func(t *testing.T) {
		buf := bytes.Buffer{}
		err := blogrenderer.Render(&buf, aPost)

		if err != nil {
			t.Fatal(err)
		}

		got := buf.String()
		want := `<h1>hello world</h1>
<p>This is a description</p>
Tags: <ul><li>go</li><li>tdd</li></ul>`

		if got != want {
			t.Errorf("got '%s' want '%s'", got, want)
		}
	})
```

لاحظ أن كتابة هذا الاختبار _تبدو_ مرهقة. فرؤية كل هذا التوصيف (markup) داخل الاختبار شعور سيئ، ولم نُدرج بعد المتن (body) ولا HTML الفعلي الذي نريده مع كل محتوى `<head>` وكل هيكل الصفحة الذي نحتاجه.

ومع ذلك، فلنتحمّل هذا العناء _في الوقت الحالي_.

## جرّب تشغيل الاختبار

من المفترض أن يفشل، شاكيًا من أنه لا يحتوي على النص الذي نتوقعه، لأننا لا نعرض الوصف والوسوم.

## اكتب كودًا كافيًا لنجاح الاختبار

جرّب أن تفعل هذا بنفسك بدلًا من نسخ الكود. وستكتشف أن جعل هذا الاختبار ينجح _مزعج قليلًا_! فعندما جرّبت، ظهر الخطأ التالي في محاولتي الأولى

```
=== RUN   TestRender
=== RUN   TestRender/it_converts_a_single_post_into_HTML
    renderer_test.go:32: got '<h1>hello world</h1><p>This is a description</p><ul><li>go</li><li>tdd</li></ul>' want '<h1>hello world</h1>
        <p>This is a description</p>
        Tags: <ul><li>go</li><li></li></ul>'
```

أسطر جديدة! ومن يهتم؟ حسنًا، اختبارنا يهتم، لأنه يطابق قيمة نصية دقيقة. فهل ينبغي أن يفعل؟ لقد حذفت الأسطر الجديدة الآن فقط لجعل الاختبار ينجح.

```go
func Render(w io.Writer, p Post) error {
	_, err := fmt.Fprintf(w, "<h1>%s</h1>\n<p>%s</p>\n", p.Title, p.Description)
	if err != nil {
		return err
	}

	_, err = fmt.Fprint(w, "Tags: <ul>")
	if err != nil {
		return err
	}

	for _, tag := range p.Tags {
		_, err = fmt.Fprintf(w, "<li>%s</li>", tag)
		if err != nil {
			return err
		}
	}

	_, err = fmt.Fprint(w, "</ul>")
	if err != nil {
		return err
	}

	return nil
}
```

**يا للهول**. ليس أجمل كود كتبته، وما زلنا في مرحلة مبكرة جدًا من تنفيذ التوصيف. سنحتاج إلى محتوى وعناصر أكثر بكثير في صفحتنا، ونرى سريعًا أن هذه المقاربة غير مناسبة.

لكن الأهم أن لدينا اختبارًا ناجحًا؛ لدينا برمجية تعمل.

## إعادة الهيكلة

مع شبكة الأمان التي يمنحها اختبار ناجح لكود يعمل، يمكننا الآن التفكير في تغيير مقاربة التنفيذ في مرحلة إعادة الهيكلة.

### تقديم القوالب

تمتلك Go حزمتين للقوالب هما [text/template](https://pkg.go.dev/text/template) و[html/template](https://pkg.go.dev/html/template)، وتشتركان في الواجهة (interface) نفسها. وكلتاهما تتيحان لك دمج قالب مع بعض البيانات لإنتاج نص.

فما الفرق في نسخة HTML؟

> تنفّذ الحزمة template (html/template) قوالب مدفوعة بالبيانات لتوليد مخرجات HTML آمنة ضد حقن الكود (code injection). وهي توفّر الواجهة نفسها التي توفّرها الحزمة text/template، وينبغي استخدامها بدلًا من text/template كلما كانت المخرجات HTML.

لغة القوالب شبيهة جدًا بـ [Mustache](https://mustache.github.io) وتتيح لك توليد المحتوى ديناميكيًا بأسلوب نظيف جدًا مع فصل جميل للاهتمامات. ومقارنة بلغات القوالب الأخرى التي ربما استخدمتها، فهي مقيّدة جدًا أو "بلا منطق" (logic-less) كما يحب Mustache أن يقول. وهذا قرار تصميمي مهم **ومتعمَّد**.

وبينما نركّز هنا على توليد HTML، فإن كان مشروعك يجري عمليات معقّدة من دمج النصوص وتعويذاتها (incantations)، فقد ترغب في الاستعانة بـ `text/template` لتنظيف كودك.

### العودة إلى الكود

هذا قالب لمدونتنا:

`<h1>{{.Title}}</h1><p>{{.Description}}</p>Tags: <ul>{{range .Tags}}<li>{{.}}</li>{{end}}</ul>`

فأين نعرّف هذا النص؟ حسنًا، أمامنا بضعة خيارات، لكن لنُبقِ الخطوات صغيرة ولنبدأ بنص عادي بسيط

```go
package blogrenderer

import (
	"html/template"
	"io"
)

const (
	postTemplate = `<h1>{{.Title}}</h1><p>{{.Description}}</p>Tags: <ul>{{range .Tags}}<li>{{.}}</li>{{end}}</ul>`
)

func Render(w io.Writer, p Post) error {
	templ, err := template.New("blog").Parse(postTemplate)
	if err != nil {
		return err
	}

	return templ.Execute(w, p)
}
```

ننشئ قالبًا جديدًا باسم معيّن، ثم نحلّل نص القالب. بعد ذلك يمكننا استخدام الـ method `Execute` عليه، ونمرّر بياناتنا، وهي هنا `Post`.

سيستبدل القالب أشياء مثل `{{.Description}}` بمحتوى `p.Description`. وتمنحك القوالب أيضًا بعض البدائيات البرمجية مثل `range` للمرور على القيم، و`if`. ويمكنك العثور على تفاصيل أكثر في [توثيق text/template](https://pkg.go.dev/text/template).

_ينبغي أن تكون هذه إعادة هيكلة خالصة._ فلا حاجة لتغيير اختباراتنا، ومن المفترض أن تستمر في النجاح. والأهم أن كودنا أصبح أسهل قراءةً، ومعالجة الأخطاء المزعجة صارت أقل بكثير.

كثيرًا ما يشتكي الناس من إسهاب معالجة الأخطاء في Go، لكنك قد تكتشف طرقًا أفضل لكتابة كودك بحيث يكون أقل عرضة للأخطاء من الأساس، كما هنا.

### المزيد من إعادة الهيكلة

كان استخدام `html/template` تحسينًا بلا شك، لكن وجوده كثابت نصي في كودنا ليس جيدًا:

- لا يزال من الصعب قراءته.
- ليس صديقًا لبيئات التطوير والمحررات. فلا تظليل للصياغة (syntax highlighting)، ولا إمكانية لإعادة التنسيق أو إعادة الهيكلة... إلخ.
- يبدو مثل HTML، لكنك لا تستطيع التعامل معه فعليًا كما تفعل مع ملف HTML "عادي"

ما نريده هو أن تعيش قوالبنا في ملفات منفصلة لننظّمها بشكل أفضل، ونتعامل معها كما لو كانت ملفات HTML.

أنشئ مجلدًا اسمه "templates" وضع داخله ملفًا اسمه `blog.gohtml`، والصق قالبنا فيه.

غيّر الآن كودنا لتضمين أنظمة الملفات (embed) باستخدام [وظيفة التضمين المدمجة في Go 1.16](https://pkg.go.dev/embed).

```go
package blogrenderer

import (
	"embed"
	"html/template"
	"io"
)

var (
	//go:embed "templates/*"
	postTemplates embed.FS
)

func Render(w io.Writer, p Post) error {
	templ, err := template.ParseFS(postTemplates, "templates/*.gohtml")
	if err != nil {
		return err
	}

	return templ.Execute(w, p)
}
```

بتضمين "نظام ملفات" داخل كودنا، نستطيع تحميل قوالب متعددة ودمجها بحرية. وسيكون هذا مفيدًا عندما نريد مشاركة منطق العرض بين قوالب مختلفة، مثل رأس (header) أعلى صفحة HTML وتذييل (footer).

### التضمين؟

مرّ التضمين سريعًا في فصل [قراءة الملفات](reading-files.md). ويشرح [توثيق المكتبة القياسية](https://pkg.go.dev/embed) قائلًا

> توفر الحزمة embed الوصول إلى الملفات المضمَّنة في برنامج Go أثناء تشغيله.
>
> ويمكن لملفات مصدر Go التي تستورد "embed" استخدام توجيه //go:embed لتهيئة متغير من النوع string أو []byte أو FS بمحتويات ملفات تُقرأ من مجلد الحزمة أو مجلداته الفرعية وقت الترجمة.

فلماذا نريد استخدام هذا؟ حسنًا، البديل هو أننا _نستطيع_ تحميل قوالبنا من نظام ملفات "عادي". لكن هذا يعني أن علينا التأكد من أن القوالب في المسار الصحيح أينما أردنا استخدام هذه البرمجية. وقد يكون لديك في عملك بيئات متعددة مثل التطوير والاختبار المبدئي (staging) والإنتاج (live). ولكي يعمل هذا، ستحتاج إلى التأكد من نسخ قوالبك إلى المكان الصحيح.

مع التضمين، تُدرَج الملفات في برنامجك عندما تبنيه. وهذا يعني أنك بمجرد أن تبني برنامجك (وهو ما ينبغي أن تفعله مرة واحدة فقط)، تظل الملفات متاحة لك دائمًا.

واللطيف أنك لا تستطيع تضمين ملفات فردية فحسب، بل أنظمة ملفات أيضًا؛ ونظام الملفات هذا ينفّذ [io/fs](https://pkg.go.dev/io/fs)، ما يعني أن كودك لا يحتاج إلى الاهتمام بنوع نظام الملفات الذي يتعامل معه.

لكن إن أردت استخدام قوالب مختلفة حسب الإعدادات، فقد تفضّل الالتزام بتحميل القوالب من القرص بالطريقة التقليدية الأكثر شيوعًا.

## الخطوة التالية: اجعل القالب "أنيقًا"

لا نريد فعلًا أن يكون قالبنا معرّفًا كنص من سطر واحد. فنحن نريد توزيعه على أسطر ليصبح أسهل قراءةً وتعاملًا، هكذا:

```handlebars
<h1>{{.Title}}</h1>

<p>{{.Description}}</p>

Tags: <ul>{{range .Tags}}<li>{{.}}</li>{{end}}</ul>
```

لكن إن فعلنا ذلك، يفشل اختبارنا. والسبب أن اختبارنا يتوقع إرجاع نص دقيق جدًا.

لكننا في الحقيقة لا نهتم فعليًا بالمسافات البيضاء (whitespace). وستصبح صيانة هذا الاختبار كابوسًا إذا اضطررنا إلى تحديث نص التحقق (assertion) يدويًا بجهد ومشقة كلما أجرينا تغييرات طفيفة على التوصيف. وكلما كبر القالب، صارت إدارة هذا النوع من التعديلات أصعب، وخرجت تكاليف العمل عن السيطرة.

## تقديم اختبارات الموافقة (Approval Tests)

[Go Approval Tests](https://github.com/approvals/go-approval-tests)

> تتيح ApprovalTests اختبارًا سهلًا لكائنات أكبر، ونصوص، وأي شيء آخر يمكن حفظه في ملف (صور، أصوات، CSV... إلخ)

الفكرة شبيهة بملفات "golden" أو اختبار اللقطات (snapshot testing). فبدلًا من صيانة النصوص داخل ملف الاختبار على نحو مرهق، تستطيع أداة الموافقة أن تقارن المخرجات نيابةً عنك بملف "معتمد" أنشأته أنت. ثم تنسخ ببساطة الإصدار الجديد إن وافقت عليه. أعد تشغيل الاختبار وستعود إلى اللون الأخضر.

أضف اعتمادية (dependency) على `"github.com/approvals/go-approval-tests"` (بالأمر `go get github.com/approvals/go-approval-tests`) إلى مشروعك، وعدّل الاختبار إلى ما يلي

```go
func TestRender(t *testing.T) {
	var (
		aPost = blogrenderer.Post{
			Title:       "hello world",
			Body:        "This is a post",
			Description: "This is a description",
			Tags:        []string{"go", "tdd"},
		}
	)

	t.Run("it converts a single post into HTML", func(t *testing.T) {
		buf := bytes.Buffer{}

		if err := blogrenderer.Render(&buf, aPost); err != nil {
			t.Fatal(err)
		}

		approvals.VerifyString(t, buf.String())
	})
}
```

في المرة الأولى التي تشغّله فيها سيفشل، لأننا لم نوافق على شيء بعد

```
=== RUN   TestRender
=== RUN   TestRender/it_converts_a_single_post_into_HTML
    renderer_test.go:29: Failed Approval: received does not match approved.
```

وسيكون قد أنشأ ملفين بالشكل التالي

- `renderer_test.TestRender.it_converts_a_single_post_into_HTML.received.txt`
- `renderer_test.TestRender.it_converts_a_single_post_into_HTML.approved.txt`

يحتوي ملف received على الإصدار الجديد غير المعتمد من المخرجات. انسخه إلى ملف approved الفارغ وأعد تشغيل الاختبار.

وبنسخك الإصدار الجديد تكون قد "اعتمدت" التغيير، ويمر الاختبار الآن.

ولترى سير العمل ماثلًا أمامك، عدّل القالب كما تحدثنا ليصبح أسهل قراءة (لكنه دلاليًا هو نفسه).

```handlebars
<h1>{{.Title}}</h1>

<p>{{.Description}}</p>

Tags: <ul>{{range .Tags}}<li>{{.}}</li>{{end}}</ul>
```

أعد تشغيل الاختبار. سيُنشأ ملف "received" جديد لأن مخرجات كودنا تختلف عن الإصدار المعتمد. ألقِ نظرة عليهما، وإن أعجبتك التغييرات فانسخ الإصدار الجديد ببساطة وأعد تشغيل الاختبار. واحرص على تسجيل ملفات approved في نظام التحكم بالمصادر.

تجعل هذه المقاربة إدارة التغييرات في أشياء كبيرة وقبيحة مثل HTML أبسط بكثير. فيمكنك استخدام أداة diff لعرض الفروق وإدارتها، وهي تُبقي كود اختبارك أنظف.

![استخدم أداة diff لإدارة التغييرات](https://i.imgur.com/0MoNdva.png)

هذا في الواقع استخدام بسيط نسبيًا لاختبارات الموافقة، وهي أداة مفيدة للغاية في ترسانة اختباراتك. ولدى [Emily Bache](https://twitter.com/emilybache) [فيديو ممتع تستخدم فيه اختبارات الموافقة لإضافة مجموعة واسعة بشكل مذهل من الاختبارات إلى قاعدة كود معقدة بلا أي اختبارات](https://www.youtube.com/watch?v=zyM2Ep28ED8). و"الاختبار التوافقي" (Combinatorial Testing) شيء يستحق النظر فيه بلا شك.

الآن بعد أن أجرينا هذا التغيير، ما زلنا نستفيد من كون كودنا مختبَرًا جيدًا، لكن الاختبارات لن تعترض طريقنا كثيرًا عندما نعبث بالتوصيف.

### هل ما زلنا نمارس TDD؟

ثمة أثر جانبي مثير لهذه المقاربة، وهو أنها تُبعدنا عن TDD. بالطبع _يمكنك_ تحرير ملفات approved يدويًا إلى الحالة التي تريدها، ثم تشغيل اختباراتك، ثم إصلاح القوالب لتُخرِج ما عرّفته.

لكن هذا سخيف! فـ TDD طريقة للعمل، وتحديدًا للتصميم؛ لكن هذا لا يعني أن علينا استخدامه بشكل دوغمائي في **كل شيء**.

المهم أننا فعلنا الصواب واستخدمنا TDD كـ**أداة تصميم** لتصميم واجهة حزمتنا. أما تغييرات القوالب فيمكن أن تكون عمليتنا فيها:

- أجرِ تغييرًا صغيرًا على القالب
- شغّل اختبار الموافقة
- افحص المخرجات بالعين لتتأكد من أنها صحيحة
- اعتمد التغيير
- كرّر

ولا ينبغي أن نتخلى عن قيمة العمل بخطوات صغيرة قابلة للتحقيق. حاول أن تجد طرقًا تجعل التغييرات صغيرة، وواصل إعادة تشغيل الاختبارات لتحصل على تغذية راجعة حقيقية حول ما تفعله.

وإذا بدأنا نفعل أشياء مثل تغيير الكود _المحيط_ بالقوالب، فقد يستدعي ذلك بالطبع العودة إلى طريقة عملنا المعتادة مع TDD.

## توسيع التوصيف

معظم المواقع تملك HTML أغنى مما لدينا الآن. فبدايةً، عنصر `html` مع `head`، وربما بعض `nav` أيضًا. وعادةً توجد فكرة للتذييل (footer) كذلك.

إن كان موقعنا سيضم صفحات مختلفة، فسنريد تعريف هذه الأشياء في مكان واحد ليبقى مظهره متناسقًا. وتدعمنا قوالب Go في تعريف أقسام يمكننا بعد ذلك استيرادها إلى قوالب أخرى.

عدّل قالبنا الحالي ليستورد قالبًا علويًا وقالبًا سفليًا

```handlebars
{{template "top" .}}
<h1>{{.Title}}</h1>

<p>{{.Description}}</p>

Tags: <ul>{{range .Tags}}<li>{{.}}</li>{{end}}</ul>
{{template "bottom" .}}
```

ثم أنشئ `top.gohtml` بالمحتوى التالي

```handlebars
{{define "top"}}
<!DOCTYPE html>
<html lang="en">
<head>
    <title>My amazing blog!</title>
    <meta charset="UTF-8"/>
    <meta name="description" content="Wow, like and subscribe, it really helps the channel guys" lang="en"/>
</head>
<body>
<nav role="navigation">
    <div>
        <h1>Budding Gopher's blog</h1>
        <ul>
            <li><a href="/">home</a></li>
            <li><a href="about">about</a></li>
            <li><a href="archive">archive</a></li>
        </ul>
    </div>
</nav>
<main>
{{end}}
```

و`bottom.gohtml`

```handlebars
{{define "bottom"}}
</main>
<footer>
    <ul>
        <li><a href="https://twitter.com/quii">Twitter</a></li>
        <li><a href="https://github.com/quii">GitHub</a></li>
    </ul>
</footer>
</body>
</html>
{{end}}
```

(ومن الواضح أن لك الحرية في وضع أي توصيف يعجبك!)

نحتاج الآن إلى تحديد قالب معيّن لتشغيله. في عارض المدونة، غيّر أمر `Execute` إلى `ExecuteTemplate`

```go
if err := templ.ExecuteTemplate(w, "blog.gohtml", p); err != nil {
	return err
}
```

أعد تشغيل اختبارك. من المفترض أن يُنشأ ملف "received" جديد ويفشل الاختبار. افحصه، وإن أعجبتك النتيجة فاعتمده بنسخه فوق الإصدار القديم. ثم أعد تشغيل الاختبار وينبغي أن ينجح.

## عذر للعب بقياس الأداء (Benchmarking)

قبل أن نمضي قدمًا، لنفكر فيما يفعله كودنا.

```go
func Render(w io.Writer, p Post) error {
	templ, err := template.ParseFS(postTemplates, "templates/*.gohtml")
	if err != nil {
		return err
	}

	return templ.ExecuteTemplate(w, "blog.gohtml", p)
}
```

- يحلّل القوالب
- يستخدم القالب لعرض منشور إلى `io.Writer`

وبينما سيكون أثر إعادة تحليل القوالب عند كل منشور ضئيلًا نسبيًا في معظم الحالات، فإن الجهد المبذول في *عدم* فعل ذلك ضئيل أيضًا، وسيجعل الكود أنظف قليلًا.

ولرؤية أثر عدم تكرار هذا التحليل مرارًا، يمكننا استخدام أداة قياس الأداء لنرى مدى سرعة دالتنا.

```go
func BenchmarkRender(b *testing.B) {
	var (
		aPost = blogrenderer.Post{
			Title:       "hello world",
			Body:        "This is a post",
			Description: "This is a description",
			Tags:        []string{"go", "tdd"},
		}
	)

	for b.Loop() {
		blogrenderer.Render(io.Discard, aPost)
	}
}
```

وهذه هي النتائج على حاسوبي

```
BenchmarkRender-8 22124 53812 ns/op
```

ولكي نتوقف عن إعادة تحليل القوالب مرارًا وتكرارًا، سننشئ نوعًا يحتفظ بالقالب المحلَّل، وله method تقوم بالعرض

```go
type PostRenderer struct {
	templ *template.Template
}

func NewPostRenderer() (*PostRenderer, error) {
	templ, err := template.ParseFS(postTemplates, "templates/*.gohtml")
	if err != nil {
		return nil, err
	}

	return &PostRenderer{templ: templ}, nil
}

func (r *PostRenderer) Render(w io.Writer, p Post) error {
	return r.templ.ExecuteTemplate(w, "blog.gohtml", p)
}
```

هذا يغيّر واجهة كودنا فعلًا، لذا سنحتاج إلى تحديث اختبارنا

```go
func TestRender(t *testing.T) {
	var (
		aPost = blogrenderer.Post{
			Title:       "hello world",
			Body:        "This is a post",
			Description: "This is a description",
			Tags:        []string{"go", "tdd"},
		}
	)

	postRenderer, err := blogrenderer.NewPostRenderer()

	if err != nil {
		t.Fatal(err)
	}

	t.Run("it converts a single post into HTML", func(t *testing.T) {
		buf := bytes.Buffer{}

		if err := postRenderer.Render(&buf, aPost); err != nil {
			t.Fatal(err)
		}

		approvals.VerifyString(t, buf.String())
	})
}
```

وقياس الأداء لدينا

```go
func BenchmarkRender(b *testing.B) {
	var (
		aPost = blogrenderer.Post{
			Title:       "hello world",
			Body:        "This is a post",
			Description: "This is a description",
			Tags:        []string{"go", "tdd"},
		}
	)

	postRenderer, err := blogrenderer.NewPostRenderer()

	if err != nil {
		b.Fatal(err)
	}

	for b.Loop() {
		postRenderer.Render(io.Discard, aPost)
	}
}
```

من المفترض أن يستمر الاختبار في النجاح. فماذا عن قياس الأداء؟

`BenchmarkRender-8 362124 3131 ns/op`. كانت قيمة ns/op القديمة `53812 ns/op`، فهذا تحسين جيد! وعندما نضيف methods أخرى للعرض، مثل صفحة فهرس، سيبسّط ذلك الكود لأننا لن نحتاج إلى تكرار تحليل القوالب.

## العودة إلى العمل الحقيقي

في ما يخص عرض المنشورات، الجزء المهم المتبقي هو عرض `Body` فعليًا. فكما تتذكر، من المفترض أن يكون markdown كتبه المؤلف، لذا سيحتاج إلى تحويل إلى HTML.

وسنترك هذا تمرينًا لك أنت يا قارئنا العزيز. من المفترض أن تجد مكتبة Go تفعل ذلك عنك. واستخدم اختبار الموافقة للتحقق من صحة ما تفعله.

### عن اختبار مكتبات الطرف الثالث

**ملاحظة**. احترس من أن تقلق أكثر من اللازم بشأن اختبار سلوك مكتبة طرف ثالث اختبارًا صريحًا في اختبارات الوحدة.

فكتابة اختبارات ضد كود لا تتحكم فيه مضيعة للجهد وتضيف عبئًا على الصيانة. وأحيانًا قد ترغب في استخدام [حقن الاعتماديات](./dependency-injection.md) للتحكم في اعتمادية ما ومحاكاة سلوكها بـ mock من أجل اختبار.

لكنني في هذه الحالة أرى أن تحويل markdown إلى HTML تفصيل تنفيذي من تفاصيل العرض، ويكفي أن تمنحنا اختبارات الموافقة ثقة كافية.

### عرض الفهرس

الوظيفة التالية التي سننجزها هي عرض فهرس يسرد المنشورات في قائمة مرتبة (ordered list) في HTML.

نحن نوسّع واجهتنا، لذا سنعيد وضع قبعة TDD.

## اكتب الاختبار أولًا

تبدو صفحة الفهرس بسيطة في ظاهرها، لكن كتابة الاختبار لا تزال تدفعنا إلى اتخاذ بعض قرارات التصميم

```go
t.Run("it renders an index of posts", func(t *testing.T) {
	buf := bytes.Buffer{}
	posts := []blogrenderer.Post{{Title: "Hello World"}, {Title: "Hello World 2"}}

	if err := postRenderer.RenderIndex(&buf, posts); err != nil {
		t.Fatal(err)
	}

	got := buf.String()
	want := `<ol><li><a href="/post/hello-world">Hello World</a></li><li><a href="/post/hello-world-2">Hello World 2</a></li></ol>`

	if got != want {
		t.Errorf("got %q want %q", got, want)
	}
})
```

1. نستخدم حقل عنوان `Post` كجزء من مسار الرابط (URL)، لكننا لا نريد مسافات في الرابط، لذا نستبدلها بشرطات.
2. أضفنا method اسمها `RenderIndex` إلى `PostRenderer`، وتأخذ هي أيضًا `io.Writer` وشريحة من `Post`.

لو التزمنا هنا بمقاربة الاختبار بعد الكتابة مع اختبارات الموافقة، لما أجبنا عن هذه الأسئلة في بيئة مضبوطة. **فالاختبارات تمنحنا مساحة للتفكير**.

## جرّب تشغيل الاختبار

```
./renderer_test.go:41:13: undefined: blogrenderer.RenderIndex
```

## اكتب أصغر قدر من الكود ليعمل الاختبار وافحص مخرجاته الفاشلة

```go
func (r *PostRenderer) RenderIndex(w io.Writer, posts []Post) error {
	return nil
}
```

من المفترض أن يُنتج ما سبق فشل الاختبار التالي

```
=== RUN   TestRender
=== RUN   TestRender/it_renders_an_index_of_posts
    renderer_test.go:49: got "" want "<ol><li><a href=\"/post/hello-world\">Hello World</a></li><li><a href=\"/post/hello-world-2\">Hello World 2</a></li></ol>"
--- FAIL: TestRender (0.00s)
```

## اكتب كودًا كافيًا لنجاح الاختبار

رغم أن هذا _يبدو_ كأنه ينبغي أن يكون سهلًا، فهو مزعج قليلًا. لقد فعلته على عدة خطوات

```go
func (r *PostRenderer) RenderIndex(w io.Writer, posts []Post) error {
	indexTemplate := `<ol>{{range .}}<li><a href="/post/{{.Title}}">{{.Title}}</a></li>{{end}}</ol>`

	templ, err := template.New("index").Parse(indexTemplate)
	if err != nil {
		return err
	}

	if err := templ.Execute(w, posts); err != nil {
		return err
	}

	return nil
}
```

لم أرد في البداية أن أُرهق نفسي بملفات قوالب منفصلة، أردت فقط أن يعمل. وأنا أرى تحليل القوالب وفصلها مقدمًا إعادة هيكلة يمكنني القيام بها لاحقًا.

هذا لا ينجح، لكنه قريب.

```
=== RUN   TestRender
=== RUN   TestRender/it_renders_an_index_of_posts
    renderer_test.go:49: got "<ol><li><a href=\"/post/Hello%20World\">Hello World</a></li><li><a href=\"/post/Hello%20World%202\">Hello World 2</a></li></ol>" want "<ol><li><a href=\"/post/hello-world\">Hello World</a></li><li><a href=\"/post/hello-world-2\">Hello World 2</a></li></ol>"
--- FAIL: TestRender (0.00s)
    --- FAIL: TestRender/it_renders_an_index_of_posts (0.00s)
```

يمكنك أن ترى أن كود القوالب يهرّب (escape) المسافات في خصائص `href`. نحتاج إلى طريقة لاستبدال المسافات بشرطات في النص. ولا يمكننا ببساطة المرور على `[]Post` واستبدالها في الذاكرة، لأننا ما زلنا نريد عرض المسافات للمستخدم في الروابط.

أمامنا بضعة خيارات. وأولها سنستكشفه هو تمرير دالة إلى قالبنا.

### تمرير الدوال إلى القوالب

```go
func (r *PostRenderer) RenderIndex(w io.Writer, posts []Post) error {
	indexTemplate := `<ol>{{range .}}<li><a href="/post/{{sanitiseTitle .Title}}">{{.Title}}</a></li>{{end}}</ol>`

	templ, err := template.New("index").Funcs(template.FuncMap{
		"sanitiseTitle": func(title string) string {
			return strings.ToLower(strings.Replace(title, " ", "-", -1))
		},
	}).Parse(indexTemplate)
	if err != nil {
		return err
	}

	if err := templ.Execute(w, posts); err != nil {
		return err
	}

	return nil
}
```

_قبل أن تحلّل قالبًا_ يمكنك إضافة `template.FuncMap` إلى قالبك، وهو يتيح لك تعريف دوال يمكن استدعاؤها داخل قالبك. وفي هذه الحالة أنشأنا دالة `sanitiseTitle` نستدعيها بعد ذلك داخل قالبنا بـ `{{sanitiseTitle .Title}}`.

هذه ميزة قوية؛ فإرسال دوال إلى قالبك سيتيح لك فعل أشياء رائعة جدًا، لكن هل ينبغي لك ذلك؟ بالعودة إلى مبادئ Mustache والقوالب بلا منطق، لماذا دعوا إلى خلوّها من المنطق؟ **وما الخطأ في وجود منطق داخل القوالب؟**

وكما أظهرنا، فلكي نختبر قوالبنا، _اضطررنا إلى تقديم نوع مختلف تمامًا من الاختبار_.

تخيّل أنك أدخلت إلى القالب دالة لها بضع حالات سلوك مختلفة وحالات حدّية (edge cases)، **فكيف ستختبرها**؟ مع هذا التصميم الحالي، وسيلتك الوحيدة لاختبار هذا المنطق هي _عرض HTML ومقارنة النصوص_. وهذه ليست طريقة سهلة ولا سليمة لاختبار المنطق، وبالتأكيد ليست ما تريده لمنطق أعمال _مهم_.

ورغم أن تقنية اختبارات الموافقة خفّضت كلفة صيانة هذه الاختبارات، فهي لا تزال أغلى صيانةً من معظم اختبارات الوحدة التي ستكتبها. وما زالت حساسة لأي تغيير طفيف في التوصيف، لكننا جعلنا إدارتها أسهل. وعلينا مع ذلك أن نجتهد في هندسة كودنا بحيث لا نضطر إلى كتابة الكثير من الاختبارات حول قوالبنا، وأن نفصل الاهتمامات فصلًا سليمًا حتى لا يبقى داخل كود العرض أي منطق لا حاجة له فيه.

ما تمنحك إياه محركات القوالب المتأثرة بـ Mustache هو قيد مفيد، فلا تحاول التحايل عليه كثيرًا؛ **ولا تسر ضد التيار**. بل تبنَّ فكرة [نماذج العرض (view models)](https://stackoverflow.com/a/11074506/3193)، حيث تبني أنواعًا محددة تحتوي البيانات التي تحتاج إلى عرضها، بطريقة ملائمة للغة القوالب.

وبهذه الطريقة، يمكن اختبار أي منطق أعمال مهم تستخدمه لتوليد حزمة البيانات تلك اختبارًا منفصلًا، بعيدًا عن عالم HTML والقوالب الفوضوي.

### فصل الاهتمامات

فما الذي يمكننا فعله بدلًا من ذلك؟

#### إضافة method إلى `Post` ثم استدعاؤها في القالب

يمكننا استدعاء methods في كود القوالب على الأنواع التي نرسلها، فيمكننا إذن إضافة method اسمها `SanitisedTitle` إلى `Post`. وهذا سيبسّط القالب وسيتيح لنا اختبار هذا المنطق بسهولة على حدة إن أردنا. وربما يكون هذا أسهل حل، وإن لم يكن بالضرورة الأبسط.

ومن مساوئ هذه المقاربة أنها لا تزال منطق _عرض_. فهي لا تهم بقية النظام، لكنها أصبحت الآن جزءًا من واجهة كائن مجال (domain object) أساسي. وهذا النوع من المقاربات قد يؤدي بمرور الوقت إلى إنشاء [كائنات إلهية (God Objects)](https://en.wikipedia.org/wiki/God_object).

#### إنشاء نوع نموذج عرض مخصص، مثل `PostViewModel`، يحتوي بالضبط البيانات التي نحتاجها

فبدلًا من أن يكون كود العرض مقترنًا بكائن المجال `Post`، فإنه يأخذ نموذج عرض بدلًا منه.

```go
type PostViewModel struct {
	Title, SanitisedTitle, Description, Body string
	Tags                                     []string
}
```

وسيضطر مستدعو كودنا إلى التحويل من `[]Post` إلى `[]PostView`، مولّدين `SanitizedTitle`. وطريقة لإبقاء هذا نظيفًا أن تكون لدينا دالة `func NewPostView(p Post) PostView` تغلّف هذا التحويل.

وهذا سيُبقي كود العرض بلا منطق، وهو على الأرجح أشد فصل للاهتمامات يمكننا تحقيقه، لكن المقابل هو عملية أكثر تعقيدًا قليلًا لعرض منشوراتنا.

كلا الخيارين جيد، وأنا في هذه الحالة أَميل إلى الأول. وعندما يتطور نظامك، ينبغي أن تحترس من إضافة المزيد والمزيد من الـ methods المرتجلة فقط لتيسير عملية العرض؛ فتصبح نماذج العرض المخصصة أكثر فائدة عندما يصبح التحويل بين كائن المجال والعرض أكثر تشابكًا.

فيمكننا إضافة هذه الـ method إلى `Post`

```go
func (p Post) SanitisedTitle() string {
	return strings.ToLower(strings.Replace(p.Title, " ", "-", -1))
}
```

ثم يمكننا العودة إلى عالم أبسط في كود العرض

```go
func (r *PostRenderer) RenderIndex(w io.Writer, posts []Post) error {
	indexTemplate := `<ol>{{range .}}<li><a href="/post/{{.SanitisedTitle}}">{{.Title}}</a></li>{{end}}</ol>`

	templ, err := template.New("index").Parse(indexTemplate)
	if err != nil {
		return err
	}

	if err := templ.Execute(w, posts); err != nil {
		return err
	}

	return nil
}
```

## إعادة الهيكلة

أخيرًا من المفترض أن يكون الاختبار ناجحًا. ويمكننا الآن نقل قالبنا إلى ملف (`templates/index.gohtml`) وتحميله مرة واحدة عند إنشاء العارض.

```go
package blogrenderer

import (
	"embed"
	"html/template"
	"io"
)

var (
	//go:embed "templates/*"
	postTemplates embed.FS
)

type PostRenderer struct {
	templ *template.Template
}

func NewPostRenderer() (*PostRenderer, error) {
	templ, err := template.ParseFS(postTemplates, "templates/*.gohtml")
	if err != nil {
		return nil, err
	}

	return &PostRenderer{templ: templ}, nil
}

func (r *PostRenderer) Render(w io.Writer, p Post) error {
	return r.templ.ExecuteTemplate(w, "blog.gohtml", p)
}

func (r *PostRenderer) RenderIndex(w io.Writer, posts []Post) error {
	return r.templ.ExecuteTemplate(w, "index.gohtml", posts)
}
```

بتحليل أكثر من قالب واحد إلى `templ`، صرنا مضطرين الآن إلى استدعاء `ExecuteTemplate` وتحديد _أي_ قالب نريد عرضه حسب الحاجة، لكن آمل أن توافقني أن الكود الذي وصلنا إليه يبدو رائعًا.

ثمة خطر _طفيف_ إذا أعاد أحدهم تسمية أحد ملفات القوالب، إذ سيُدخل ذلك خطأً، لكن اختبارات الوحدة السريعة سترصد هذا بسرعة.

والآن وقد رضينا عن تصميم واجهة حزمتنا واستخرجنا بعض السلوك الأساسي بـ TDD، لنغيّر اختبارنا ليستخدم الموافقات.

```go
	t.Run("it renders an index of posts", func(t *testing.T) {
		buf := bytes.Buffer{}
		posts := []blogrenderer.Post{{Title: "Hello World"}, {Title: "Hello World 2"}}

		if err := postRenderer.RenderIndex(&buf, posts); err != nil {
			t.Fatal(err)
		}

		approvals.VerifyString(t, buf.String())
	})
```

تذكّر أن تشغّل الاختبار لتراه يفشل، ثم اعتمد التغيير.

وأخيرًا يمكننا إضافة هيكل الصفحة إلى صفحة الفهرس:

```handlebars
{{template "top" .}}
<ol>{{range .}}<li><a href="/post/{{.SanitisedTitle}}">{{.Title}}</a></li>{{end}}</ol>
{{template "bottom" .}}
```

أعد تشغيل الاختبار، واعتمد التغيير، ونكون قد انتهينا من الفهرس!

## عرض متن markdown

شجعتك على تجربته بنفسك، وهذا هو الأسلوب الذي انتهيت إليه.

```go
package blogrenderer

import (
	"embed"
	"github.com/gomarkdown/markdown"
	"github.com/gomarkdown/markdown/parser"
	"html/template"
	"io"
)

var (
	//go:embed "templates/*"
	postTemplates embed.FS
)

type PostRenderer struct {
	templ *template.Template
}

func NewPostRenderer() (*PostRenderer, error) {
	templ, err := template.ParseFS(postTemplates, "templates/*.gohtml")
	if err != nil {
		return nil, err
	}

	return &PostRenderer{templ: templ}, nil
}

func (r *PostRenderer) Render(w io.Writer, p Post) error {
	return r.templ.ExecuteTemplate(w, "blog.gohtml", newPostVM(p))
}

func (r *PostRenderer) RenderIndex(w io.Writer, posts []Post) error {
	return r.templ.ExecuteTemplate(w, "index.gohtml", posts)
}

type postViewModel struct {
	Post
	HTMLBody template.HTML
}

func newPostVM(p Post) postViewModel {
	vm := postViewModel{Post: p}
	extensions := parser.CommonExtensions | parser.AutoHeadingIDs
	mdParser := parser.NewWithExtensions(extensions)
	vm.HTMLBody = template.HTML(markdown.ToHTML([]byte(p.Body), mdParser, nil))
	return vm
}
```

استخدمت مكتبة [gomarkdown](https://github.com/gomarkdown/markdown) الممتازة، وقد عملت تمامًا كما كنت آمل.

لاحظ أننا ننشئ `parser.Parser` جديدًا عند كل استدعاء لـ `newPostVM`، بدلًا من إنشاء واحد مرة واحدة وتخزينه على `PostRenderer`. فعادةً، إن استطعت إنشاء شيء مرة واحدة وإعادة استخدامه، فذلك يستحق العناء لأنه يوفّر كلفة الإنشاء المتكرر. لكن هذه الغريزة العامة تنهار هنا: فمحلل gomarkdown يحتفظ بحالة داخلية أثناء بناء شجرة المستند، وليس آمنًا لإعادة استخدامه في عدة استدعاءات لـ `Parse`. وستحصل على panic (أو، في الإصدارات الأقدم، على إلغاء الإشارة إلى مؤشر فارغ (nil pointer dereference) أقل فائدة بكثير) في المرة الثانية التي تستخدم فيها المحلل نفسه. وبما أن `PostRenderer` مقصود منه عرض منشورات كثيرة على مدار عمره، فإن إنشاء محلل جديد عند كل استدعاء هو الطريقة الصحيحة لاستخدامه، وهو زهيد على أي حال لأن إنشاءه بسيط مقارنة بعمل التحليل نفسه.

إن كنت قد جرّبت فعل هذا بنفسك، فقد تكون لاحظت أن عرض المتن كان يُخرج HTML مهرَّبة (escaped). وهذه ميزة أمنية في حزمة html/template في Go لمنع إخراج HTML خبيث من طرف ثالث.

وللتحايل على ذلك، ستحتاج في النوع الذي ترسله إلى العرض إلى تغليف HTML الموثوق به في [template.HTML](https://pkg.go.dev/html/template#HTML)

> يغلّف HTML جزءًا من مستند HTML معروف أنه آمن. ولا ينبغي استخدامه مع HTML قادم من طرف ثالث، أو HTML يحتوي وسومًا أو تعليقات غير مغلقة. فمخرجات أدوات تعقيم HTML (sanitiser) السليمة وقالب مُهرَّب بواسطة هذه الحزمة مناسبان للاستخدام مع HTML.
>
> ويشكّل استخدام هذا النوع خطرًا أمنيًا: إذ ينبغي أن يأتي المحتوى المغلَّف من مصدر موثوق، لأنه سيُدرَج حرفيًا في مخرجات القالب.

لذا أنشأت نموذج عرض **غير مُصدَّر** (`postViewModel`)، لأنني ما زلت أرى هذا تفصيلًا تنفيذيًا داخليًا من تفاصيل العرض. فلا حاجة لي إلى اختباره على حدة، ولا أريد أن يلوّث واجهتي.

أنشئ واحدًا عند العرض حتى أحلّل `Body` إلى `HTMLBody`، ثم أستخدم ذلك الحقل في القالب لعرض HTML.

## الخلاصة

إن جمعت ما تعلمته في فصل [قراءة الملفات](reading-files.md) مع ما تعلمته هنا، أمكنك بيسر بناء مُولّد مواقع ساكنة (static site generator) بسيط ومختبَر جيدًا، وإطلاق مدونة خاصة بك. وابحث عن بعض دروس CSS لتبدو جميلة أيضًا.

تتجاوز هذه المقاربة المدونات. فأخذ البيانات من أي مصدر، سواء كان قاعدة بيانات أو API أو نظام ملفات، وتحويلها إلى HTML وإعادتها من خادم، تقنية بسيطة تمتد عبر عقود. يحب الناس أن يندبوا تعقيد تطوير الويب الحديث، لكن هل أنت متأكد أنك لا تفرض التعقيد على نفسك بنفسك؟

إن Go رائعة لتطوير الويب، خصوصًا عندما تفكر بوضوح في متطلباتك الحقيقية للموقع الذي تصنعه. فغالبًا ما يكون توليد HTML على الخادم مقاربة أفضل وأبسط وأعلى أداءً من إنشاء "تطبيق ويب" بتقنيات مثل React.

### ما تعلمناه

- كيف ننشئ قوالب HTML ونعرضها.
- كيف نركّب القوالب معًا و[نتجنب التكرار (DRY)](https://en.wikipedia.org/wiki/Don't_repeat_yourself) في التوصيف المرتبط، لنحافظ على مظهر وإحساس متناسقين.
- كيف نمرّر الدوال إلى القوالب، ولماذا ينبغي أن تفكر مرتين قبل ذلك.
- كيف نكتب "اختبارات الموافقة" التي تساعدنا على اختبار المخرجات الكبيرة القبيحة لأشياء مثل عارضات القوالب.

### عن القوالب بلا منطق

كما في كل مرة، الأمر كله يتعلق بـ**فصل الاهتمامات**. فمن المهم أن نفكر في مسؤوليات الأجزاء المختلفة من نظامنا. وكثيرًا ما يُسرّب الناس منطق أعمال مهمًا إلى القوالب، فيخلطون الاهتمامات ويجعلون الأنظمة صعبة الفهم والصيانة والاختبار.

### ليس لـ HTML فقط

تذكّر أن Go تملك `text/template` لتوليد أنواع أخرى من البيانات من قالب. وإن وجدت نفسك بحاجة إلى تحويل البيانات إلى نوع من المخرجات المنظمة، فقد تفيدك التقنيات المطروحة في هذا الفصل.

### مراجع ومزيد من المواد

- [كتاب 'Learn Web Development with Go' لـ John Calhoun](https://www.calhoun.io/intro-to-templates-p1-contextual-encoding/) يضم عددًا من المقالات الممتازة عن القوالب.
- [Hotwire](https://hotwired.dev) - يمكنك استخدام هذه التقنيات لإنشاء تطبيقات ويب بـ Hotwire. وقد بناه Basecamp، وهم في الأساس فريق Ruby on Rails، لكن لأنه يعمل من جهة الخادم، يمكننا استخدامه مع Go.
