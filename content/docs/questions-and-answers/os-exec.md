---
title: تنفيذ أوامر النظام (OS Exec)
weight: 340
---

# تنفيذ أوامر النظام (OS Exec)

**[يمكنك العثور على كل الكود هنا](https://github.com/quii/learn-go-with-tests/tree/main/q-and-a/os-exec)**

يسأل [keith6014](https://www.reddit.com/user/keith6014) على [reddit](https://www.reddit.com/r/golang/comments/aaz8ji/testdata_and_function_setup_help/)

> أُنفّذ أمرًا باستخدام os/exec.Command() يُنتج بيانات XML. وسيُنفَّذ هذا الأمر داخل دالة اسمها GetData().

> ولكي أختبر GetData()، لديّ بعض بيانات الاختبار (testdata) التي أنشأتها.

> في ملف _test.go لديّ TestGetData يستدعي GetData()، لكنه سيستخدم os.exec، وأريد بدلًا من ذلك أن يستخدم بيانات اختباري.

> فما الطريقة الجيدة لتحقيق ذلك؟ هل ينبغي عند استدعاء GetData أن يكون لديّ وضع "test" بعلامة ما ليقرأ ملفًا، مثل GetData(mode string)؟

بضعة أمور

- عندما يصعب اختبار شيء ما، فالسبب غالبًا أن فصل الاهتمامات (separation of concerns) ليس على أحسن وجه
- لا تُضِف "أوضاع اختبار" إلى كودك، بل استخدم [حقن الاعتماديات](../go-fundamentals/dependency-injection.md) لتتمكن من نمذجة اعتمادياتك وفصل الاهتمامات.

وقد أخذت حريتي في تخمين شكل الكود

```go
type Payload struct {
	Message string `xml:"message"`
}

func GetData() string {
	cmd := exec.Command("cat", "msg.xml")

	out, _ := cmd.StdoutPipe()
	var payload Payload
	decoder := xml.NewDecoder(out)

	// these 3 can return errors but I'm ignoring for brevity
	cmd.Start()
	decoder.Decode(&payload)
	cmd.Wait()

	return strings.ToUpper(payload.Message)
}
```

- يستخدم `exec.Command` الذي يتيح لك تنفيذ أمر خارجي بالنسبة للعملية (process)
- نلتقط المخرجات في `cmd.StdoutPipe` التي تُرجع لنا `io.ReadCloser` (وسيصبح هذا مهمًا لاحقًا)
- وبقية الكود منسوخ تقريبًا من [التوثيق الممتاز](https://golang.org/pkg/os/exec/#example_Cmd_StdoutPipe).
    - نلتقط أي مخرجات من stdout في `io.ReadCloser` ثم نشغّل الأمر بـ `Start` ثم ننتظر قراءة كل البيانات باستدعاء `Wait`. وبين هذين الاستدعاءين نفكّ ترميز البيانات إلى struct الـ `Payload`.

هذا ما يحتوي عليه `msg.xml`

```xml
<payload>
    <message>Happy New Year!</message>
</payload>
```

كتبت اختبارًا بسيطًا لأعرضه في العمل

```go
func TestGetData(t *testing.T) {
	got := GetData()
	want := "HAPPY NEW YEAR!"

	if got != want {
		t.Errorf("got %q, want %q", got, want)
	}
}
```

## كود قابل للاختبار

الكود القابل للاختبار مفصول وأحادي الغرض. ويبدو لي أن في هذا الكود اهتمامين رئيسيين

1. جلب بيانات XML الخام
2. فك ترميز بيانات XML وتطبيق منطق أعمالنا (في هذه الحالة `strings.ToUpper` على `<message>`)

الجزء الأول مجرد نسخ للمثال من المكتبة القياسية.

أما الجزء الثاني فهو حيث يوجد منطق أعمالنا، وبنظرة إلى الكود نرى من أين تبدأ "نقطة الفصل" في منطقنا؛ إنها حيث نحصل على `io.ReadCloser`. ويمكننا استخدام هذا التجريد الموجود لفصل الاهتمامات وجعل كودنا قابلًا للاختبار.

**مشكلة GetData أن منطق الأعمال مترابط مع وسيلة الحصول على XML. ولتحسين تصميمنا نحتاج إلى فصل الاثنين**

يمكن أن يقوم `TestGetData` بدور اختبار التكامل (integration test) بين اهتمامينا، لذا سنُبقيه لنتأكد أنه يظل يعمل.

وهذا شكل الكود بعد الفصل

```go
type Payload struct {
	Message string `xml:"message"`
}

func GetData(data io.Reader) string {
	var payload Payload
	xml.NewDecoder(data).Decode(&payload)
	return strings.ToUpper(payload.Message)
}

func getXMLFromCommand() io.Reader {
	cmd := exec.Command("cat", "msg.xml")
	out, _ := cmd.StdoutPipe()

	cmd.Start()
	data, _ := io.ReadAll(out)
	cmd.Wait()

	return bytes.NewReader(data)
}

func TestGetDataIntegration(t *testing.T) {
	got := GetData(getXMLFromCommand())
	want := "HAPPY NEW YEAR!"

	if got != want {
		t.Errorf("got %q, want %q", got, want)
	}
}
```

الآن بعد أن صارت `GetData` تتلقى مدخلها من مجرد `io.Reader`، جعلناها قابلة للاختبار ولم تعد معنية بكيفية جلب البيانات؛ فيستطيع الناس إعادة استخدام الدالة مع أي شيء يُرجع `io.Reader` (وهو أمر شائع جدًا). مثلًا، يمكننا أن نبدأ بجلب XML من رابط (URL) بدلًا من سطر الأوامر.

```go
func TestGetData(t *testing.T) {
	input := strings.NewReader(`
<payload>
    <message>Cats are the best animal</message>
</payload>`)

	got := GetData(input)
	want := "CATS ARE THE BEST ANIMAL"

	if got != want {
		t.Errorf("got %q, want %q", got, want)
	}
}

```

وهذا مثال لاختبار وحدة (unit test) لـ `GetData`.

وبفصل الاهتمامات واستخدام تجريدات موجودة أصلًا في Go، يصبح اختبار منطق أعمالنا المهم أمرًا يسيرًا.
