---
title: الـ Maps
weight: 80
---

# الـ Maps

**[يمكنك العثور على كل كود هذا الفصل هنا](https://github.com/quii/learn-go-with-tests/tree/main/maps)**

في [المصفوفات والشرائح](arrays-and-slices.md)، رأيت كيف تخزّن القيم بالترتيب. والآن سننظر إلى طريقة لتخزين العناصر حسب `key` (مفتاح) والبحث عنها بسرعة.

تتيح لك الـ Maps تخزين العناصر بطريقة تشبه القاموس؛ فيمكنك التفكير في الـ `key` كأنها الكلمة، وفي الـ `value` كأنه التعريف. وهل توجد طريقة أفضل لتعلّم الـ Maps من بناء قاموسنا الخاص؟

أولًا، بافتراض أن لدينا بالفعل بعض الكلمات وتعريفاتها في القاموس، فإذا بحثنا عن كلمة ينبغي أن يُرجع لنا تعريفها.

## اكتب الاختبار أولًا

في `dictionary_test.go`

```go
package main

import "testing"

func TestSearch(t *testing.T) {
	dictionary := map[string]string{"test": "this is just a test"}

	got := Search(dictionary, "test")
	want := "this is just a test"

	if got != want {
		t.Errorf("got %q want %q given, %q", got, want, "test")
	}
}
```

تعريف الـ Map يشبه إلى حد ما تعريف المصفوفة، إلا أنه يبدأ بالكلمة المفتاحية `map` ويحتاج إلى نوعين: الأول نوع المفتاح ويُكتب داخل `[]`، والثاني نوع القيمة ويأتي مباشرة بعد `[]`.

ونوع المفتاح مميز؛ إذ لا يمكن أن يكون إلا نوعًا قابلًا للمقارنة (comparable)، لأنه دون القدرة على معرفة ما إذا كان مفتاحان متساويين لا سبيل لنا إلى التأكد من حصولنا على القيمة الصحيحة. والأنواع القابلة للمقارنة مشروحة بالتفصيل في [مواصفات اللغة](https://golang.org/ref/spec#Comparison_operators).

أما نوع القيمة فيمكن أن يكون أي نوع تريده، بل يمكن أن يكون Map آخر.

وكل ما تبقّى في هذا الاختبار ينبغي أن يكون مألوفًا.

## جرّب تشغيل الاختبار

عند تشغيل `go test` سيفشل المترجم بالخطأ `./dictionary_test.go:8:9: undefined: Search`.

## اكتب أصغر قدر من الكود ليعمل الاختبار وافحص المخرجات

في `dictionary.go`

```go
package main

func Search(dictionary map[string]string, word string) string {
	return ""
}
```

من المفترض أن يفشل اختبارك الآن بـ *رسالة خطأ واضحة*

`dictionary_test.go:12: got '' want 'this is just a test' given, 'test'`.

## اكتب كودًا كافيًا لنجاح الاختبار

```go
func Search(dictionary map[string]string, word string) string {
	return dictionary[word]
}
```

الحصول على قيمة من الـ Map يشبه الحصول على قيمة من مصفوفة عبر `map[key]`.

## إعادة الهيكلة

```go
func TestSearch(t *testing.T) {
	dictionary := map[string]string{"test": "this is just a test"}

	got := Search(dictionary, "test")
	want := "this is just a test"

	assertStrings(t, got, want)
}

func assertStrings(t testing.TB, got, want string) {
	t.Helper()

	if got != want {
		t.Errorf("got %q want %q", got, want)
	}
}
```

قررت إنشاء دالة مساعدة `assertStrings` لجعل التنفيذ أعم.

### استخدام نوع مخصص

يمكننا تحسين استخدام قاموسنا بإنشاء نوع جديد يغلّف map وجعل `Search` method.

في `dictionary_test.go`:

```go
func TestSearch(t *testing.T) {
	dictionary := Dictionary{"test": "this is just a test"}

	got := dictionary.Search("test")
	want := "this is just a test"

	assertStrings(t, got, want)
}
```

بدأنا نستخدم نوع `Dictionary`، وهو نوع لم نعرّفه بعد، ثم استدعينا `Search` على نسخة `Dictionary`.

ولم نحتج إلى تغيير `assertStrings`.

في `dictionary.go`:

```go
type Dictionary map[string]string

func (d Dictionary) Search(word string) string {
	return d[word]
}
```

هنا أنشأنا نوع `Dictionary` يعمل كغلاف رفيع حول `map`. وبعد تعريف النوع المخصص، يمكننا إنشاء الـ method `Search`.

## اكتب الاختبار أولًا

كان البحث الأساسي سهل التنفيذ جدًا، لكن ماذا سيحدث إذا مرّرنا كلمة غير موجودة في قاموسنا؟

في الحقيقة لا نحصل على أي شيء، وهذا جيد لأن البرنامج يستطيع مواصلة العمل، لكن هناك أسلوب أفضل: أن تخبرنا الدالة بأن الكلمة ليست في القاموس. وهكذا لا يبقى المستخدم متسائلًا هل الكلمة غير موجودة أصلًا أم أنه لا يوجد لها تعريف فقط (قد لا يبدو هذا مفيدًا كثيرًا في قاموس، لكنه سيناريو قد يكون محوريًا في حالات استخدام أخرى).

```go
func TestSearch(t *testing.T) {
	dictionary := Dictionary{"test": "this is just a test"}

	t.Run("known word", func(t *testing.T) {
		got, _ := dictionary.Search("test")
		want := "this is just a test"

		assertStrings(t, got, want)
	})

	t.Run("unknown word", func(t *testing.T) {
		_, err := dictionary.Search("unknown")
		want := "could not find the word you were looking for"

		if err == nil {
			t.Fatal("expected to get an error.")
		}

		assertStrings(t, err.Error(), want)
	})
}
```

طريقة التعامل مع هذا السيناريو في Go هي إرجاع وسيط ثانٍ من نوع `Error`.

ولاحظ، كما رأينا في [قسم المؤشرات والأخطاء](pointers-and-errors.md)، أننا نفحص هنا أولًا أن الخطأ ليس `nil`، ثم نستخدم الـ method `.Error()` للحصول على النص الذي يمكننا تمريره إلى التحقق.

## جرّب تشغيل الاختبار

هذا الكود لا يُترجم

```
./dictionary_test.go:18:10: assignment mismatch: 2 variables but 1 values
```

## اكتب أصغر قدر من الكود ليعمل الاختبار وافحص المخرجات

```go
func (d Dictionary) Search(word string) (string, error) {
	return d[word], nil
}
```

من المفترض أن يفشل اختبارك الآن برسالة خطأ أوضح بكثير.

`dictionary_test.go:22: expected to get an error.`

## اكتب كودًا كافيًا لنجاح الاختبار

```go
func (d Dictionary) Search(word string) (string, error) {
	definition, ok := d[word]
	if !ok {
		return "", errors.New("could not find the word you were looking for")
	}

	return definition, nil
}
```

لنجعل هذا ينجح، نستفيد هنا من خاصية مثيرة في البحث داخل الـ Map: فهو يمكن أن يُرجع قيمتين، والقيمة الثانية منطقية (boolean) تشير إلى ما إذا كان المفتاح قد وُجد بنجاح.

وتتيح لنا هذه الخاصية التمييز بين كلمة غير موجودة أصلًا وكلمة لا يوجد لها تعريف فقط.

## إعادة الهيكلة

```go
var ErrNotFound = errors.New("could not find the word you were looking for")

func (d Dictionary) Search(word string) (string, error) {
	definition, ok := d[word]
	if !ok {
		return "", ErrNotFound
	}

	return definition, nil
}
```

يمكننا التخلص من الخطأ "السحري" في دالة `Search` باستخراجه إلى متغير، وسيتيح لنا ذلك أيضًا اختبارًا أفضل.

```go
t.Run("unknown word", func(t *testing.T) {
	_, got := dictionary.Search("unknown")
	if got == nil {
		t.Fatal("expected to get an error.")
	}
	assertError(t, got, ErrNotFound)
})
```
```go
func assertError(t testing.TB, got, want error) {
	t.Helper()

	if !errors.Is(got, want) {
		t.Errorf("got error %q want %q", got, want)
	}
}
```

فبإنشاء دالة مساعدة جديدة تمكّنا من تبسيط اختبارنا، وبدأنا نستخدم متغير `ErrNotFound` حتى لا يفشل اختبارنا إذا غيّرنا نص الخطأ في المستقبل.

## اكتب الاختبار أولًا

لدينا طريقة رائعة للبحث في القاموس، لكن لا توجد لدينا طريقة لإضافة كلمات جديدة إليه.

```go
func TestAdd(t *testing.T) {
	dictionary := Dictionary{}
	dictionary.Add("test", "this is just a test")

	want := "this is just a test"
	got, err := dictionary.Search("test")
	if err != nil {
		t.Fatal("should find added word:", err)
	}

	assertStrings(t, got, want)
}
```

نستفيد في هذا الاختبار من دالة `Search` لجعل التحقق من القاموس أسهل قليلًا.

## اكتب أصغر قدر من الكود ليعمل الاختبار وافحص المخرجات

في `dictionary.go`

```go
func (d Dictionary) Add(word, definition string) {
}
```

من المفترض أن يفشل اختبارك الآن

```
dictionary_test.go:31: should find added word: could not find the word you were looking for
```

## اكتب كودًا كافيًا لنجاح الاختبار

```go
func (d Dictionary) Add(word, definition string) {
	d[word] = definition
}
```

إضافة عنصر إلى Map تشبه أيضًا الإضافة إلى مصفوفة؛ كل ما عليك فعله هو تحديد مفتاح وجعله مساويًا لقيمة.

### المؤشرات والنسخ وما شابه

من الخصائص المثيرة للـ Maps أنك تستطيع تعديلها دون تمرير عنوانها إليها (مثل `&myMap`)

وقد يجعلها ذلك _تبدو_ كأنها "نوع مرجعي"، [لكن كما يوضح Dave Cheney](https://dave.cheney.net/2017/04/30/if-a-map-isnt-a-reference-variable-what-is-it) فهي ليست كذلك.

> قيمة الـ map هي مؤشر (pointer) إلى بنية runtime.hmap.

لذا فعندما تمرر map إلى دالة أو method فأنت تنسخها فعلًا، لكن الجزء الخاص بالمؤشر فقط، لا بنية البيانات الأساسية التي تحوي البيانات.

ومن المفاجآت (gotchas) مع الـ Maps أنها قد تكون بقيمة `nil`. والـ map من نوع `nil` تتصرف كخريطة فارغة عند القراءة، أما محاولة الكتابة فيها فتسبب panic وقت التشغيل. ويمكنك قراءة المزيد عن الـ Maps [هنا](https://blog.golang.org/go-maps-in-action).

لذلك ينبغي ألا تهيئ أبدًا متغير map من نوع nil:

```go
var m map[string]string
```

بدلًا من ذلك يمكنك تهيئة map فارغة أو استخدام الكلمة المفتاحية `make` لإنشاء map لك:

```go
var dictionary = map[string]string{}

// OR

var dictionary = make(map[string]string)
```

وكلا الأسلوبين ينشئ `hash map` فارغة ويجعل `dictionary` يشير إليها، ما يضمن ألا تصادف panic وقت التشغيل أبدًا.

## إعادة الهيكلة

لا يوجد الكثير لنعيد هيكلته في التنفيذ، لكن الاختبار يمكن تبسيطه قليلًا.

```go
func TestAdd(t *testing.T) {
	dictionary := Dictionary{}
	word := "test"
	definition := "this is just a test"

	dictionary.Add(word, definition)

	assertDefinition(t, dictionary, word, definition)
}

func assertDefinition(t testing.TB, dictionary Dictionary, word, definition string) {
	t.Helper()

	got, err := dictionary.Search(word)
	if err != nil {
		t.Fatal("should find added word:", err)
	}
	assertStrings(t, got, definition)
}
```

أنشأنا متغيرين للكلمة والتعريف، ونقلنا التحقق من التعريف إلى دالة مساعدة خاصة به.

تبدو دالة `Add` لدينا جيدة، إلا أننا لم نفكر في ما يحدث عندما تكون القيمة التي نحاول إضافتها موجودة بالفعل!

لن تُصدر الـ Maps خطأً إذا كانت القيمة موجودة بالفعل، بل ستستمر وتستبدل القيمة بالقيمة الجديدة الممرَّرة. وقد يكون هذا مناسبًا عمليًا، لكنه يجعل اسم دالتنا غير دقيق؛ فـ `Add` ينبغي ألا تعدّل القيم الموجودة، بل تضيف كلمات جديدة فقط إلى قاموسنا.

## اكتب الاختبار أولًا

```go
func TestAdd(t *testing.T) {
	t.Run("new word", func(t *testing.T) {
		dictionary := Dictionary{}
		word := "test"
		definition := "this is just a test"

		err := dictionary.Add(word, definition)

		assertError(t, err, nil)
		assertDefinition(t, dictionary, word, definition)
	})

	t.Run("existing word", func(t *testing.T) {
		word := "test"
		definition := "this is just a test"
		dictionary := Dictionary{word: definition}
		err := dictionary.Add(word, "new test")

		assertError(t, err, ErrWordExists)
		assertDefinition(t, dictionary, word, definition)
	})
}
```

في هذا الاختبار عدّلنا `Add` لتُرجع خطأً نتحقق منه مقابل متغير خطأ جديد هو `ErrWordExists`، كما عدّلنا الاختبار السابق ليفحص أن الخطأ `nil`.

## جرّب تشغيل الاختبار

سيفشل المترجم لأننا لا نُرجع قيمة من `Add`.

```
./dictionary_test.go:30:13: dictionary.Add(word, definition) used as value
./dictionary_test.go:41:13: dictionary.Add(word, "new test") used as value
```

## اكتب أصغر قدر من الكود ليعمل الاختبار وافحص المخرجات

في `dictionary.go`

```go
var (
	ErrNotFound   = errors.New("could not find the word you were looking for")
	ErrWordExists = errors.New("cannot add word because it already exists")
)

func (d Dictionary) Add(word, definition string) error {
	d[word] = definition
	return nil
}
```

الآن نحصل على خطأين إضافيين؛ فما زلنا نعدّل القيمة، ونُرجع خطأ `nil`.

```
dictionary_test.go:43: got error '%!q(<nil>)' want 'cannot add word because it already exists'
dictionary_test.go:44: got 'new test' want 'this is just a test'
```

## اكتب كودًا كافيًا لنجاح الاختبار

```go
func (d Dictionary) Add(word, definition string) error {
	_, err := d.Search(word)

	switch err {
	case ErrNotFound:
		d[word] = definition
	case nil:
		return ErrWordExists
	default:
		return err
	}

	return nil
}
```

نستخدم هنا جملة `switch` للمطابقة على الخطأ. ووجود `switch` كهذه يوفّر شبكة أمان إضافية، في حال أرجعت `Search` خطأً غير `ErrNotFound`.

## إعادة الهيكلة

ليس لدينا الكثير لنعيد هيكلته، لكن مع تزايد استخدامنا للأخطاء يمكننا إجراء بعض التعديلات.

```go
const (
	ErrNotFound   = DictionaryErr("could not find the word you were looking for")
	ErrWordExists = DictionaryErr("cannot add word because it already exists")
)

type DictionaryErr string

func (e DictionaryErr) Error() string {
	return string(e)
}
```

جعلنا الأخطاء ثوابت؛ وقد تطلب ذلك إنشاء نوع `DictionaryErr` خاص بنا يحقق واجهة `error`. ويمكنك قراءة المزيد من التفاصيل في [هذا المقال الممتاز لـ Dave Cheney](https://dave.cheney.net/2016/04/07/constant-errors). وببساطة، فإن ذلك يجعل الأخطاء أكثر قابلية لإعادة الاستخدام وثباتًا.

بعد ذلك، لننشئ دالة لتحديث (`Update`) تعريف كلمة.

## اكتب الاختبار أولًا

```go
func TestUpdate(t *testing.T) {
	word := "test"
	definition := "this is just a test"
	dictionary := Dictionary{word: definition}
	newDefinition := "new definition"

	dictionary.Update(word, newDefinition)

	assertDefinition(t, dictionary, word, newDefinition)
}
```

ترتبط `Update` ارتباطًا وثيقًا بـ `Add` وستكون تنفيذنا التالي.

## جرّب تشغيل الاختبار

```
./dictionary_test.go:53:2: dictionary.Update undefined (type Dictionary has no field or method Update)
```

## اكتب أصغر قدر من الكود ليعمل الاختبار وافحص مخرجاته الفاشلة

نحن نعرف بالفعل كيف نتعامل مع خطأ كهذا؛ علينا تعريف دالتنا.

```go
func (d Dictionary) Update(word, definition string) {}
```

وبعد ذلك، نستطيع أن نرى أننا نحتاج إلى تغيير تعريف الكلمة.

```
dictionary_test.go:55: got 'this is just a test' want 'new definition'
```

## اكتب كودًا كافيًا لنجاح الاختبار

رأينا بالفعل كيف نفعل ذلك عندما أصلحنا مشكلة `Add`، فلننفّذ إذن شيئًا شبيهًا جدًا بـ `Add`.

```go
func (d Dictionary) Update(word, definition string) {
	d[word] = definition
}
```

لا توجد إعادة هيكلة نحتاج إليها هنا لأن التغيير كان بسيطًا. لكن أصبح لدينا الآن المشكلة نفسها التي واجهتنا مع `Add`؛ فإذا مرّرنا كلمة جديدة، ستضيفها `Update` إلى القاموس.

## اكتب الاختبار أولًا

```go
t.Run("existing word", func(t *testing.T) {
	word := "test"
	definition := "this is just a test"
	dictionary := Dictionary{word: definition}
	newDefinition := "new definition"

	err := dictionary.Update(word, newDefinition)

	assertError(t, err, nil)
	assertDefinition(t, dictionary, word, newDefinition)
})

t.Run("new word", func(t *testing.T) {
	word := "test"
	definition := "this is just a test"
	dictionary := Dictionary{}

	err := dictionary.Update(word, definition)

	assertError(t, err, ErrWordDoesNotExist)
})
```

أضفنا نوع خطأ آخر للحالة التي تكون فيها الكلمة غير موجودة، كما عدّلنا `Update` لتُرجع قيمة `error`.

## جرّب تشغيل الاختبار

```
./dictionary_test.go:53:16: dictionary.Update(word, newDefinition) used as value
./dictionary_test.go:64:16: dictionary.Update(word, definition) used as value
./dictionary_test.go:66:23: undefined: ErrWordDoesNotExist
```

نحصل هذه المرة على 3 أخطاء، لكننا نعرف كيف نتعامل معها.

## اكتب أصغر قدر من الكود ليعمل الاختبار وافحص مخرجاته الفاشلة

```go
const (
	ErrNotFound         = DictionaryErr("could not find the word you were looking for")
	ErrWordExists       = DictionaryErr("cannot add word because it already exists")
	ErrWordDoesNotExist = DictionaryErr("cannot perform operation on word because it does not exist")
)

func (d Dictionary) Update(word, definition string) error {
	d[word] = definition
	return nil
}
```

أضفنا نوع الخطأ الخاص بنا ونُرجع خطأ `nil`.

وبهذه التغييرات نحصل الآن على خطأ واضح جدًا:

```
dictionary_test.go:66: got error '%!q(<nil>)' want 'cannot perform operation on word because it does not exist'
```

## اكتب كودًا كافيًا لنجاح الاختبار

```go
func (d Dictionary) Update(word, definition string) error {
	_, err := d.Search(word)

	switch err {
	case ErrNotFound:
		return ErrWordDoesNotExist
	case nil:
		d[word] = definition
	default:
		return err
	}

	return nil
}
```

تبدو هذه الدالة شبه مطابقة لـ `Add`، إلا أننا عكسنا متى نحدّث `dictionary` ومتى نُرجع خطأ.

### ملاحظة عن تعريف خطأ جديد لـ Update

كان بإمكاننا إعادة استخدام `ErrNotFound` وعدم إضافة خطأ جديد، لكن من الأفضل غالبًا أن يكون لدينا خطأ دقيق عندما يفشل التحديث.

فالأخطاء المحددة تمنحك معلومات أكثر عمّا حدث خطأ. وإليك مثالًا في تطبيق ويب:

> يمكنك إعادة توجيه المستخدم عند مواجهة `ErrNotFound`، لكن اعرض رسالة خطأ عند مواجهة `ErrWordDoesNotExist`.

بعد ذلك، لننشئ دالة لحذف (`Delete`) كلمة من القاموس.

## اكتب الاختبار أولًا

```go
func TestDelete(t *testing.T) {
	word := "test"
	dictionary := Dictionary{word: "test definition"}

	dictionary.Delete(word)

	_, err := dictionary.Search(word)
	assertError(t, err, ErrNotFound)
}
```

ينشئ اختبارنا `Dictionary` بكلمة ثم يتحقق مما إذا كانت الكلمة قد حُذفت.

## جرّب تشغيل الاختبار

بتشغيل `go test` نحصل على:

```
./dictionary_test.go:74:6: dictionary.Delete undefined (type Dictionary has no field or method Delete)
```

## اكتب أصغر قدر من الكود ليعمل الاختبار وافحص مخرجاته الفاشلة

```go
func (d Dictionary) Delete(word string) {

}
```

بعد أن نضيف هذا، يخبرنا الاختبار أننا لا نحذف الكلمة.

```
dictionary_test.go:78: got error '%!q(<nil>)' want 'could not find the word you were looking for'
```

## اكتب كودًا كافيًا لنجاح الاختبار

```go
func (d Dictionary) Delete(word string) {
	delete(d, word)
}
```

تمتلك Go دالة مدمجة اسمها `delete` تعمل على الـ Maps، وهي تأخذ وسيطين ولا تُرجع شيئًا: الوسيط الأول هو الـ map والثاني هو المفتاح المراد حذفه.

## إعادة الهيكلة

لا يوجد الكثير لنعيد هيكلته، لكن يمكننا تطبيق المنطق نفسه من `Update` للتعامل مع الحالات التي تكون فيها الكلمة غير موجودة.

```go
func TestDelete(t *testing.T) {
	t.Run("existing word", func(t *testing.T) {
		word := "test"
		dictionary := Dictionary{word: "test definition"}

		err := dictionary.Delete(word)

		assertError(t, err, nil)

		_, err = dictionary.Search(word)

		assertError(t, err, ErrNotFound)
	})

	t.Run("non-existing word", func(t *testing.T) {
		word := "test"
		dictionary := Dictionary{}

		err := dictionary.Delete(word)

		assertError(t, err, ErrWordDoesNotExist)
	})
}
```

## جرّب تشغيل الاختبار

سيفشل المترجم لأننا لا نُرجع قيمة من `Delete`.

```
./dictionary_test.go:77:10: dictionary.Delete(word) (no value) used as value
./dictionary_test.go:90:10: dictionary.Delete(word) (no value) used as value
```

## اكتب كودًا كافيًا لنجاح الاختبار

```go
func (d Dictionary) Delete(word string) error {
	_, err := d.Search(word)

	switch err {
	case ErrNotFound:
		return ErrWordDoesNotExist
	case nil:
		delete(d, word)
	default:
		return err
	}

	return nil
}
```

نستخدم مرة أخرى جملة switch للمطابقة على الخطأ عندما نحاول حذف كلمة غير موجودة.

## الخلاصة

تناولنا في هذا القسم الكثير، فصنعنا واجهة CRUD كاملة (إنشاء، قراءة، تحديث، وحذف) لقاموسنا. وخلال الطريق تعلمنا كيف:

* ننشئ الـ Maps
* نبحث عن العناصر في الـ Maps
* نضيف عناصر جديدة إلى الـ Maps
* نحدّث العناصر في الـ Maps
* نحذف العناصر من الـ Map
* تعلمنا المزيد عن الأخطاء
  * كيف ننشئ أخطاء تكون ثوابت
  * كتابة مغلّفات الأخطاء (error wrappers)
