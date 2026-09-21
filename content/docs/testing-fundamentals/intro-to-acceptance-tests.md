---
title: مقدمة إلى اختبارات القبول
weight: 220
---

# مقدمة إلى اختبارات القبول

في `$WORK`، كنا نواجه حاجة متكررة إلى "الإيقاف الرشيق" (graceful shutdown) لخدماتنا. فالإيقاف الرشيق يضمن أن نظامك يُنهي عمله كما ينبغي قبل إنهائه. ومثال من الحياة الواقعية: شخص يحاول إنهاء مكالمة هاتفية على نحو لائق قبل الانتقال إلى الاجتماع التالي، بدلًا من إغلاقها في منتصف الجملة.

سيقدّم هذا الفصل مقدمة عن الإيقاف الرشيق في سياق خادم HTTP، وعن كيفية كتابة "اختبارات القبول" (acceptance tests) لتمنح نفسك ثقة في سلوك كودك.

بعد قراءة هذا الفصل ستعرف كيف تشارك حزمًا باختبارات ممتازة، وتقلل جهد الصيانة، وتزيد ثقتك بجودة عملك.

## ما يكفي من المعلومات عن Kubernetes

نشغّل برامجنا على [Kubernetes](https://kubernetes.io/) (K8s). وسيُنهي K8s "الـ pods" (وهي برامجنا في الواقع العملي) لأسباب متنوعة، ومن أشهرها عندما ندفع كودًا جديدًا نريد نشره.

ونضع لأنفسنا معايير عالية فيما يخص [مقاييس DORA](https://cloud.google.com/blog/products/devops-sre/using-the-four-keys-to-measure-your-devops-performance)، لذا نعمل بطريقة ننشر فيها تحسينات وميزات صغيرة متزايدة إلى الإنتاج عدة مرات يوميًا.

وعندما يريد k8s إنهاء pod، فإنه يبدأ ["دورة حياة الإنهاء" (termination lifecycle)](https://cloud.google.com/blog/products/containers-kubernetes/kubernetes-best-practices-terminating-with-grace)، وجزء من ذلك إرسال إشارة SIGTERM إلى برنامجنا. وهذه رسالة k8s إلى كودنا:

> عليك أن توقف نفسك، وتُنهي أي عمل تقوم به، لأنني بعد "فترة سماح" معينة سأرسل `SIGKILL`، وحينها تنطفئ الأنوار عليك.

وعند `SIGKILL`، سيتوقف فورًا أي عمل كان برنامجك يقوم به.

## إذا لم تكن تملك فترة سماح

بحسب طبيعة برنامجك، إذا تجاهلت `SIGTERM` فقد تصادف مشكلات.

كانت مشكلتنا تحديدًا مع طلبات HTTP الجارية (in-flight). فحين كان اختبار آلي يجرّب واجهتنا البرمجية، إن قرر k8s إيقاف الـ pod مات الخادم، ولم يتلقَّ الاختبار ردًا من الخادم، فيفشل الاختبار.

وكان هذا يطلق تنبيهًا في قناة الحوادث لدينا، يتطلب من مطور أن يتوقف عن عمله ويعالج المشكلة. وهذه الإخفاقات المتقطعة تشتّت انتباه فريقنا وتزعجه.

ولا تقتصر هذه المشكلات على اختباراتنا. فإذا أرسل مستخدم طلبًا إلى نظامك وقُتلت العملية في منتصف تنفيذه، فغالبًا سيستقبله خطأ HTTP من فئة 5xx، وهذه ليست تجربة المستخدم التي تريد تقديمها.

## عندما تملك فترة سماح

ما نريد فعله هو الاستماع إلى `SIGTERM`، وبدلًا من قتل الخادم فورًا، نريد أن:

- نتوقف عن الاستماع إلى أي طلبات جديدة
- نسمح للطلبات الجارية بأن تكتمل
- *ثم* ننهي العملية

## كيف تحصل على فترة سماح

لحسن الحظ، تمتلك Go بالفعل آلية للإيقاف الرشيق للخادم عبر [net/http/Server.Shutdown](https://pkg.go.dev/net/http#Server.Shutdown).

> يُوقف Shutdown الخادم إيقافًا رشيقًا دون مقاطعة أي اتصالات نشطة. ويعمل Shutdown بأن يغلق أولًا كل المستمعين المفتوحين، ثم يغلق كل الاتصالات الخاملة، ثم ينتظر بلا حد أقصى حتى تعود الاتصالات إلى حالة الخمول ثم يُغلق الخادم. وإذا انتهت صلاحية الـ context المقدَّم قبل اكتمال الإيقاف، أعاد Shutdown خطأ الـ context، وإلا أعاد أي خطأ مُعاد من إغلاق الـ Listener(s) الأساسية للخادم.

لمعالجة `SIGTERM` يمكننا استخدام [os/signal.Notify](https://pkg.go.dev/os/signal#Notify)، وهو يرسل أي إشارات واردة إلى channel نوفّره.

وباستخدام هاتين الميزتين من المكتبة القياسية، يمكنك الاستماع إلى `SIGTERM` والإيقاف الرشيق.

## حزمة الإيقاف الرشيق

لهذه الغاية كتبت [https://pkg.go.dev/github.com/quii/go-graceful-shutdown](https://pkg.go.dev/github.com/quii/go-graceful-shutdown). وهي توفّر دالة decorator لـ `*http.Server` لاستدعاء الـ method `Shutdown` عند اكتشاف إشارة `SIGTERM`.

```go
func main() {
	var (
		ctx        = context.Background()
		httpServer = &http.Server{Addr: ":8080", Handler: http.HandlerFunc(acceptancetests.SlowHandler)}
		server     = gracefulshutdown.NewServer(httpServer)
	)

	if err := server.ListenAndServe(ctx); err != nil {
		// this will typically happen if our responses aren't written before the ctx deadline, not much can be done
		log.Fatalf("uh oh, didn't shutdown gracefully, some responses may have been lost %v", err)
	}

	// hopefully, you'll always see this instead
	log.Println("shutdown gracefully! all responses were sent")
}
```

تفاصيل الكود ليست مهمة كثيرًا في هذه القراءة، لكن يجدر بك إلقاء نظرة سريعة عليه قبل المتابعة.

## الاختبارات وحلقات التغذية الراجعة

عندما كتبنا حزمة `gracefulshutdown`، كان لدينا اختبارات وحدة (unit tests) تُثبت أنها تتصرف على نحو صحيح، وهذا منحنا الثقة لإعادة الهيكلة بجرأة. لكننا مع ذلك لم نشعر "بالثقة" أنها تعمل **فعلًا**.

أضفنا حزمة `cmd` وصنعنا برنامجًا حقيقيًا يستخدم الحزمة التي نكتبها. وكنا نشغّله يدويًا، ونطلق طلب HTTP إليه، ثم نرسل `SIGTERM` لنرى ما سيحدث.

**ينبغي أن يشعر المهندس بداخلك بالانزعاج من الاختبار اليدوي**.
إنه ممل، ولا يتوسع، وغير دقيق، ومهدر. وإذا كنت تكتب حزمة تنوي مشاركتها، وتريد أن تبقى بسيطة ورخيصة في التعديل، فلن يكفيك الاختبار اليدوي.

## اختبارات القبول

إذا قرأت بقية هذا الكتاب، فستكون قد كتبت في الغالب "اختبارات الوحدة". واختبارات الوحدة أداة رائعة تتيح إعادة الهيكلة بلا خوف، وتقود تصميمًا معياريًا جيدًا، وتمنع الانحدارات، وتوفّر تغذية راجعة سريعة.

بطبيعتها، لا تختبر سوى أجزاء صغيرة من نظامك. وعادةً لا *تكفي* اختبارات الوحدة وحدها لاستراتيجية اختبار فعّالة. تذكّر أننا نريد أن تكون أنظمتنا **قابلة للشحن دائمًا**. لا يمكننا الاعتماد على الاختبار اليدوي، لذا نحتاج نوعًا آخر من الاختبار: **اختبارات القبول**.

### ما هي؟

اختبارات القبول نوع من "اختبار الصندوق الأسود" (black-box test). ويُشار إليها أحيانًا بـ"الاختبارات الوظيفية" (functional tests). وينبغي أن تجرّب النظام كما يفعل مستخدم النظام.

ومصطلح "الصندوق الأسود" يشير إلى أن كود الاختبار لا يصل إلى الباطن الداخلي للنظام، بل لا يستطيع إلا استخدام واجهته العامة وتقديم تحققات على السلوكيات التي يلاحظها. وهذا يعني أنها لا تستطيع اختبار النظام إلا ككل متكامل.

وهذه سمة مفيدة لأنها تعني أن الاختبارات تجرّب النظام كما يفعل المستخدم، فلا يمكنها استخدام أي حيل خاصة قد تُنجح اختبارًا من دون أن تثبت فعلًا ما تحتاج إلى إثباته. وهذا يشبه مبدأ تفضيل أن تكون ملفات اختبار الوحدة لديك داخل حزمة اختبار منفصلة، مثلًا `package mypkg_test` بدلًا من `package mypkg`.

### فوائد اختبارات القبول

- عندما تنجح، تعرف أن نظامك كاملًا يتصرف كما تريد.
- وهي أدق وأسرع وأقل جهدًا من الاختبار اليدوي.
- وعندما تُكتب جيدًا، تعمل كتوثيق دقيق ومُتحقَّق منه لنظامك. فلا تقع في فخ التوثيق الذي ينفصل عن السلوك الحقيقي للنظام.
- بلا mocking! فكل شيء حقيقي.

### العيوب المحتملة مقارنةً باختبارات الوحدة

- كتابتها مكلفة.
- وتستغرق وقتًا أطول في التشغيل.
- وهي مرتبطة بتصميم النظام.
- وعندما تفشل، لا تعطيك عادةً السبب الجذري، وقد يصعب تتبع أخطائها.
- ولا تعطيك تغذية راجعة عن الجودة الداخلية لنظامك. فيمكنك كتابة كود رديء تمامًا ومع ذلك ينجح اختبار قبول.
- وليست كل السيناريوهات عملية التطبيق بسبب طبيعة الصندوق الأسود.

ولهذا السبب من الحماقة الاعتماد على اختبارات القبول وحدها. فهي لا تملك كثيرًا من خصائص اختبارات الوحدة، والنظام الذي يضم عددًا كبيرًا منها سيعاني عادةً من حيث تكاليف الصيانة وسوء وقت الوصول.

#### وقت الوصول (Lead time)؟

يشير وقت الوصول إلى المدة التي يستغرقها الانتقال من دمج commit في فرعك الرئيسي إلى نشره في الإنتاج. وقد يتراوح هذا الرقم لدى بعض الفرق بين أسابيع بل أشهر، وبين دقائق معدودة. ومرة أخرى، في `$WORK` نُقدّر نتائج DORA ونريد إبقاء وقت الوصول لدينا أقل من 10 دقائق.

يلزم نهج اختبار متوازن لنظام موثوق بوقت وصول ممتاز، ويُوصف هذا عادةً بمصطلح [هرم الاختبار (Test Pyramid)](https://martinfowler.com/articles/practical-test-pyramid.html).

## كيف تكتب اختبارات قبول أساسية

كيف يرتبط هذا بالمشكلة الأصلية؟ لقد كتبنا للتو حزمة هنا، وهي قابلة لاختبار الوحدة بالكامل.

وكما ذكرت، لم تمنحنا اختبارات الوحدة الثقة التي نحتاجها تمامًا. نريد أن نتأكد *تمامًا* أن الحزمة تعمل عند دمجها مع برنامج حقيقي قيد التشغيل. وينبغي أن نستطيع أتمتة الفحوص اليدوية التي كنا نجريها.

لنلقِ نظرة على برنامج الاختبار:

```go
func main() {
	var (
		ctx        = context.Background()
		httpServer = &http.Server{Addr: ":8080", Handler: http.HandlerFunc(acceptancetests.SlowHandler)}
		server     = gracefulshutdown.NewServer(httpServer)
	)

	if err := server.ListenAndServe(ctx); err != nil {
		// this will typically happen if our responses aren't written before the ctx deadline, not much can be done
		log.Fatalf("uh oh, didn't shutdown gracefully, some responses may have been lost %v", err)
	}

	// hopefully, you'll always see this instead
	log.Println("shutdown gracefully! all responses were sent")
}
```

قد تكون خمّنت أن `SlowHandler` فيه `time.Sleep` لتأخير الرد، حتى يكون لدي وقت لإرسال `SIGTERM` ورؤية ما سيحدث. والباقي كود نمطي إلى حد كبير:

- أنشئ `net/http/Server`؛
- غلّفه بالمكتبة (انظر: [نمط الـ Decorator](https://en.wikipedia.org/wiki/Decorator_pattern))؛
- استخدم النسخة المغلّفة لتنفيذ `ListenAndServe`.

### الخطوات العامة لاختبار القبول

- ابنِ البرنامج
- شغّله (وانتظر حتى يستمع على `8080`)
- أرسل طلب HTTP إلى الخادم
- قبل أن تسنح للخادم فرصة إرسال رد HTTP، أرسل `SIGTERM`
- انظر إن كنا ما زلنا نتلقى ردًا

### بناء البرنامج وتشغيله

```go
package acceptancetests

import (
	"fmt"
	"math/rand"
	"net"
	"os"
	"os/exec"
	"path/filepath"
	"syscall"
	"time"
)

const (
	baseBinName = "temp-testbinary"
)

func LaunchTestProgram(port string) (cleanup func(), sendInterrupt func() error, err error) {
	binName, err := buildBinary()
	if err != nil {
		return nil, nil, err
	}

	sendInterrupt, kill, err := runServer(binName, port)

	cleanup = func() {
		if kill != nil {
			kill()
		}
		os.Remove(binName)
	}

	if err != nil {
		cleanup() // even though it's not listening correctly, the program could still be running
		return nil, nil, err
	}

	return cleanup, sendInterrupt, nil
}

func buildBinary() (string, error) {
	binName := randomString(10) + "-" + baseBinName

	build := exec.Command("go", "build", "-o", binName)

	if err := build.Run(); err != nil {
		return "", fmt.Errorf("cannot build tool %s: %s", binName, err)
	}
	return binName, nil
}

func runServer(binName string, port string) (sendInterrupt func() error, kill func(), err error) {
	dir, err := os.Getwd()
	if err != nil {
		return nil, nil, err
	}

	cmdPath := filepath.Join(dir, binName)

	cmd := exec.Command(cmdPath)

	if err := cmd.Start(); err != nil {
		return nil, nil, fmt.Errorf("cannot run temp converter: %s", err)
	}

	kill = func() {
		_ = cmd.Process.Kill()
	}

	sendInterrupt = func() error {
		return cmd.Process.Signal(syscall.SIGTERM)
	}

	err = waitForServerListening(port)

	return
}

func waitForServerListening(port string) error {
	for i := 0; i < 30; i++ {
		conn, _ := net.Dial("tcp", net.JoinHostPort("localhost", port))
		if conn != nil {
			conn.Close()
			return nil
		}
		time.Sleep(100 * time.Millisecond)
	}
	return fmt.Errorf("nothing seems to be listening on localhost:%s", port)
}

func randomString(n int) string {
	var letters = []rune("abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789")

	s := make([]rune, n)
	for i := range s {
		s[i] = letters[rand.Intn(len(letters))]
	}
	return string(s)
}
```

`LaunchTestProgram` مسؤولة عن:

- بناء البرنامج
- تشغيل البرنامج
- انتظار أن يستمع على المنفذ `8080`
- توفير دالة `cleanup` لقتل البرنامج وحذفه، لضمان أننا بعد انتهاء اختباراتنا نبقى في حالة نظيفة
- توفير دالة `interrupt` لإرسال `SIGTERM` إلى البرنامج حتى نستطيع اختبار السلوك

وبصراحة، هذا ليس أجمل كود في العالم، لكن ركّز فقط على الدالة المصدَّرة `LaunchTestProgram`؛ فالدوال غير المصدَّرة التي تستدعيها كود نمطي غير مثير.

وكما نوقش، يميل إعداد اختبارات القبول إلى أن يكون أصعب. لكن هذا الكود يجعل كود *الاختبار* أسهل قراءةً إلى حد كبير، وغالبًا مع اختبارات القبول، ما إن تكتب الكود الشكلي حتى ينتهي الأمر وتنساه.

### اختبار القبول (أو الاختبارات)

أردنا اختبارَي قبول لبرنامجين، أحدهما بإيقاف رشيق والآخر بدونه، لنرى نحن والقراء الفرق في السلوك. ومع `LaunchTestProgram` لبناء البرنامجين وتشغيلهما، صارت كتابة اختبارَي القبول بسيطة جدًا، ونستفيد من إعادة الاستخدام مع بعض دوال المساعدة.

وهذا اختبار الخادم *مع* الإيقاف الرشيق، [ويمكنك العثور على الاختبار بدونه على GitHub](https://github.com/quii/go-graceful-shutdown/blob/main/acceptancetests/withoutgracefulshutdown/main_test.go)

```go
package main

import (
	"testing"
	"time"

	"github.com/quii/go-graceful-shutdown/acceptancetests"
	"github.com/quii/go-graceful-shutdown/assert"
)

const (
	port = "8080"
	url  = "<http://localhost:" + port
)

func TestGracefulShutdown(t *testing.T) {
	cleanup, sendInterrupt, err := acceptancetests.LaunchTestProgram(port)
	if err != nil {
		t.Fatal(err)
	}
	t.Cleanup(cleanup)

	// just check the server works before we shut things down
	assert.CanGet(t, url)

	// fire off a request, and before it has a chance to respond send SIGTERM.
	time.AfterFunc(50*time.Millisecond, func() {
		assert.NoError(t, sendInterrupt())
	})
	// Without graceful shutdown, this would fail
	assert.CanGet(t, url)

	// after interrupt, the server should be shutdown, and no more requests will work
	assert.CantGet(t, url)
}
```

ومع تغليف الإعداد بعيدًا، صارت الاختبارات شاملة وواصفة للسلوك وسهلة المتابعة نسبيًا.

`assert.CanGet/CantGet` دوال مساعدة صنعتها لتقليل التكرار في هذا التحقق الشائع في هذه المجموعة.

```go
func CanGet(t testing.TB, url string) {
	errChan := make(chan error)

	go func() {
		res, err := http.Get(url)
		if err != nil {
			errChan <- err
			return
		}
		res.Body.Close()
		errChan <- nil
	}()

	select {
	case err := <-errChan:
		NoError(t, err)
	case <-time.After(3 * time.Second):
		t.Errorf("timed out waiting for request to %q", url)
	}
}
```

سيطلق هذا طلب `GET` إلى `URL` على goroutine، وإذا ردّ بلا خطأ قبل 3 ثوانٍ فلن يفشل. وقد حُذفت `CantGet` للإيجاز، [لكن يمكنك عرضها على GitHub هنا](https://github.com/quii/go-graceful-shutdown/blob/main/assert/assert.go#L61).

ومن المهم أن نلاحظ مجددًا أن Go تمتلك كل الأدوات التي تحتاجها لكتابة اختبارات القبول جاهزة. فلست *بحاجة* إلى إطار عمل خاص لبناء اختبارات القبول.

### استثمار صغير بعائد كبير

بهذه الاختبارات، يستطيع القراء النظر إلى البرامج المثال والثقة بأن المثال *يعمل فعلًا*، فيثقون بما تدّعيه الحزمة.

والأهم أننا، بصفتنا المؤلف، نحصل على **تغذية راجعة سريعة** و**ثقة هائلة** بأن الحزمة تعمل في بيئة واقعية.

```shell
go test -count=1 ./...
ok  	github.com/quii/go-graceful-shutdown	0.196s
?   	github.com/quii/go-graceful-shutdown/acceptancetests	[no test files]
ok  	github.com/quii/go-graceful-shutdown/acceptancetests/withgracefulshutdown	4.785s
ok  	github.com/quii/go-graceful-shutdown/acceptancetests/withoutgracefulshutdown	2.914s
?   	github.com/quii/go-graceful-shutdown/assert	[no test files]
```

## الخلاصة

في هذا المقال، أدخلنا اختبارات القبول إلى حزام أدواتك في الاختبار. فهي بالغة القيمة عندما تبدأ في بناء أنظمة حقيقية، وهي مكمّل مهم لاختبارات الوحدة.

وتعتمد طبيعة *كيفية* كتابة اختبارات القبول على النظام الذي تبنيه، لكن المبادئ تبقى نفسها. عامل نظامك كـ"صندوق أسود". وإذا كنت تبني موقعًا إلكترونيًا، فينبغي أن تتصرف اختباراتك كتصرف المستخدم، لذا ستحتاج إلى متصفح ويب بلا واجهة (headless) مثل [Selenium](https://www.selenium.dev/)، للنقر على الروابط وملء النماذج وما إلى ذلك. وفي حالة واجهة RESTful API، سترسل طلبات HTTP باستخدام عميل.

### المضي أبعد في الأنظمة الأكثر تعقيدًا

لا تميل الأنظمة غير البسيطة إلى أن تكون تطبيقات أحادية العملية كالتي ناقشناها. فعادةً ستعتمد على أنظمة أخرى مثل قاعدة بيانات. ولهذه السيناريوهات، ستحتاج إلى أتمتة بيئة محلية لتجربتك فيها. وأدوات مثل [docker-compose](https://docs.docker.com/compose/) مفيدة لتشغيل حاويات البيئة التي تحتاجها لتنفيذ نظامك محليًا.

### الفصل التالي

في هذا المقال كُتب اختبار القبول بأثر رجعي. لكن في كتاب [Growing Object-Oriented Software](http://www.growing-object-oriented-software.com) يوضح المؤلفان أننا نستطيع استخدام اختبارات القبول في نهج موجه بالاختبار لتعمل كـ"نجم الشمال" الذي يوجّه جهودنا.

وكلما زاد تعقيد الأنظمة، يمكن أن تخرج تكاليف كتابة اختبارات القبول وصيانتها عن السيطرة سريعًا. وهناك قصص لا تُحصى عن فرق تطوير شلّتها مجموعات اختبارات قبول مكلفة.

سيقدّم الفصل التالي استخدام اختبار القبول لتوجيه تصميمنا، إلى جانب مبادئ وتقنيات لإدارة تكاليف اختبارات القبول.

### تحسين جودة المصادر المفتوحة

إذا كنت تكتب حزمًا تنوي مشاركتها، فأشجعك على إنشاء برامج مثال بسيطة توضح ما تفعله حزمتك، وعلى استثمار وقتك في وجود اختبارات قبول سهلة المتابعة تمنحك أنت والمستخدمين المحتملين لعملك ثقة.

ومثل [الأمثلة القابلة للاختبار (Testable Examples)](https://go.dev/blog/examples)، فإن رؤية هذا الجهد الإضافي الصغير في تجربة المطور تبني الثقة في عملك إلى حد بعيد، وستقلل تكاليف الصيانة لديك.

## إعلان توظيف في `$WORK`

إذا كنت ترغب في العمل في بيئة مع مهندسين آخرين يحلون مشكلات مثيرة، وتسكن قريبًا من لندن أو بورتو أو حولهما، وتستمتع بمحتوى هذا الفصل والكتاب — فأرجو [مراسلتي على Twitter](https://twitter.com/quii)، وربما نعمل معًا قريبًا!
