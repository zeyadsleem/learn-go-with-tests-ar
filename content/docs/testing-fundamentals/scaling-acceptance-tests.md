---
title: توسيع نطاق اختبارات القبول
weight: 230
---

# توسيع نطاق اختبارات القبول

هذا الفصل تكملة لفصل [مقدمة إلى اختبارات القبول](intro-to-acceptance-tests.md). ويمكنك العثور على [كود هذا الفصل النهائي على GitHub](https://github.com/quii/go-specs-greet).

اختبارات القبول ضرورية، وهي تؤثر تأثيرًا مباشرًا في قدرتك على تطوير نظامك بثقة عبر الزمن، وبكلفة تغيير معقولة.

وهي أيضًا أداة بديعة تساعدك في التعامل مع الكود القديم (legacy code). فحين تواجه قاعدة كود ضعيفة بلا أي اختبارات، قاوم الرغبة في البدء بإعادة الهيكلة. واكتب بدلًا من ذلك بعض اختبارات القبول لتصنع لنفسك شبكة أمان تتيح لك تغيير داخل النظام بحرية دون التأثير في سلوكه الخارجي الوظيفي. فاختبارات القبول لا تهتم بالجودة الداخلية، لذا فهي مناسبة جدًا لهذه المواقف.

بعد قراءتك هذا الفصل ستدرك أن اختبارات القبول مفيدة للتحقق، ويمكن استخدامها أيضًا في عملية التطوير بأن تساعدنا على تغيير نظامنا بتأنٍّ ومنهجية، ما يقلل الجهد المهدر.

## مواد تمهيدية

وُلدت فكرة هذا الفصل من سنوات طويلة من الإحباط مع اختبارات القبول. وهذان فيديوهان أنصحك بمشاهدتهما:

- Dave Farley - [How to write acceptance tests](https://www.youtube.com/watch?v=JDD5EEJgpHU)
- Nat Pryce - [E2E functional tests that can run in milliseconds](https://www.youtube.com/watch?v=Fk4rCn4YLLU)

كتاب "Growing Object Oriented Software" (GOOS) كتاب بالغ الأهمية لكثير من مهندسي البرمجيات، ومنهم أنا. والنهج الذي يوصي به هو نفسه الذي أدرّب المهندسين الذين أعمل معهم على اتباعه.

- [GOOS](http://www.growing-object-oriented-software.com) - Nat Pryce & Steve Freeman

وأخيرًا، تحدثت أنا و[Riya Dattani](https://twitter.com/dattaniriya) عن هذا الموضوع في سياق الـ BDD في محاضرتنا، [Acceptance tests, BDD and Go](https://www.youtube.com/watch?v=ZMWJCk_0WrY).

## مراجعة سريعة

نتحدث عن اختبارات "الصندوق الأسود" (black-box) التي تتحقق من أن نظامك يتصرف كما هو متوقع من الخارج، ومن "**منظور الأعمال**". فهذه الاختبارات لا تملك وصولًا إلى أعماق النظام الذي تختبره؛ بل تهتم فقط بـ**ما** يفعله نظامك لا بـ**كيفية** فعله.

## تشريح اختبارات القبول السيئة

على مدى سنوات طويلة، عملت في شركات وفرق عدة. أدرك كل منها الحاجة إلى اختبارات القبول؛ طريقة ما لاختبار نظام من وجهة نظر المستخدم والتحقق من أنه يعمل كما أُريد له، لكن كلفة هذه الاختبارات صارت، بلا استثناء تقريبًا، مشكلة حقيقية للفريق.

- بطيئة التشغيل
- هشّة
- متقلبة (flaky)
- مكلفة الصيانة، ويبدو أنها تجعل تغيير البرمجيات أصعب مما ينبغي
- لا تعمل إلا في بيئة معينة، ما يسبب حلقات تغذية راجعة بطيئة وضعيفة

لنفترض أنك تنوي كتابة اختبار قبول حول موقع إلكتروني تبنيه. فتقرر استخدام متصفح ويب بلا واجهة (headless) (مثل [Selenium](https://www.selenium.dev)) لمحاكاة مستخدم ينقر أزرار موقعك للتحقق من أنه يفعل ما ينبغي عليه فعله.

ومع الوقت، يضطر ترميز (markup) موقعك إلى التغيير كلما اكتُشفت ميزات جديدة، ويتجادل المهندسون للمرة الألف حول ما إذا كان ينبغي أن يكون شيء ما `<article>` أو `<section>`.

ومع أن فريقك لا يجري إلا تغييرات طفيفة على النظام، تكاد لا تُلاحظ من المستخدم الفعلي، تجد نفسك تهدر وقتًا طويلًا في تحديث اختبارات القبول لديك.

### الارتباط المحكم

فكّر فيما يدفع اختبارات القبول إلى التغيير:

- تغيير في السلوك الخارجي. إذا أردت تغيير ما يفعله النظام، فتغيير مجموعة اختبارات القبول يبدو معقولًا، بل مرغوبًا.
- تغيير في تفاصيل التنفيذ / إعادة الهيكلة (refactoring). من الناحية المثالية، لا ينبغي أن يدفع ذلك إلى تغيير، أو إن دفعه فيكون تغييرًا طفيفًا.

لكن السبب الثاني هو ما يجعل اختبارات القبول تضطر إلى التغيير في كثير من الأحيان، إلى حد أن المهندسين يتحوّلون إلى مترددين في تغيير نظامهم بسبب الجهد المتوقّع لتحديث الاختبارات!

![أنا وRiya نتحدث عن فصل الاهتمامات في اختباراتنا](https://i.imgur.com/bbG6z57.png)

تنبع هذه المشكلات من عدم تطبيق عادات هندسية راسخة ومجرّبة كتب عنها المؤلفون المذكورون أعلاه. **فلا يمكنك كتابة اختبارات القبول كما تكتب اختبارات الوحدة**؛ فهي تحتاج إلى تفكير أعمق وممارسات مختلفة.

## تشريح اختبارات القبول الجيدة

إذا أردنا اختبارات قبول لا تتغير إلا عندما نغيّر السلوك لا تفاصيل التنفيذ، فمن المنطقي أننا نحتاج إلى فصل هذين الاهتمامين.

### عن أنواع التعقيد

بصفتنا مهندسي برمجيات، علينا التعامل مع نوعين من التعقيد.

- **التعقيد العرضي** (accidental complexity) هو التعقيد الذي نضطر إلى التعامل معه لأننا نعمل مع الحواسيب، أشياء مثل الشبكات والأقراص وواجهات البرمجة (APIs) وغيرها.

- **التعقيد الجوهري** (essential complexity) يُشار إليه أحيانًا بـ"منطق المجال" (domain logic). وهو القواعد والحقائق الخاصة بمجالك.
  - مثال: "إذا سحب صاحب الحساب مالًا أكثر مما هو متاح، يصبح مدينًا بالسحب على المكشوف". هذه العبارة لا تقول شيئًا عن الحواسيب؛ فقد كانت صحيحة قبل أن تُستخدم الحواسيب في البنوك أصلًا!

ينبغي أن يكون التعقيد الجوهري قابلًا للتعبير لشخص غير تقني، ومن القيّم أن ننمذجه في كود "المجال" لدينا، وفي اختبارات القبول لدينا.

### فصل الاهتمامات

ما اقترحه Dave Farley في الفيديو السابق، وما ناقشناه أنا وRiya أيضًا، هو أن تكون لدينا فكرة **المواصفات** (specifications). فالمواصفات تصف سلوك النظام الذي نريده دون أن ترتبط بتعقيد عرضي أو بتفاصيل التنفيذ.

ينبغي أن تبدو هذه الفكرة معقولة لك. فنحن في كود الإنتاج نسعى كثيرًا إلى فصل الاهتمامات وفك ارتباط وحدات العمل. أفلا تتردد في إدخال `interface` ليتيح لمعالج `HTTP` لديك أن ينفصل عن الاهتمامات غير المتعلقة بـ HTTP؟ لنأخذ خط التفكير نفسه إلى اختبارات القبول لدينا.

يصف Dave Farley بنية محددة.

![Dave Farley يتحدث عن اختبارات القبول](https://i.imgur.com/nPwpihG.png)

وفي GopherconUK، صغنا أنا وRiya ذلك بمصطلحات Go.

![فصل الاهتمامات](https://i.imgur.com/qdY4RJe.png)

### اختبارات فائقة

يتيح لنا فصل كيفية تنفيذ المواصفة أن نعيد استخدامها في سيناريوهات مختلفة. فيمكننا:

#### جعل الـ drivers لدينا قابلة للتهيئة

وهذا يعني أنك تستطيع تشغيل اختبارات القبول لديك محليًا، وفي بيئات الاختبار (staging)، و(من الناحية المثالية) في الإنتاج.
- كثير من الفرق يهندس أنظمته بحيث يستحيل تشغيل اختبارات القبول محليًا. وهذا يُدخل حلقة تغذية راجعة بطيئة بشكل لا يُطاق. ألن تفضّل أن تكون واثقًا أن اختبارات القبول لديك ستنجح _قبل_ دمج كودك؟ وإذا بدأت الاختبارات تنكسر، فهل يُقبل أن تعجز عن إعادة إنتاج الفشل محليًا، وتضطر بدلًا من ذلك إلى تسجيل تغييراتك وتأمل أن تنجح بعد 20 دقيقة في بيئة مختلفة؟
- تذكّر أن نجاح اختباراتك في بيئة الاختبار لا يعني أن نظامك سيعمل. فتماثل بيئة التطوير والإنتاج (Dev/Prod parity) في أفضل الأحوال نصف حقيقة. [أنا أختبر في بيئة الإنتاج](https://increment.com/testing/i-test-in-production/).
- هناك دائمًا فروق بين البيئات يمكن أن تؤثر في *سلوك* نظامك. فقد تكون ترويسات التخزين المؤقت (cache headers) في شبكة توزيع المحتوى (CDN) مضبوطة خطأً؛ وقد تتصرف خدمة لاحقة (downstream) تعتمد عليها بشكل مختلف؛ وقد تكون قيمة تهيئة غير صحيحة. لكن ألن يكون جميلًا أن تستطيع تشغيل مواصفاتك في الإنتاج لاكتشاف هذه المشكلات سريعًا؟

#### وصّل _drivers مختلفة_ لاختبار أجزاء أخرى من نظامك

تمنحنا هذه المرونة القدرة على اختبار السلوكيات عند مستويات مختلفة من التجريد والمعمارية، ما يتيح لنا اختبارات أكثر تركيزًا تتجاوز اختبارات الصندوق الأسود.
- على سبيل المثال، قد تكون لديك صفحة ويب وواجهة برمجية خلفها. فلمَ لا تستخدم المواصفة نفسها لاختبار الاثنتين؟ يمكنك استخدام متصفح ويب بلا واجهة لصفحة الويب، واستدعاءات HTTP للواجهة البرمجية.
- وإن أخذنا الفكرة أبعد، فمن الناحية المثالية نريد أن **ينمذج الكود التعقيد الجوهري** (ككود "مجال")، لذا ينبغي أن نستطيع أيضًا استخدام مواصفاتنا في اختبارات الوحدة. وهذا سيمنحنا تغذية راجعة سريعة بأن التعقيد الجوهري في نظامنا منمذج ويتصرف على نحو صحيح.

### تغيّر اختبارات القبول للأسباب الصحيحة

مع هذا النهج، يصبح السبب الوحيد لتغيّر مواصفاتك هو تغيّر سلوك النظام، وهذا أمر معقول.

- إذا اضطرت واجهتك البرمجية HTTP إلى التغيير، فلديك مكان واحد واضح لتحديثها، وهو الـ driver.
- وإذا تغيّر الترميز (markup) لديك، حدّث الـ driver المعني مرة أخرى.

وكلما كبر نظامك، ستجد نفسك تعيد استخدام الـ drivers في اختبارات عدة، وهذا يعني مجددًا أنك عند تغيّر تفاصيل التنفيذ لن تحتاج إلا إلى تحديث مكان واحد واضح عادةً.

وحين يُطبّق النهج على نحو صحيح، يمنحنا مرونة في تفاصيل تنفيذنا وثباتًا في مواصفاتنا. والأهم أنه يوفّر بنية بسيطة وواضحة لإدارة التغيير، وهذه تصبح ضرورية مع نمو النظام والفريق الذي يعمل عليه.

### اختبارات القبول كطريقة لتطوير البرمجيات

في محاضرتنا، ناقشنا أنا وRiya اختبارات القبول وعلاقتها بـ BDD. وتحدثنا عن أن بدء عملك بمحاولة _فهم المشكلة التي تسعى إلى حلها_ والتعبير عنها في صورة مواصفة يساعد على تركيز نيتك، وهو طريقة رائعة لبدء عملك.

تعرفت على طريقة العمل هذه أول مرة في GOOS. وقد لخّصت الأفكار منذ فترة في مدونتي. وهذا مقتطف من مقالي [Why TDD](https://quii.dev/The_Why_of_TDD)

---

يتركّز TDD على تمكينك من التصميم للسلوك الذي تحتاجه بدقة، على نحو تكراري. فعند بدء مجال جديد، عليك تحديد سلوك رئيسي ضروري وتقليص النطاق بجرأة.

اتبع نهجًا "من الأعلى إلى الأسفل" (top-down)، يبدأ باختبار قبول (AT) يمارس السلوك من الخارج. وسيكون هذا الاختبار بمثابة نجم الشمال لجهودك. وكل ما ينبغي أن تركّز عليه هو جعل ذلك الاختبار ينجح. والأرجح أن يظل هذا الاختبار فاشلًا لفترة بينما تطوّر كودًا كافيًا لنجاحه.

![](https://i.imgur.com/pxTaYu4.png)

وبمجرد أن يصبح اختبار القبول جاهزًا، يمكنك الدخول في عملية TDD لاستخلاص وحدات كافية لنجاح اختبار القبول. والحيلة هنا ألا تقلق كثيرًا بشأن التصميم في هذه المرحلة؛ اكتفِ بكود كافٍ لنجاح اختبار القبول لأنك ما زلت تتعلم المشكلة وتستكشفها.

غالبًا ما تكون هذه الخطوة الأولى أوسع مما تظن، إذ تشمل إعداد خوادم الويب والتوجيه والتهيئة وغيرها، ولهذا من الضروري إبقاء نطاق العمل صغيرًا. نريد أن نخطو تلك الخطوة الإيجابية الأولى على لوحتنا البيضاء وأن تكون مدعومة باختبار قبول ناجح، حتى نواصل التكرار بسرعة وأمان.

![](https://i.imgur.com/t5y5opw.png)

وبينما تطوّر، أصغِ إلى اختباراتك، فهي ينبغي أن تمنحك إشارات تساعدك على دفع تصميمك في اتجاه أفضل، لكن مع بقائه مرتكزًا على السلوك لا على خيالنا.

عادةً ما تنمو وحدتك الأولى التي تقوم بالعمل الشاق لنجاح اختبار القبول حتى تصبح أكبر مما يريحك، حتى مع هذا القدر الصغير من السلوك. وهنا يمكنك البدء بالتفكير في كيفية تقسيم المشكلة وإدخال متعاونين جدد.

![](https://i.imgur.com/UYqd7Cq.png)

وهنا تكون الـ test doubles (مثل الـ fakes والـ mocks) مفيدة، لأن معظم التعقيد الذي يعيش داخل البرمجيات لا يقيم عادةً في تفاصيل التنفيذ، بل "بين" الوحدات وكيفية تفاعلها.

#### مخاطر النهج من الأسفل إلى الأعلى

هذا نهج "من الأعلى إلى الأسفل" لا "من الأسفل إلى الأعلى". ولهذا الأخير استخداماته، لكنه يحمل عنصر مخاطرة. فبناء "خدمات" وكود دون دمجه سريعًا في تطبيقك ودون التحقق منه باختبار عالي المستوى، **يعرّضك لهدر جهد كبير في أفكار غير متحقَّق منها**.

وهذه خاصية جوهرية في النهج الموجّه باختبارات القبول، أي استخدام الاختبارات للحصول على تحقق حقيقي من كودنا.

كثيرًا ما صادفت مهندسين أنجزوا قطعة كود، بمعزل عن غيرها، بنهج من الأسفل إلى الأعلى، يعتقدون أنها ستحل مهمة ما، لكنها:

- لا تعمل كما نريد
- تفعل أشياء لا نحتاجها
- لا تتكامل بسهولة
- تحتاج إلى إعادة كتابة كثيرة على أي حال

وهذا هدر.

## كفى حديثًا، حان وقت الكود

على خلاف الفصول الأخرى، ستحتاج إلى تثبيت [Docker](https://www.docker.com) لأننا سنشغّل تطبيقاتنا في حاويات (containers). ومن المفترض في هذه المرحلة من الكتاب أنك مرتاح في كتابة كود Go والاستيراد من حزم مختلفة وغير ذلك.

أنشئ مشروعًا جديدًا بالأمر `go mod init github.com/quii/go-specs-greet` (يمكنك وضع أي شيء تريده هنا، لكن إن غيّرت المسار فستحتاج إلى تغيير كل الاستيرادات الداخلية لتطابقه)

أنشئ مجلدًا باسم `specifications` ليحتوي مواصفاتنا، وأضف إليه ملفًا باسم `greet.go`

```go
package specifications

import (
	"testing"

	"github.com/alecthomas/assert/v2"
)

type Greeter interface {
	Greet() (string, error)
}

func GreetSpecification(t testing.TB, greeter Greeter) {
	got, err := greeter.Greet()
	assert.NoError(t, err)
	assert.Equal(t, got, "Hello, world")
}
```

يتولى محرري (Goland) عناء إضافة الاعتماديات عني، لكن إن احتجت إلى فعل ذلك يدويًا فستكتب

`go get github.com/alecthomas/assert/v2`

بناءً على تصميم اختبارات القبول عند Farley (مواصفة ← DSL ← driver ← نظام)، أصبحت لدينا مواصفة مفصولة عن التنفيذ. فهي لا تعرف ولا تهتم _بكيفية_ تنفيذ `Greet`؛ بل تكتفي بالتعقيد الجوهري في مجالنا. ونسلّم بأن هذا التعقيد ليس كبيرًا الآن، لكننا سنوسّع المواصفة لإضافة مزيد من الوظائف مع مواصلة التكرار. ومن المهم دائمًا أن نبدأ صغارًا!

ويمكنك أن ترى في الـ interface خطوتنا الأولى نحو DSL؛ ومع نمو المشروع قد تجد حاجة إلى تجريد مختلف، لكن هذا يكفي الآن.

في هذه المرحلة، قد يدفع هذا القدر من الشكليات لفصل مواصفتنا عن التنفيذ بعض الناس إلى اتهامنا بـ"الإفراط في التجريد". **وأعدك أن اختبارات القبول المرتبطة بالتنفيذ ارتباطًا مفرطًا تصبح عبئًا حقيقيًا على فرق الهندسة**. وأنا واثق أن معظم اختبارات القبول في الميدان مكلفة الصيانة بسبب هذا الارتباط غير المناسب؛ لا العكس المتمثل في الإفراط في التجريد.

يمكننا استخدام هذه المواصفة للتحقق من أي "نظام" قادر على `Greet`.

### النظام الأول: واجهة HTTP البرمجية

نحتاج إلى تقديم "خدمة تحية" عبر HTTP. لذا سنحتاج إلى إنشاء:

1. **driver**. وفي حالتنا هذه، يتعامل المرء مع نظام HTTP باستخدام **عميل HTTP**. سيعرف هذا الكود كيف يتعامل مع واجهتنا البرمجية. فالـ drivers تترجم الـ DSLs إلى استدعاءات خاصة بالنظام؛ وفي حالتنا، سينفّذ الـ driver الـ interface الذي تحدده المواصفات.
2. **خادم HTTP** فيه واجهة برمجية للتحية
3. **اختبار**، وهو المسؤول عن إدارة دورة حياة تشغيل الخادم، ثم وصل الـ driver بالمواصفة لتشغيلها كاختبار

## اكتب الاختبار أولًا

قد تكون العملية الأولية لإنشاء اختبار صندوق أسود يترجم برنامجك ويشغّله، ثم ينفّذ الاختبار وينظّف كل شيء بعده، شاقة إلى حد بعيد. ولهذا يُفضَّل القيام بها في بداية مشروعك وبأقل وظائف ممكنة. فأنا عادةً أبدأ كل مشاريعي بتطبيق خادم "hello world"، مع تجهيز كل اختباراتي واستعدادها لأبني الوظائف الفعلية بسرعة.

قد يستغرق النموذج الذهني لـ"المواصفات" و"الـ drivers" و"اختبارات القبول" بعض الوقت لتعتاد عليه، لذا تابع بعناية. وقد يفيدك "العمل بالاتجاه المعاكس" بمحاولة استدعاء المواصفة أولًا.

أنشئ بنية ما لتحتوي البرنامج الذي نعتزم إطلاقه.

`mkdir -p cmd/httpserver`

داخل المجلد الجديد، أنشئ ملفًا جديدًا باسم `greeter_server_test.go`، وأضف ما يلي.

```go
package main_test

import (
	"testing"

	"github.com/quii/go-specs-greet/specifications"
)

func TestGreeterServer(t *testing.T) {
	specifications.GreetSpecification(t, nil)
}
```

نريد تشغيل مواصفتنا في اختبار Go. ولدينا بالفعل وصول إلى `*testing.T`، فهذا هو الوسيط الأول، لكن ماذا عن الثاني؟

‏`specifications.Greeter` هو interface، وسننفّذه بـ `Driver` بتغيير كود `TestGreeterServer` الجديد إلى ما يلي:

```go
import (
	go_specs_greet "github.com/quii/go-specs-greet"
)

func TestGreeterServer(t *testing.T) {
	driver := go_specs_greet.Driver{BaseURL: "http://localhost:8080"}
	specifications.GreetSpecification(t, driver)
}
```

من الأفضل أن يكون الـ `Driver` لدينا قابلًا للتهيئة ليعمل ضد بيئات مختلفة، ومنها المحلية، لذا أضفنا حقل `BaseURL`.

## جرّب تشغيل الاختبار

```
./greeter_server_test.go:46:12: undefined: go_specs_greet.Driver
```

ما زلنا نمارس TDD هنا! إنها خطوة أولى كبيرة علينا القيام بها؛ نحتاج إلى إنشاء بضعة ملفات وكتابة كود أكثر ربما مما اعتدنا عليه، لكن هذا هو الحال غالبًا عند البداية الأولى. ومن المهم جدًا أن نتذكّر قواعد الخطوة الحمراء.

> ارتكب من الذنوب ما يلزم لنجاح الاختبار

## اكتب أصغر قدر من الكود ليعمل الاختبار وافحص مخرجاته الفاشلة

أمسك أنفك؛ تذكّر أننا نستطيع إعادة الهيكلة بعد نجاح الاختبار. وهذا كود الـ driver في `driver.go` الذي سنضعه في جذر المشروع:

```go
package go_specs_greet

import (
	"io"
	"net/http"
)

type Driver struct {
	BaseURL string
}

func (d Driver) Greet() (string, error) {
	res, err := http.Get(d.BaseURL + "/greet")
	if err != nil {
		return "", err
	}
	defer res.Body.Close()
	greeting, err := io.ReadAll(res.Body)
	if err != nil {
		return "", err
	}
	return string(greeting), nil
}
```

ملاحظات:

- قد تقول إنه كان ينبغي أن أكتب اختبارات لاستخلاص مختلف حالات `if err != nil`، لكن بحسب خبرتي، ما دمت لا تفعل شيئًا بـ `err`، فإن الاختبارات التي تقول "تُعيد الخطأ الذي تستقبله" ذات قيمة منخفضة نسبيًا.
- **لا ينبغي أن تستخدم عميل HTTP الافتراضي**. سنمرر لاحقًا عميل HTTP لتهيئته بمُهَل زمنية (timeouts) وغيرها، لكننا الآن نحاول فقط الوصول إلى اختبار ناجح.
- في `greeter_server_test.go` استدعينا دالة Driver من حزمة `go_specs_greet` التي أنشأناها الآن، ولا تنسَ إضافة `github.com/quii/go-specs-greet` إلى استيراداتها.
جرّب إعادة تشغيل الاختبارات؛ ينبغي أن تُترجم الآن لكن ألا تنجح.

```
Get "http://localhost:8080/greet": dial tcp [::1]:8080: connect: connection refused
```

لدينا `Driver`، لكننا لم نشغّل تطبيقنا بعد، لذا لا يستطيع تنفيذ طلب HTTP. ونحتاج إلى أن ينسّق اختبار القبول لدينا مهمة بناء نظامنا وتشغيله ثم إنهائه أخيرًا لينفّذ الاختبار.

### تشغيل تطبيقنا

من الشائع أن تبني الفرق صور Docker لأنظمتها لتُنشر، لذا سنفعل الشيء نفسه في اختبارنا

لمساعدتنا على استخدام Docker في اختباراتنا، سنستخدم [Testcontainers](https://golang.testcontainers.org). فهي تمنحنا طريقة برمجية لبناء صور Docker وإدارة دورات حياة الحاويات.

`go get github.com/testcontainers/testcontainers-go`

يمكنك الآن تعديل `cmd/httpserver/greeter_server_test.go` ليصبح كما يلي:

```go
package main_test

import (
	"context"
	"testing"

	"github.com/alecthomas/assert/v2"
	go_specs_greet "github.com/quii/go-specs-greet"
	"github.com/quii/go-specs-greet/specifications"
	"github.com/testcontainers/testcontainers-go"
	"github.com/testcontainers/testcontainers-go/wait"
)

func TestGreeterServer(t *testing.T) {
	ctx := context.Background()

	req := testcontainers.ContainerRequest{
		FromDockerfile: testcontainers.FromDockerfile{
			Context:    "../../.",
			Dockerfile: "./cmd/httpserver/Dockerfile",
			// set to false if you want less spam, but this is helpful if you're having troubles
			PrintBuildLog: true,
		},
		ExposedPorts: []string{"8080:8080"},
		WaitingFor:   wait.ForHTTP("/").WithPort("8080"),
	}
	container, err := testcontainers.GenericContainer(ctx, testcontainers.GenericContainerRequest{
		ContainerRequest: req,
		Started:          true,
	})
	assert.NoError(t, err)
	t.Cleanup(func() {
		assert.NoError(t, container.Terminate(ctx))
	})

	driver := go_specs_greet.Driver{BaseURL: "http://localhost:8080"}
	specifications.GreetSpecification(t, driver)
}
```

جرّب تشغيل الاختبار.

```
=== RUN   TestGreeterHandler
2022/09/10 18:49:44 Starting container id: 03e8588a1be4 image: docker.io/testcontainers/ryuk:0.3.3
2022/09/10 18:49:45 Waiting for container id 03e8588a1be4 image: docker.io/testcontainers/ryuk:0.3.3
2022/09/10 18:49:45 Container is ready id: 03e8588a1be4 image: docker.io/testcontainers/ryuk:0.3.3
    greeter_server_test.go:32: Did not expect an error but got:
        Error response from daemon: Cannot locate specified Dockerfile: ./cmd/httpserver/Dockerfile: failed to create container
--- FAIL: TestGreeterHandler (0.59s)
```

نحتاج إلى إنشاء ملف Dockerfile لبرنامجنا. داخل مجلد `httpserver`، أنشئ ملفًا باسم `Dockerfile` وأضف ما يلي.

```dockerfile
# Make sure to specify the same Go version as the one in the go.mod file.
# For example, golang:1.22.1-alpine.
FROM golang:1.18-alpine

WORKDIR /app

COPY go.mod ./

RUN go mod download

COPY . .

RUN go build -o svr cmd/httpserver/*.go

EXPOSE 8080
CMD [ "./svr" ]
```

لا تقلق كثيرًا بشأن التفاصيل هنا؛ يمكن صقلها وتحسينها، لكنها ستكفي في هذا المثال. وميزة نهجنا هنا أننا نستطيع لاحقًا تحسين ملف Dockerfile لدينا، ولدينا اختبار يثبت أنه يعمل كما نريد. وهذه قوة حقيقية لامتلاك اختبارات صندوق أسود!

جرّب إعادة تشغيل الاختبار؛ ينبغي أن يشتكي من عدم قدرته على بناء الصورة. وهذا بالطبع لأننا لم نكتب برنامجًا لبنائه بعد!

لكي ينفّذ الاختبار بالكامل، سنحتاج إلى إنشاء برنامج يستمع على `8080`، لكن **هذا كل شيء**. التزم بانضباط TDD، ولا تكتب كود الإنتاج الذي سيجعل الاختبار ينجح حتى نتحقق من أن الاختبار يفشل كما نتوقع.

أنشئ ملف `main.go` داخل مجلد `httpserver` بما يلي

```go
package main

import (
	"log"
	"net/http"
)

func main() {
	handler := http.HandlerFunc(func(writer http.ResponseWriter, request *http.Request) {
	})
	if err := http.ListenAndServe(":8080", handler); err != nil {
		log.Fatal(err)
	}
}
```

جرّب تشغيل الاختبار مرة أخرى، وينبغي أن يفشل بما يلي.

```
    greet.go:16: Expected values to be equal:
        +Hello, World
        \ No newline at end of file
--- FAIL: TestGreeterHandler (2.09s)
```

## اكتب كودًا كافيًا لنجاح الاختبار

حدّث الـ handler ليتصرف كما تريد مواصفتنا

```go
import (
	"fmt"
	"log"
	"net/http"
)

func main() {
	handler := http.HandlerFunc(func(w http.ResponseWriter, _ *http.Request) {
		fmt.Fprint(w, "Hello, world")
	})
	if err := http.ListenAndServe(":8080", handler); err != nil {
		log.Fatal(err)
	}
}
```

## إعادة الهيكلة

مع أن هذا ليس إعادة هيكلة من الناحية التقنية، لا ينبغي أن نعتمد على عميل HTTP الافتراضي، فلنغيّر الـ Driver لدينا لنتمكن من تمرير عميل، وهو ما سيمنحه اختبارنا.

```go
import (
	"io"
	"net/http"
)

type Driver struct {
	BaseURL string
	Client  *http.Client
}

func (d Driver) Greet() (string, error) {
	res, err := d.Client.Get(d.BaseURL + "/greet")
	if err != nil {
		return "", err
	}
	defer res.Body.Close()
	greeting, err := io.ReadAll(res.Body)
	if err != nil {
		return "", err
	}
	return string(greeting), nil
}
```

وفي اختبارنا في `cmd/httpserver/greeter_server_test.go`، حدّث إنشاء الـ driver ليمرر عميلًا.

```go
client := http.Client{
	Timeout: 1 * time.Second,
}

driver := go_specs_greet.Driver{BaseURL: "http://localhost:8080", Client: &client}
specifications.GreetSpecification(t, driver)
```

من الممارسات الجيدة أن تُبقي `main.go` بسيطًا قدر الإمكان؛ إذ ينبغي أن يهتم فقط بجمع اللبنات التي تصنعها في تطبيق واحد.

أنشئ ملفًا في جذر المشروع باسم `handler.go` وانقل كودنا إليه.

```go
package go_specs_greet

import (
	"fmt"
	"net/http"
)

func Handler(w http.ResponseWriter, r *http.Request) {
	fmt.Fprint(w, "Hello, world")
}
```

حدّث `main.go` ليستورد الـ handler ويستخدمه بدلًا من ذلك.

```go
package main

import (
	"net/http"

	go_specs_greet "github.com/quii/go-specs-greet"
)

func main() {
	handler := http.HandlerFunc(go_specs_greet.Handler)
	http.ListenAndServe(":8080", handler)
}
```

## تأمّل

بدت الخطوة الأولى وكأنها جهد شاق. فقد أنشأنا عدة ملفات `go` لإنشاء معالج HTTP يعيد نصًا ثابتًا واختباره. وهذا الإعداد وهذه الشكليات في "التكرار صفر" ستفيدنا كثيرًا في التكرارات التالية.

ينبغي أن يكون تغيير الوظائف بسيطًا ومحكومًا بدفعه عبر المواصفة والتعامل مع أي تغييرات يفرضها علينا. والآن بعد أن أصبح `DockerFile` و`testcontainers` مهيأين لاختبار القبول لدينا، لا ينبغي أن نضطر إلى تغيير هذين الملفين إلا إذا تغيّرت طريقة تركيب تطبيقنا.

وسنرى ذلك في متطلبنا التالي: تحية شخص بعينه.

## اكتب الاختبار أولًا

عدّل مواصفتنا

```go
package specifications

import (
	"testing"

	"github.com/alecthomas/assert/v2"
)

type Greeter interface {
	Greet(name string) (string, error)
}

func GreetSpecification(t testing.TB, greeter Greeter) {
	got, err := greeter.Greet("Mike")
	assert.NoError(t, err)
	assert.Equal(t, got, "Hello, Mike")
}
```

للسماح لنا بتحية أشخاص بعينهم، نحتاج إلى تغيير الـ interface الخاص بنظامنا ليقبل وسيط `name`.

## جرّب تشغيل الاختبار

```
./greeter_server_test.go:48:39: cannot use driver (variable of type go_specs_greet.Driver) as type specifications.Greeter in argument to specifications.GreetSpecification:
	go_specs_greet.Driver does not implement specifications.Greeter (wrong type for Greet method)
		have Greet() (string, error)
		want Greet(name string) (string, error)
```

أدى التغيير في المواصفة إلى حاجة الـ driver لدينا إلى تحديث.

## اكتب أصغر قدر من الكود ليعمل الاختبار وافحص مخرجاته الفاشلة

حدّث الـ driver ليمرر قيمة استعلام (query value) باسم `name` في الطلب لطلب تحية اسم بعينه.

```go
import "io"

func (d Driver) Greet(name string) (string, error) {
	res, err := d.Client.Get(d.BaseURL + "/greet?name=" + name)
	if err != nil {
		return "", err
	}
	defer res.Body.Close()
	greeting, err := io.ReadAll(res.Body)
	if err != nil {
		return "", err
	}
	return string(greeting), nil
}
```

ينبغي أن يعمل الاختبار الآن، وأن يفشل.

```
    greet.go:16: Expected values to be equal:
        -Hello, world
        \ No newline at end of file
        +Hello, Mike
        \ No newline at end of file
--- FAIL: TestGreeterHandler (1.92s)
```

## اكتب كودًا كافيًا لنجاح الاختبار

استخرج `name` من الطلب وقم بالتحية.

```go
import (
	"fmt"
	"net/http"
)

func Handler(w http.ResponseWriter, r *http.Request) {
	fmt.Fprintf(w, "Hello, %s", r.URL.Query().Get("name"))
}
```

ينبغي أن ينجح الاختبار الآن.

## إعادة الهيكلة

في فصل [إعادة النظر في معالجات HTTP،](../questions-and-answers/http-handlers-revisited.md) ناقشنا أهمية أن تكون معالجات HTTP مسؤولة فقط عن التعامل مع اهتمامات HTTP؛ وأن أي "منطق مجال" ينبغي أن يعيش خارج الـ handler. وهذا يتيح لنا تطوير منطق المجال بمعزل عن HTTP، ما يجعله أسهل في الاختبار والفهم.

فلنفصل هذين الاهتمامين عن بعضهما.

حدّث الـ handler لدينا في `./handler.go` كما يلي:

```go
func Handler(w http.ResponseWriter, r *http.Request) {
	name := r.URL.Query().Get("name")
	fmt.Fprint(w, Greet(name))
}
```

أنشئ ملفًا جديدًا `./greet.go`:

```go
package go_specs_greet

import "fmt"

func Greet(name string) string {
	return fmt.Sprintf("Hello, %s", name)
}
```

## استطراد بسيط إلى نمط التصميم "المهايئ" (adapter)

الآن بعد أن فصلنا منطق مجالنا الخاص بتحية الناس في دالة منفصلة، صرنا أحرارًا في كتابة اختبارات وحدة لدالة التحية. وهذا بلا شك أبسط بكثير من اختبارها عبر مواصفة تمرّ بـ driver يصل إلى خادم ويب، فقط ليحصل على نص!

ألن يكون جميلًا لو استطعنا إعادة استخدام مواصفتنا هنا أيضًا؟ فهدف المواصفة، بعد كل شيء، أن تكون مفصولة عن تفاصيل التنفيذ. وإذا كانت المواصفة تلتقط **تعقيدنا الجوهري** وكان كود "المجال" لدينا يفترض أن ينمذجه، فينبغي أن نستطيع استخدامهما معًا.

فلنجرّب ذلك بإنشاء `./greet_test.go` كما يلي:

```go
package go_specs_greet_test

import (
	"testing"

	go_specs_greet "github.com/quii/go-specs-greet"
	"github.com/quii/go-specs-greet/specifications"
)

func TestGreet(t *testing.T) {
	specifications.GreetSpecification(t, go_specs_greet.Greet)
}

```

سيكون هذا جميلًا، لكنه لا يعمل

```
./greet_test.go:11:39: cannot use go_specs_greet.Greet (value of type func(name string) string) as type specifications.Greeter in argument to specifications.GreetSpecification:
	func(name string) string does not implement specifications.Greeter (missing Greet method)
```

مواصفتنا تريد شيئًا له method اسمه `Greet()` لا دالة.

خطأ الترجمة محبط؛ فلدينا شيء "نعلم" أنه `Greeter`، لكنه ليس بالـ**شكل** المناسب تمامًا ليسمح لنا المترجم باستخدامه. وهذا ما يعالجه نمط الـ **adapter**.

> في [هندسة البرمجيات](https://en.wikipedia.org/wiki/Software_engineering)، نمط **المهايئ** (adapter pattern) هو [نمط تصميم برمجي](https://en.wikipedia.org/wiki/Software_design_pattern) (يُعرف أيضًا باسم [wrapper](https://en.wikipedia.org/wiki/Wrapper_function)، وهي تسمية بديلة يتشاركها مع [نمط decorator](https://en.wikipedia.org/wiki/Decorator_pattern)) يسمح باستخدام [واجهة](https://en.wikipedia.org/wiki/Interface_(computer_science)) [فئة](https://en.wikipedia.org/wiki/Class_(computer_science)) موجودة كواجهة أخرى.[[1\]](https://en.wikipedia.org/wiki/Adapter_pattern#cite_note-HeadFirst-1) ويُستخدم غالبًا لجعل الفئات الموجودة تعمل مع غيرها دون تعديل [كودها المصدري](https://en.wikipedia.org/wiki/Source_code).

كلمات كثيرة فخمة لشيء بسيط نسبيًا، وهذا حال أنماط التصميم غالبًا، ولهذا يرفع الناس أعينهم تأففًا منها. لا تكمن قيمة أنماط التصميم في تطبيقات محددة، بل في كونها لغة لوصف حلول محددة لمشكلات شائعة يواجهها المهندسون. وإذا كان لديك فريق يتشارك مفردات مشتركة، فإن ذلك يقلل الاحتكاك في التواصل.

أضف هذا الكود في `./specifications/adapters.go`

```go
type GreetAdapter func(name string) string

func (g GreetAdapter) Greet(name string) (string, error) {
	return g(name), nil
}
```

يمكننا الآن استخدام المهايئ لدينا في اختبارنا لوصل دالة `Greet` بمواصفتنا.

```go
package go_specs_greet_test

import (
	"testing"

	gospecsgreet "github.com/quii/go-specs-greet"
	"github.com/quii/go-specs-greet/specifications"
)

func TestGreet(t *testing.T) {
	specifications.GreetSpecification(
		t,
		specifications.GreetAdapter(gospecsgreet.Greet),
	)
}
```

يكون نمط الـ adapter مفيدًا عندما يكون لديك نوع يُظهر السلوك الذي تريده واجهة ما، لكنه ليس بالشكل المناسب.

## تأمّل

بدا تغيير السلوك بسيطًا، أليس كذلك؟ حسنًا، ربما كان ذلك بسبب طبيعة المشكلة فحسب، لكن طريقة العمل هذه تمنحك انضباطًا وطريقة بسيطة قابلة للتكرار لتغيير نظامك من أعلاه إلى أسفله:

- حلّل مشكلتك وحدّد تحسينًا طفيفًا في نظامك يدفعك في الاتجاه الصحيح
- التقط التعقيد الجوهري الجديد في مواصفة
- تابع أخطاء الترجمة حتى يعمل اختبار القبول
- حدّث تنفيذك ليتصرف النظام وفق المواصفة
- أعد الهيكلة

بعد ألم التكرار الأول، لم نضطر إلى تعديل كود اختبار القبول لدينا لأننا نفصل المواصفات والـ drivers والتنفيذ. فتغيير مواصفتنا تطلّب منا تحديث الـ driver ثم التنفيذ أخيرًا، أما الكود المتكرر الخاص بـ_كيفية_ تشغيل النظام كحاوية فلم يتأثر.

حتى مع كلفة بناء صورة Docker لتطبيقنا وتشغيل الحاوية، فحلقة التغذية الراجعة لاختبار تطبيقنا **بالكامل** ضيقة جدًا:

```
quii@Chriss-MacBook-Pro go-specs-greet % go test ./...
ok  	github.com/quii/go-specs-greet	0.181s
ok  	github.com/quii/go-specs-greet/cmd/httpserver	2.221s
?   	github.com/quii/go-specs-greet/specifications	[no test files]
```

الآن، تخيّل أن مديرة التكنولوجيا لديك (CTO) قررت أن gRPC هو _المستقبل_. إنها تريدك أن تعرض الوظائف نفسها عبر خادم gRPC مع الحفاظ على خادم HTTP الحالي.

هذا مثال على **التعقيد العرضي**. وتذكّر أن التعقيد العرضي هو التعقيد الذي نضطر إلى التعامل معه لأننا نعمل مع الحواسيب، أشياء مثل الشبكات والأقراص وواجهات البرمجة وغيرها. **والتعقيد الجوهري لم يتغيّر**، لذا لا ينبغي أن نضطر إلى تغيير مواصفاتنا.

كثير من بنى المستودعات وأنماط التصميم تتعامل أساسًا مع فصل أنواع التعقيد. فعلى سبيل المثال، يطلب نمط "المنافذ والمهايئات" (ports and adapters) أن تفصل كود مجالك عن أي شيء له علاقة بالتعقيد العرضي؛ فهذا الكود يعيش في مجلد "adapters".

### تهيئة التغيير بسهولة

أحيانًا يكون من المنطقي القيام ببعض إعادة الهيكلة _قبل_ إجراء تغيير.

> اجعل التغيير سهلًا أولًا، ثم أجرِ التغيير السهل

~Kent Beck

لهذا السبب، لننقل كود `http` لدينا - `driver.go` و`handler.go` - إلى حزمة اسمها `httpserver` داخل مجلد `adapters`، ونغيّر اسم حزمتيهما إلى `httpserver`.

ستحتاج الآن إلى استيراد الحزمة الجذرية في `handler.go` للإشارة إلى method الـ Greet...

```go
package httpserver

import (
	"fmt"
	"net/http"

	go_specs_greet "github.com/quii/go-specs-greet/domain/interactions"
)

func Handler(w http.ResponseWriter, r *http.Request) {
	name := r.URL.Query().Get("name")
	fmt.Fprint(w, go_specs_greet.Greet(name))
}

```

استورد مهايئ httpserver لديك في main.go:

```go
package main

import (
	"net/http"

	"github.com/quii/go-specs-greet/adapters/httpserver"
)

func main() {
	handler := http.HandlerFunc(httpserver.Handler)
	http.ListenAndServe(":8080", handler)
}
```

وحدّث الاستيراد والإشارة إلى `Driver` في greeter_server_test.go:

```go
driver := httpserver.Driver{BaseURL: "http://localhost:8080", Client: &client}
```

وأخيرًا، من المفيد جمع كود مستوى المجال لديك في مجلد خاص به أيضًا. لا تكن كسولًا وتضع في مشاريعك مجلد `domain` يضم مئات الأنواع والدوال غير المترابطة. ابذل جهدًا في التفكير في مجالك، وضمّ الأفكار التي تنتمي إلى بعضها إلى بعض. وهذا سيجعل مشروعك أسهل في الفهم وسيحسّن جودة استيراداتك.

بدلًا من رؤية

```go
domain.Greet
```

وهو أمر غريب بعض الشيء، فضّل

```go
interactions.Greet
```

أنشئ مجلد `domain` ليحتوي كل كود مجالك، وداخله مجلد `interactions`. وقد تحتاج، بحسب أدواتك، إلى تحديث بعض الاستيرادات والكود.

ينبغي أن تبدو شجرة مشروعنا الآن هكذا:

```
quii@Chriss-MacBook-Pro go-specs-greet % tree
.
├── Makefile
├── README.md
├── adapters
│   └── httpserver
│       ├── driver.go
│       └── handler.go
├── cmd
│   └── httpserver
|       ├── Dockerfile
│       ├── greeter_server_test.go
│       └── main.go
├── domain
│   └── interactions
│       ├── greet.go
│       └── greet_test.go
├── go.mod
├── go.sum
└── specifications
    └── adapters.go
    └── greet.go

```

كود مجالنا، أي **التعقيد الجوهري**، يعيش في جذر وحدة go لدينا، أما الكود الذي سيتيح لنا استخدامه في "العالم الحقيقي" فمنظَّم في **مهايئات** (adapters). ومجلد `cmd` هو حيث نركّب هذه التجميعات المنطقية في تطبيقات عملية، لها اختبارات صندوق أسود تتحقق من أن كل شيء يعمل. جميل!

وأخيرًا، يمكننا إجراء قليل _جدًا_ من الترتيب في اختبار القبول لدينا. وإذا تأملت الخطوات عالية المستوى في اختبار القبول:

- بناء صورة docker
- الانتظار حتى يستمع على _بعض_ المنافذ
- إنشاء driver يفهم كيفية ترجمة الـ DSL إلى استدعاءات خاصة بالنظام
- وصل الـ driver بالمواصفة

... ستدرك أن لدينا المتطلبات نفسها لاختبار قبول لخادم gRPC!

يبدو مجلد `adapters` مكانًا جيدًا مثل أي مكان آخر، لذا داخل ملف باسم `docker.go`، غلّف الخطوتين الأوليين في دالة سنعيد استخدامها لاحقًا.

```go
package adapters

import (
	"context"
	"fmt"
	"testing"
	"time"

	"github.com/alecthomas/assert/v2"
	"github.com/docker/go-connections/nat"
	"github.com/testcontainers/testcontainers-go"
	"github.com/testcontainers/testcontainers-go/wait"
)

func StartDockerServer(
	t testing.TB,
	port string,
	dockerFilePath string,
) {
	ctx := context.Background()
	t.Helper()
	req := testcontainers.ContainerRequest{
		FromDockerfile: testcontainers.FromDockerfile{
			Context:       "../../.",
			Dockerfile:    dockerFilePath,
			PrintBuildLog: true,
		},
		ExposedPorts: []string{fmt.Sprintf("%s:%s", port, port)},
		WaitingFor:   wait.ForListeningPort(nat.Port(port)).WithStartupTimeout(5 * time.Second),
	}
	container, err := testcontainers.GenericContainer(ctx, testcontainers.GenericContainerRequest{
		ContainerRequest: req,
		Started:          true,
	})
	assert.NoError(t, err)
	t.Cleanup(func() {
		assert.NoError(t, container.Terminate(ctx))
	})
}
```

وهذا يمنحنا فرصة لترتيب اختبار القبول لدينا قليلًا

```go
func TestGreeterServer(t *testing.T) {
	var (
		port           = "8080"
		dockerFilePath = "./cmd/httpserver/Dockerfile"
		baseURL        = fmt.Sprintf("http://localhost:%s", port)
		driver         = httpserver.Driver{BaseURL: baseURL, Client: &http.Client{
			Timeout: 1 * time.Second,
		}}
	)

	adapters.StartDockerServer(t, port, dockerFilePath)
	specifications.GreetSpecification(t, driver)
}
```

وهذا ينبغي أن يجعل كتابة الاختبار _التالي_ أبسط.

## اكتب الاختبار أولًا

يمكن إنجاز هذه الوظيفة الجديدة بإنشاء `adapter` جديد يتفاعل مع كود مجالنا. ولهذا السبب:

- يجب ألا نضطر إلى تغيير المواصفة؛
- ينبغي أن نستطيع إعادة استخدام المواصفة؛
- ينبغي أن نستطيع إعادة استخدام كود المجال.

أنشئ مجلدًا جديدًا باسم `grpcserver` داخل `cmd` ليحتوي برنامجنا الجديد واختبار القبول المقابل له. وداخل `cmd/grpc_server/greeter_server_test.go`، أضف اختبار قبول يشبه كثيرًا اختبار خادم HTTP لدينا، ليس مصادفة بل بحكم التصميم.

```go
package main_test

import (
	"fmt"
	"testing"

	"github.com/quii/go-specs-greet/adapters"
	"github.com/quii/go-specs-greet/adapters/grpcserver"
	"github.com/quii/go-specs-greet/specifications"
)

func TestGreeterServer(t *testing.T) {
	var (
		port           = "50051"
		dockerFilePath = "./cmd/grpcserver/Dockerfile"
		driver         = grpcserver.Driver{Addr: fmt.Sprintf("localhost:%s", port)}
	)

	adapters.StartDockerServer(t, port, dockerFilePath)
	specifications.GreetSpecification(t, &driver)
}
```

الفروق الوحيدة هي:

- نستخدم ملف docker مختلفًا، لأننا نبني برنامجًا مختلفًا
- وهذا يعني أننا سنحتاج إلى `Driver` جديد يستخدم `gRPC` للتفاعل مع برنامجنا الجديد

## جرّب تشغيل الاختبار

```
./greeter_server_test.go:26:12: undefined: grpcserver
```

لم ننشئ `Driver` بعد، لذا لن تُترجم.

## اكتب أصغر قدر من الكود ليعمل الاختبار وافحص مخرجاته الفاشلة

أنشئ مجلدًا باسم `grpcserver` داخل `adapters`، وأنشئ داخله `driver.go`

```go
package grpcserver

type Driver struct {
	Addr string
}

func (d Driver) Greet(name string) (string, error) {
	return "", nil
}
```

إذا شغّلت مرة أخرى، ينبغي أن _تترجم_ الآن لكن ألا تنجح، لأننا لم ننشئ Dockerfile ولا البرنامج المقابل لتشغيله.

أنشئ ملف `Dockerfile` جديدًا داخل `cmd/grpcserver`.

```dockerfile
# Make sure to specify the same Go version as the one in the go.mod file.
FROM golang:1.18-alpine

WORKDIR /app

COPY go.mod ./

RUN go mod download

COPY . .

RUN go build -o svr cmd/grpcserver/*.go

EXPOSE 50051
CMD [ "./svr" ]
```

وملف `main.go`

```go
package main

import "fmt"

func main() {
	fmt.Println("implement me")
}
```

ينبغي أن تجد الآن أن الاختبار يفشل لأن خادمنا لا يستمع على المنفذ. والآن حان وقت بدء بناء عميلنا وخادمنا باستخدام gRPC.

## اكتب كودًا كافيًا لنجاح الاختبار

### gRPC

إن لم تكن معتادًا على gRPC، فسأبدأ بإلقاء نظرة على [موقع gRPC](https://grpc.io). ومع ذلك، فهو في هذا الفصل مجرد نوع آخر من المهايئات إلى نظامنا، طريقة تتيح لأنظمة أخرى استدعاء (**r**emote **p**rocedure **c**all) كود مجالنا الممتاز.

والفكرة المختلفة هنا أنك تعرّف "تعريف خدمة" (service definition) باستخدام Protocol Buffers، ثم تولّد كود الخادم والعميل من هذا التعريف. وهذا لا يعمل مع Go فحسب بل مع معظم اللغات الشائعة أيضًا. ويعني ذلك أنك تستطيع مشاركة التعريف مع فرق أخرى في شركتك قد لا تكتب Go أصلًا، وتبقى التواصل بين الخدمات سلسًا.

إن لم تكن قد استخدمت gRPC من قبل، فستحتاج إلى تثبيت **مترجم Protocol buffer** وبعض **إضافات Go**. [ويحتوي موقع gRPC على تعليمات واضحة للقيام بذلك](https://grpc.io/docs/languages/go/quickstart/).

داخل المجلد نفسه الذي فيه driver الجديد، أضف ملف `greet.proto` بما يلي

```protobuf
syntax = "proto3";

option go_package = "github.com/quii/adapters/grpcserver";

package grpcserver;

service Greeter {
  rpc Greet (GreetRequest) returns (GreetReply) {}
}

message GreetRequest {
  string name = 1;
}

message GreetReply {
  string message = 1;
}
```

لفهم هذا التعريف، لا تحتاج إلى أن تكون خبيرًا في Protocol Buffers. فنحن نعرّف خدمة بها method اسمها Greet، ثم نصف أنواع الرسائل الداخلة والخارجة.

داخل `adapters/grpcserver` شغّل ما يلي لتوليد كود العميل والخادم

```
protoc --go_out=. --go_opt=paths=source_relative \
    --go-grpc_out=. --go-grpc_opt=paths=source_relative \
    greet.proto
```

إذا نجح الأمر، سيكون لدينا كود مولَّد لاستخدامه. لنبدأ باستخدام كود العميل المولَّد داخل `Driver` لدينا.

```go
package grpcserver

import (
	"context"

	"google.golang.org/grpc"
	"google.golang.org/grpc/credentials/insecure"
)

type Driver struct {
	Addr string
}

func (d Driver) Greet(name string) (string, error) {
	//todo: we shouldn't redial every time we call greet, refactor out when we're green
	conn, err := grpc.Dial(d.Addr, grpc.WithTransportCredentials(insecure.NewCredentials()))
	if err != nil {
		return "", err
	}
	defer conn.Close()

	client := NewGreeterClient(conn)
	greeting, err := client.Greet(context.Background(), &GreetRequest{
		Name: name,
	})
	if err != nil {
		return "", err
	}

	return greeting.Message, nil
}
```

الآن بعد أن أصبح لدينا عميل، نحتاج إلى تحديث `main.go` لإنشاء خادم. وتذكّر أننا في هذه المرحلة نحاول فقط جعل اختبارنا ينجح ولا نقلق بشأن جودة الكود.

```go
package main

import (
	"context"
	"log"
	"net"

	"github.com/quii/go-specs-greet/adapters/grpcserver"
	"google.golang.org/grpc"
)

func main() {
	lis, err := net.Listen("tcp", ":50051")
	if err != nil {
		log.Fatal(err)
	}
	s := grpc.NewServer()
	grpcserver.RegisterGreeterServer(s, &GreetServer{})

	if err := s.Serve(lis); err != nil {
		log.Fatal(err)
	}
}

type GreetServer struct {
	grpcserver.UnimplementedGreeterServer
}

func (g GreetServer) Greet(ctx context.Context, request *grpcserver.GreetRequest) (*grpcserver.GreetReply, error) {
	return &grpcserver.GreetReply{Message: "fixme"}, nil
}
```

لإنشاء خادم gRPC لدينا، علينا تنفيذ الـ interface الذي ولّده لنا

```go
// GreeterServer is the server API for Greeter service.
// All implementations must embed UnimplementedGreeterServer
// for forward compatibility
type GreeterServer interface {
	Greet(context.Context, *GreetRequest) (*GreetReply, error)
	mustEmbedUnimplementedGreeterServer()
}
```

دالة `main` لدينا:

- تستمع على منفذ
- تنشئ `GreetServer` ينفّذ الـ interface، ثم تسجّله لدى `grpcServer.RegisterGreeterServer` مع `grpc.Server`.
- تستخدم الخادم مع المستمع

لن يكون جهدًا إضافيًا هائلًا أن نستدعي كود مجالنا داخل `greetServer.Greet` بدلًا من كتابة `fix-me` مباشرة في الرسالة، لكنني أفضّل تشغيل اختبار القبول أولًا لنرى هل يعمل كل شيء على مستوى النقل (transport) ونتحقق من مخرجات الاختبار الفاشل.

```
greet.go:16: Expected values to be equal:
-fixme
\ No newline at end of file
+Hello, Mike
\ No newline at end of file
```

جميل! نرى أن driver لدينا قادر على الاتصال بخادم gRPC في الاختبار.

الآن، استدعِ كود مجالنا داخل `GreetServer` لدينا

```go
type GreetServer struct {
	grpcserver.UnimplementedGreeterServer
}

func (g GreetServer) Greet(ctx context.Context, request *grpcserver.GreetRequest) (*grpcserver.GreetReply, error) {
	return &grpcserver.GreetReply{Message: interactions.Greet(request.Name)}, nil
}
```

وأخيرًا، ينجح! لدينا اختبار قبول يثبت أن خادم التحية gRPC لدينا يتصرف كما نريد.

## إعادة الهيكلة

اقترفنا عدة أخطاء لنجاح الاختبار، لكن بعد أن أصبح ناجحًا صار لدينا شبكة الأمان لإعادة الهيكلة.

### تبسيط main

كما في السابق، لا نريد أن يحتوي `main` على كود كثير. يمكننا نقل `GreetServer` الجديد إلى `adapters/grpcserver` فهناك ينبغي أن يعيش. ومن حيث التماسك (cohesion)، إذا غيّرنا تعريف الخدمة، نريد أن يكون "نطاق تأثير" التغيير محصورًا في ذلك الجزء من كودنا.

### لا تُعد الاتصال في driver لدينا في كل مرة

لدينا اختبار واحد فقط، لكن إن وسّعنا مواصفتنا (وسنفعل)، فلا معنى لأن يعيد الـ Driver الاتصال في كل استدعاء RPC.

```go
package grpcserver

import (
	"context"
	"sync"

	"google.golang.org/grpc"
	"google.golang.org/grpc/credentials/insecure"
)

type Driver struct {
	Addr string

	connectionOnce sync.Once
	conn           *grpc.ClientConn
	client         GreeterClient
}

func (d *Driver) Greet(name string) (string, error) {
	client, err := d.getClient()
	if err != nil {
		return "", err
	}

	greeting, err := client.Greet(context.Background(), &GreetRequest{
		Name: name,
	})
	if err != nil {
		return "", err
	}

	return greeting.Message, nil
}

func (d *Driver) getClient() (GreeterClient, error) {
	var err error
	d.connectionOnce.Do(func() {
		d.conn, err = grpc.Dial(d.Addr, grpc.WithTransportCredentials(insecure.NewCredentials()))
		d.client = NewGreeterClient(d.conn)
	})
	return d.client, err
}
```

هنا نوضح كيف يمكننا استخدام [`sync.Once`](https://pkg.go.dev/sync#Once) لضمان ألا يحاول `Driver` لدينا إنشاء اتصال بخادمنا إلا مرة واحدة.

لنلقِ نظرة على الحالة الحالية لبنية مشروعنا قبل المتابعة.

```
quii@Chriss-MacBook-Pro go-specs-greet % tree
.
├── Makefile
├── README.md
├── adapters
│   ├── docker.go
│   ├── grpcserver
│   │   ├── driver.go
│   │   ├── greet.pb.go
│   │   ├── greet.proto
│   │   ├── greet_grpc.pb.go
│   │   └── server.go
│   └── httpserver
│       ├── driver.go
│       └── handler.go
├── cmd
│   ├── grpcserver
│   │   ├── Dockerfile
│   │   ├── greeter_server_test.go
│   │   └── main.go
│   └── httpserver
│       ├── Dockerfile
│       ├── greeter_server_test.go
│       └── main.go
├── domain
│   └── interactions
│       ├── greet.go
│       └── greet_test.go
├── go.mod
├── go.sum
└── specifications
    └── greet.go
```

- يحتوي `adapters` وحدات وظيفية متماسكة مجموعة معًا
- يحتوي `cmd` تطبيقاتنا واختبارات القبول المقابلة لها
- كودنا مفصول تمامًا عن أي تعقيد عرضي

### توحيد ملف `Dockerfile`

لاحظت على الأرجح أن ملفَّي `Dockerfile` متطابقان تقريبًا فيما عدا مسار الملف التنفيذي الذي نريد بناءه.

تستطيع ملفات `Dockerfile` قبول وسائط تتيح لنا إعادة استخدامها في سياقات مختلفة، وهذا يبدو مثاليًا. يمكننا حذف ملفَّي Dockerfile لدينا والاستعاضة عنهما بواحد في جذر المشروع بما يلي

```dockerfile
# Make sure to specify the same Go version as the one in the go.mod file.
FROM golang:1.18-alpine

WORKDIR /app

ARG bin_to_build

COPY go.mod ./

RUN go mod download

COPY . .

RUN go build -o svr cmd/${bin_to_build}/main.go

CMD [ "./svr" ]
```

سيتعين علينا تحديث دالة `StartDockerServer` لتمرير الوسيط عند بناء الصور

```go
func StartDockerServer(
	t testing.TB,
	port string,
	binToBuild string,
) {
	ctx := context.Background()
	t.Helper()
	req := testcontainers.ContainerRequest{
		FromDockerfile: testcontainers.FromDockerfile{
			Context:    "../../.",
			Dockerfile: "Dockerfile",
			BuildArgs: map[string]*string{
				"bin_to_build": &binToBuild,
			},
			PrintBuildLog: true,
		},
		ExposedPorts: []string{fmt.Sprintf("%s:%s", port, port)},
		WaitingFor:   wait.ForListeningPort(nat.Port(port)).WithStartupTimeout(5 * time.Second),
	}
	container, err := testcontainers.GenericContainer(ctx, testcontainers.GenericContainerRequest{
		ContainerRequest: req,
		Started:          true,
	})
	assert.NoError(t, err)
	t.Cleanup(func() {
		assert.NoError(t, container.Terminate(ctx))
	})
}
```

وأخيرًا، حدّث اختباراتنا لتمرير الصورة المطلوب بناءها (افعل ذلك في الاختبار الآخر وغيّر `grpcserver` إلى `httpserver`).

```go
func TestGreeterServer(t *testing.T) {
	var (
		port   = "50051"
		driver = grpcserver.Driver{Addr: fmt.Sprintf("localhost:%s", port)}
	)

	adapters.StartDockerServer(t, port, "grpcserver")
	specifications.GreetSpecification(t, &driver)
}
```

### فصل أنواع الاختبارات المختلفة

اختبارات القبول رائعة لأنها تختبر أن النظام كله يعمل من وجهة نظر سلوكية خالصة موجهة للمستخدم، لكن لها عيوبًا مقارنة باختبارات الوحدة:

- أبطأ
- جودة التغذية الراجعة ليست مركزة كتركيز اختبار الوحدة غالبًا
- لا تساعدك في الجودة الداخلية ولا في التصميم

يرشدنا [هرم الاختبارات](https://martinfowler.com/articles/practical-test-pyramid.html) إلى نوع المزيج الذي نريده لمجموعة اختباراتنا، وينبغي أن تقرأ مقال Fowler لمزيد من التفاصيل، لكن الملخص المبسّط جدًا لهذا المقال هو "اختبارات وحدة كثيرة واختبارات قبول قليلة".

ولهذا السبب، مع نمو المشروع قد تجد نفسك غالبًا في مواقف تستغرق فيها اختبارات القبول بضع دقائق لتشغيلها. ولتقديم تجربة مطوّر ودودة لمن يستعرضون مشروعك، يمكنك تمكين المطورين من تشغيل أنواع الاختبارات المختلفة منفصلة.

من الأفضل أن يكون تشغيل `go test ./...` ممكنًا دون أي إعداد إضافي من المهندس، عدا بضعة اعتماديات رئيسية مثل مترجم Go (بداهةً) وربما Docker.

توفّر Go آلية للمهندسين لتشغيل الاختبارات "القصيرة" فقط عبر [العلامة short](https://pkg.go.dev/testing#Short)

`go test -short ./...`

يمكننا أن نضيف إلى اختبارات القبول لدينا فحصًا لما إذا كان المستخدم يريد تشغيل اختبارات القبول عبر فحص قيمة العلامة

```go
if testing.Short() {
	t.Skip()
}
```

أنشأت ملف `Makefile` لإظهار هذا الاستخدام

```makefile
build:
	golangci-lint run
	go test ./...

unit-tests:
	go test -short ./...
```

### متى ينبغي أن أكتب اختبارات القبول؟

أفضل ممارسة هي تفضيل اختبارات وحدة كثيرة سريعة واختبارات قبول قليلة، لكن كيف تقرر متى ينبغي أن تكتب اختبار قبول بدلًا من اختبارات الوحدة؟

من الصعب وضع قاعدة محددة، لكن الأسئلة التي أطرحها على نفسي عادةً هي:

- هل هذه حالة حدية (edge case)؟ أفضّل اختبار هذه الحالات باختبارات وحدة
- هل هذا شيء يتحدث عنه غير التقنيين كثيرًا؟ أفضّل أن تكون لدي ثقة كبيرة بأن الشيء الأساسي "يعمل فعلًا"، لذا أضيف اختبار قبول
- هل أصف رحلة مستخدم (user journey) لا دالة بعينها؟ اختبار قبول
- هل ستمنحني اختبارات الوحدة ثقة كافية؟ أحيانًا تأخذ رحلة موجودة لها اختبار قبول بالفعل، لكنك تضيف وظائف أخرى للتعامل مع سيناريوهات مختلفة بسبب مدخلات مختلفة. وفي هذه الحالة، إضافة اختبار قبول آخر تضيف كلفة لكنها تجلب قيمة ضئيلة، لذا أفضّل بعض اختبارات الوحدة.

## التكرار على عملنا

مع كل هذا الجهد، ستأمل أن يصبح توسيع نظامنا الآن بسيطًا. إن صناعة نظام يسهل العمل عليه ليست سهلة بالضرورة، لكنها تستحق الوقت، وهي أسهل بكثير عندما تبدأ مشروعًا.

لنوسّع واجهتنا البرمجية لتشمل وظيفة "الشتم" (curse).

## اكتب الاختبار أولًا

هذا سلوك جديد تمامًا، لذا ينبغي أن نبدأ باختبار قبول. في ملف مواصفاتنا، أضف ما يلي

```go
type MeanGreeter interface {
	Curse(name string) (string, error)
}

func CurseSpecification(t *testing.T, meany MeanGreeter) {
	got, err := meany.Curse("Chris")
	assert.NoError(t, err)
	assert.Equal(t, got, "Go to hell, Chris!")
}
```

اختر أحد اختبارات القبول لدينا وجرّب استخدام المواصفة

```go
func TestGreeterServer(t *testing.T) {
	if testing.Short() {
		t.Skip()
	}
	var (
		port   = "50051"
		driver = grpcserver.Driver{Addr: fmt.Sprintf("localhost:%s", port)}
	)

	t.Cleanup(driver.Close)
	adapters.StartDockerServer(t, port, "grpcserver")
	specifications.GreetSpecification(t, &driver)
	specifications.CurseSpecification(t, &driver)
}
```

## جرّب تشغيل الاختبار

```
# github.com/quii/go-specs-greet/cmd/grpcserver_test [github.com/quii/go-specs-greet/cmd/grpcserver.test]
./greeter_server_test.go:27:39: cannot use &driver (value of type *grpcserver.Driver) as type specifications.MeanGreeter in argument to specifications.CurseSpecification:
	*grpcserver.Driver does not implement specifications.MeanGreeter (missing Curse method)
```

لا يدعم `Driver` لدينا `Curse` بعد.

## اكتب أصغر قدر من الكود ليعمل الاختبار وافحص مخرجاته الفاشلة

تذكّر أننا نحاول فقط جعل الاختبار يعمل، لذا أضف الـ method إلى `Driver`

```go
func (d *Driver) Curse(name string) (string, error) {
	return "", nil
}
```

إذا جرّبت مرة أخرى، ينبغي أن يُترجم الاختبار ويعمل ويفشل

```
greet.go:26: Expected values to be equal:
+Go to hell, Chris!
\ No newline at end of file
```

## اكتب كودًا كافيًا لنجاح الاختبار

سنحتاج إلى تحديث مواصفة protocol buffer لدينا بإضافة method اسمها `Curse`، ثم إعادة توليد كودنا.

```protobuf
service Greeter {
  rpc Greet (GreetRequest) returns (GreetReply) {}
  rpc Curse (GreetRequest) returns (GreetReply) {}
}
```

قد تقول إن إعادة استخدام النوعين `GreetRequest` و`GreetReply` ارتباط غير مناسب، لكننا نستطيع معالجة ذلك في مرحلة إعادة الهيكلة. وكما أكرر دائمًا، نحن نحاول فقط جعل الاختبار ينجح، لنتحقق من أن البرمجية تعمل، _ثم_ يمكننا تجميلها.

أعد توليد كودنا بالأمر التالي (داخل `adapters/grpcserver`).

```
protoc --go_out=. --go_opt=paths=source_relative \
    --go-grpc_out=. --go-grpc_opt=paths=source_relative \
    greet.proto
```

### تحديث الـ driver

الآن بعد تحديث كود العميل، يمكننا استدعاء `Curse` في `Driver` لدينا

```go
func (d *Driver) Curse(name string) (string, error) {
	client, err := d.getClient()
	if err != nil {
		return "", err
	}

	greeting, err := client.Curse(context.Background(), &GreetRequest{
		Name: name,
	})
	if err != nil {
		return "", err
	}

	return greeting.Message, nil
}
```

### تحديث الخادم

وأخيرًا، نحتاج إلى إضافة method الـ `Curse` إلى `Server` لدينا

```go
package grpcserver

import (
	"context"
	"fmt"

	"github.com/quii/go-specs-greet/domain/interactions"
)

type GreetServer struct {
	UnimplementedGreeterServer
}

func (g GreetServer) Curse(ctx context.Context, request *GreetRequest) (*GreetReply, error) {
	return &GreetReply{Message: fmt.Sprintf("Go to hell, %s!", request.Name)}, nil
}

func (g GreetServer) Greet(ctx context.Context, request *GreetRequest) (*GreetReply, error) {
	return &GreetReply{Message: interactions.Greet(request.Name)}, nil
}
```

ينبغي أن تنجح الاختبارات الآن.

## إعادة الهيكلة

جرّب فعل ذلك بنفسك.

- استخرج "منطق مجال" الـ `Curse` بعيدًا عن خادم grpc، كما فعلنا مع `Greet`. واستخدم المواصفة كاختبار وحدة ضد منطق مجالك
- اجعل الأنواع في protobuf مختلفة لضمان فصل أنواع رسائل `Greet` و`Curse`.

## تنفيذ `Curse` في خادم HTTP

مرة أخرى، هذا تمرين لك أنت، عزيزي القارئ. لدينا مواصفتنا على مستوى المجال ومنطقنا على مستوى المجال مفصولان بعناية. وإذا كنت قد تابعت هذا الفصل، فينبغي أن يكون هذا مباشرًا جدًا.

- أضف المواصفة إلى اختبار القبول الموجود لخادم HTTP
- حدّث `Driver` لديك
- أضف نقطة النهاية (endpoint) الجديدة إلى الخادم، وأعد استخدام كود المجال لتنفيذ الوظيفة. وقد ترغب في استخدام `http.NewServeMux` للتعامل مع التوجيه إلى نقاط النهاية المنفصلة.

تذكّر أن تعمل بخطوات صغيرة، وتسجّل تغييراتك وتشغّل اختباراتك بانتظام. وإن واجهت صعوبة كبيرة، [يمكنك العثور على تنفيذي على GitHub](https://github.com/quii/go-specs-greet).

## حسّن كلا النظامين بتحديث منطق المجال عبر اختبار وحدة

كما ذُكر، ليس كل تغيير في نظام ينبغي أن يُدفع عبر اختبار قبول. فالتوليفات المختلفة من قواعد العمل والحالات الحدية ينبغي أن يكون دفعها عبر اختبار وحدة بسيطًا إذا فصلت الاهتمامات جيدًا.

أضف اختبار وحدة إلى دالة `Greet` لدينا ليجعل `name` افتراضيًا إلى `World` إذا كان فارغًا. وسترى كم هذا بسيط، ثم تنعكس قواعد العمل في كلا التطبيقين "مجانًا".

## الخلاصة

بناء أنظمة بكلفة تغيير معقولة يتطلب أن تكون اختبارات القبول لديك مهندسة لتساعدك، لا لتصبح عبئًا في الصيانة. ويمكن استخدامها وسيلةً لتوجيه برمجياتك، أو "تنميتها" بمنهجية كما يقول كتاب GOOS.

وآمل أن ترى في هذا المثال سير عمل تطبيقنا المتوقع والمنظَّم لدفع التغيير، وكيف يمكنك استخدامه في عملك.

يمكنك تخيل حديثك مع صاحب مصلحة (stakeholder) يريد توسيع النظام الذي تعمل عليه بطريقة ما. التقط ذلك في مواصفة محورها المجال ولا علاقة لها بالتنفيذ، واستخدمها نجمًا شماليًا يهدي جهودك. ونصف أنا وRiya الاستفادة من تقنيات BDD مثل "Example Mapping" [في محاضرتنا في GopherconUK](https://www.youtube.com/watch?v=ZMWJCk_0WrY) لمساعدتك على فهم التعقيد الجوهري بعمق أكبر وتمكينك من كتابة مواصفات أكثر تفصيلًا ومعنى.

فصل التعقيد الجوهري عن التعقيد العرضي سيجعل عملك أقل عشوائية وأكثر تنظيمًا وتأنٍّ؛ وهذا يضمن مرونة اختبارات القبول لديك ويساعدها على أن تصبح عبئًا صيانة أقل.

يقدّم Dave Farley نصيحة ممتازة:

> تخيّل أقل شخص تقني يمكن أن يخطر ببالك، ممن يفهم مجال المشكلة، يقرأ اختبارات القبول لديك. ينبغي أن تكون الاختبارات مفهومة لذلك الشخص.

حينها ينبغي أن تعمل المواصفات توثيقًا مزدوجًا. ينبغي أن تحدد بوضوح كيف ينبغي أن يتصرف النظام. وهذه الفكرة هي المبدأ الذي تقوم عليه أدوات مثل [Cucumber](https://cucumber.io)، التي تقدّم لك DSL لالتقاط السلوكيات في صورة كود، ثم تحوّل ذلك الـ DSL إلى استدعاءات نظام، كما فعلنا هنا تمامًا.

### ما الذي غطيناه

- كتابة مواصفات مجرّدة تتيح لك التعبير عن التعقيد الجوهري للمشكلة التي تحلها وإزالة التعقيد العرضي. وسيمكّنك ذلك من إعادة استخدام المواصفات في سياقات مختلفة.
- كيفية استخدام [Testcontainers](https://golang.testcontainers.org) لإدارة دورة حياة نظامك في اختبارات القبول. وهذا يتيح لك اختبار الصورة التي تنوي إطلاقها على حاسوبك اختبارًا شاملًا، فتحصل على تغذية راجعة سريعة وثقة.
- مقدمة موجزة عن حوسبة تطبيقك باستخدام Docker
- gRPC
- بدلًا من مطاردة بنى مجلدات جاهزة، يمكنك استخدام نهج تطويرك لاستخلاص بنية تطبيقك طبيعيًا، بحسب احتياجاتك أنت

### مواد إضافية

- في هذا المثال، "الـ DSL" لدينا ليس DSL بالمعنى الكامل؛ فقد استخدمنا الـ interfaces فقط لفصل مواصفتنا عن العالم الحقيقي وتمكيننا من التعبير عن منطق المجال بنقاء. ومع نمو نظامك، قد يصبح هذا المستوى من التجريد مرتبكًا وغير واضح. [اقرأ عن "نمط السيناريو" (Screenplay Pattern)](https://cucumber.io/blog/bdd/understanding-screenplay-part-1/) إن أردت العثور على مزيد من الأفكار حول كيفية هيكلة مواصفاتك.
- وللتأكيد، فإن [Growing Object-Oriented Software, Guided by Tests,](http://www.growing-object-oriented-software.com) كتاب كلاسيكي. وهو يوضح تطبيق هذا النهج "بأسلوب لندن" و"من الأعلى إلى الأسفل" في كتابة البرمجيات. وكل من استمتع بكتاب Learn Go with Tests ينبغي أن يجني قيمة كبيرة من قراءة GOOS.
- [في مستودع الكود المثال](https://github.com/quii/go-specs-greet)، يوجد كود وأفكار أكثر لم أكتب عنها هنا، مثل بناء docker متعدد المراحل، وقد ترغب في الاطلاع عليه.
  - وبشكل خاص، *للمتعة*، أنشأت **برنامجًا ثالثًا**، وهو موقع فيه بعض نماذج HTML لـ `Greet` و`Curse`. ويستفيد `Driver` من الوحدة [https://github.com/go-rod/rod](https://github.com/go-rod/rod) البديعة المظهر، والتي تتيح له العمل مع الموقع عبر متصفح، تمامًا كما يفعل المستخدم. وبنظرة إلى تاريخ git، يمكنك أن ترى كيف بدأت دون استخدام أي أدوات قوالب "فقط لنجعله يعمل". ثم، بعد أن نجح اختبار القبول لدي، صارت لدي الحرية في استخدامها دون خوف من كسر الأشياء. -->
