---
title: إعادة زيارة المصفوفات والشرائح مع الـ Generics
weight: 210
---

# إعادة زيارة المصفوفات والشرائح مع الـ Generics

**[كود هذا الفصل تكملة لفصل المصفوفات والشرائح، وتجده هنا](https://github.com/quii/learn-go-with-tests/tree/main/arrays)**

ألقِ نظرة على كل من `SumAll` و`SumAllTails` اللتين كتبناهما في [المصفوفات والشرائح](arrays-and-slices.md). وإذا لم تكن تحتفظ بنسختك، فانسخ الكود من فصل [المصفوفات والشرائح](arrays-and-slices.md) مع الاختبارات.

```go
// Sum calculates the total from a slice of numbers.
func Sum(numbers []int) int {
	var sum int
	for _, number := range numbers {
		sum += number
	}
	return sum
}

// SumAllTails calculates the sums of all but the first number given a collection of slices.
func SumAllTails(numbersToSum ...[]int) []int {
	var sums []int
	for _, numbers := range numbersToSum {
		if len(numbers) == 0 {
			sums = append(sums, 0)
		} else {
			tail := numbers[1:]
			sums = append(sums, Sum(tail))
		}
	}

	return sums
}
```

هل ترى نمطًا متكررًا؟

- أنشئ نوعًا من القيمة "الابتدائية" للنتيجة.
- مرّ على المجموعة، وطبّق نوعًا من العمليات (أو دالة) على النتيجة والعنصر التالي في الشريحة، مع إسناد قيمة جديدة للنتيجة
- أرجِع النتيجة.

هذه الفكرة شائعة في أوساط البرمجة الوظيفية (functional programming)، وتُسمى غالبًا 'reduce' أو [fold](https://en.wikipedia.org/wiki/Fold_(higher-order_function)).

> في البرمجة الوظيفية، يُشير الـ fold (ويُسمى أيضًا reduce أو accumulate أو aggregate أو compress أو inject) إلى عائلة من الدوال ذات الرتبة الأعلى التي تُحلّل بنية بيانات عودية، وتعيد عبر عملية دمج معطاة تجميع نتائج معالجة مكوناتها معالجةً عودية، لتبني قيمة إرجاع. وعادةً يُقدَّم إلى الـ fold دالة دمج، وعقدة عليا من بنية البيانات، وربما بعض القيم الافتراضية التي تُستخدم في ظروف معينة. ثم يمضي الـ fold في دمج عناصر التسلسل الهرمي لبنية البيانات، مستخدمًا الدالة بطريقة منهجية.

لطالما امتلكت Go دوالًا ذات رتبة أعلى، وبدءًا من الإصدار 1.18 صار لديها أيضًا [generics](generics.md)، فأصبح من الممكن الآن تعريف بعض هذه الدوال التي تُناقش في مجالنا الأوسع. ولا فائدة من دفن رأسك في الرمال؛ فهذا تجريد شائع جدًا خارج منظومة Go، وسيكون من المفيد أن تفهمه.

الآن، أعلم أن بعضكم ربما ينقبض قلبه من هذا.

> يُفترض أن تكون Go بسيطة

**لا تخلط بين السهولة والبساطة**. فكتابة الحلقات ونسخ الكود ولصقه أمر سهل، لكنه ليس بسيطًا بالضرورة. ولمعرفة المزيد عن الفرق بين البسيط والسهل، شاهد [محاضرة Rich Hickey الرائعة - Simple Made Easy](https://www.youtube.com/watch?v=SxdOUGdseq4).

**لا تخلط بين عدم الألفة والتعقيد**. قد يبدو الـ Fold/reduce مخيفًا في البداية ويميل إلى لغة علوم الحاسوب، لكنه في حقيقته ليس سوى تجريد لعملية شائعة جدًا: أخذ مجموعة ودمجها في عنصر واحد. وإذا تراجعت خطوة للوراء، ستدرك أنك على الأرجح تفعل هذا _كثيرًا_.

## إعادة هيكلة باستخدام الـ generics

من الأخطاء التي يقع فيها الناس غالبًا مع ميزات اللغة الجديدة اللامعة أنهم يبدأون باستخدامها دون وجود حالة استخدام ملموسة، فيعتمدون على التخمين والحدس في توجيه جهودهم.

لحسن الحظ، كتبنا دوالنا "المفيدة" ولدينا اختبارات حولها، فأصبحنا أحرارًا في تجربة الأفكار في مرحلة إعادة الهيكلة من TDD، ونحن نعرف أن أي شيء نجرّبه تُتحقق قيمته عبر اختبارات الوحدات.

واستخدام الـ generics كأداة لتبسيط الكود عبر خطوة إعادة الهيكلة أرجح بكثير أن يرشدك إلى تحسينات مفيدة بدلًا من تجريدات سابقة لأوانها.

نحن آمنون في تجربة الأشياء وإعادة تشغيل اختباراتنا؛ فإن أعجبنا التغيير سجّلناه في commit، وإن لم يعجبنا أعدنا التغيير فقط. وحرية التجربة هذه من أعظم قيم TDD الحقيقية.

ينبغي أن تكون على دراية بصياغة الـ generics [من الفصل السابق](generics.md)، فجرّب كتابة دالة `Reduce` الخاصة بك واستخدمها داخل `Sum` و`SumAllTails`.

### تلميحات

إذا فكّرت في وسائط دالتك أولًا، فسيضيق أمامك نطاق الحلول الصحيحة كثيرًا
  - المصفوفة التي تريد الـ reduce عليها
  - نوع من دالة الدمج

‏"Reduce" نمط موثّق توثيقًا هائلًا، فلا حاجة لإعادة اختراع العجلة. [اقرأ صفحة الويكي، وبخاصة قسم القوائم](https://en.wikipedia.org/wiki/Fold_(higher-order_function))، ومن المفترض أن يذكّرك بوسيط آخر ستحتاجه.

> من الناحية العملية، من المريح والطبيعي أن تكون لديك قيمة ابتدائية

### محاولتي الأولى مع `Reduce`

```go
func Reduce[A any](collection []A, f func(A, A) A, initialValue A) A {
	var result = initialValue
	for _, x := range collection {
		result = f(result, x)
	}
	return result
}
```

تلتقط `Reduce` _جوهر_ النمط؛ فهي دالة تأخذ مجموعة، ودالة تجميع، وقيمة ابتدائية، ثم تُرجع قيمة واحدة. ولا توجد فوضى ولا مشتتات حول أنواع ملموسة.

إذا كنت تفهم صياغة الـ generics، فلا ينبغي أن تواجه مشكلة في فهم ما تفعله هذه الدالة. وباستخدام المصطلح المعروف `Reduce`، يفهم المبرمجون من لغات أخرى القصد أيضًا.

### الاستخدام

```go
// Sum calculates the total from a slice of numbers.
func Sum(numbers []int) int {
	add := func(acc, x int) int { return acc + x }
	return Reduce(numbers, add, 0)
}

// SumAllTails calculates the sums of all but the first number given a collection of slices.
func SumAllTails(numbers ...[]int) []int {
	sumTail := func(acc, x []int) []int {
		if len(x) == 0 {
			return append(acc, 0)
		} else {
			tail := x[1:]
			return append(acc, Sum(tail))
		}
	}

	return Reduce(numbers, sumTail, []int{})
}
```

تصف `Sum` و`SumAllTails` الآن سلوك حساباتهما عبر الدالة المعرَّفة في أول سطر من كل منهما على التوالي. أما عملية تشغيل الحساب على المجموعة فقد جُرّدت في `Reduce`.

## تطبيقات أخرى للـ reduce

باستخدام الاختبارات يمكننا التجربة بدالة الـ reduce لنرى مدى قابليتها لإعادة الاستخدام. وقد نسخت دوال التحقق الـ generic من الفصل السابق.

```go
func TestReduce(t *testing.T) {
	t.Run("multiplication of all elements", func(t *testing.T) {
		multiply := func(x, y int) int {
			return x * y
		}

		AssertEqual(t, Reduce([]int{1, 2, 3}, multiply, 1), 6)
	})

	t.Run("concatenate strings", func(t *testing.T) {
		concatenate := func(x, y string) string {
			return x + y
		}

		AssertEqual(t, Reduce([]string{"a", "b", "c"}, concatenate, ""), "abc")
	})
}
```

### القيمة الصفرية

في مثال الضرب، نوضّح سبب وجود قيمة افتراضية كوسيط لـ `Reduce`. فلو اعتمدنا على القيمة الافتراضية 0 التي تستخدمها Go مع `int`، لضربنا قيمتنا الابتدائية في 0 ثم في القيم التالية، فلن تحصل أبدًا إلا على 0. أما بضبطها على 1، فسيبقى العنصر الأول في الشريحة كما هو، وتُضرب البقية في العناصر التالية.

وإذا أردت أن تبدو ذكيًا أمام أصدقائك المهووسين، فستسمّي هذا [العنصر المحايد (The Identity Element)](https://en.wikipedia.org/wiki/Identity_element).

> في الرياضيات، العنصر المحايد (identity element)، أو العنصر المعتدل (neutral element)، لعملية ثنائية على مجموعة هو عنصر من المجموعة يترك كل عنصر فيها دون تغيير عند تطبيق العملية.

وفي الجمع، العنصر المحايد هو 0.

`1 + 0 = 1`

وفي الضرب، هو 1.

`1 * 1 = 1`

## ماذا لو أردنا الـ reduce إلى نوع مختلف عن `A`؟

لنفترض أن لدينا قائمة معاملات من النوع `Transaction`، وأردنا دالة تأخذها مع اسم لنحدد به رصيده البنكي.

لنتبع عملية TDD.

## اكتب الاختبار أولًا

```go
func TestBadBank(t *testing.T) {
	transactions := []Transaction{
		{
			From: "Chris",
			To:   "Riya",
			Sum:  100,
		},
		{
			From: "Adil",
			To:   "Chris",
			Sum:  25,
		},
	}

	AssertEqual(t, BalanceFor(transactions, "Riya"), 100)
	AssertEqual(t, BalanceFor(transactions, "Chris"), -75)
	AssertEqual(t, BalanceFor(transactions, "Adil"), -25)
}
```

## جرّب تشغيل الاختبار
```
# github.com/quii/learn-go-with-tests/arrays/v8 [github.com/quii/learn-go-with-tests/arrays/v8.test]
./bad_bank_test.go:6:20: undefined: Transaction
./bad_bank_test.go:18:14: undefined: BalanceFor
```

## اكتب أصغر قدر من الكود ليعمل الاختبار وافحص مخرجاته الفاشلة

ليس لدينا أنواعنا أو دوالنا بعد، فأضفها ليعمل الاختبار.

```go
type Transaction struct {
	From string
	To   string
	Sum  float64
}

func BalanceFor(transactions []Transaction, name string) float64 {
	return 0.0
}
```

عند تشغيل الاختبار ينبغي أن ترى ما يلي:

```
=== RUN   TestBadBank
    bad_bank_test.go:19: got 0, want 100
    bad_bank_test.go:20: got 0, want -75
    bad_bank_test.go:21: got 0, want -25
--- FAIL: TestBadBank (0.00s)
```

## اكتب كودًا كافيًا لنجاح الاختبار

لنكتب الكود أولًا كأننا لا نملك دالة `Reduce`.

```go
func BalanceFor(transactions []Transaction, name string) float64 {
	var balance float64
	for _, t := range transactions {
		if t.From == name {
			balance -= t.Sum
		}
		if t.To == name {
			balance += t.Sum
		}
	}
	return balance
}
```

## إعادة الهيكلة

في هذه المرحلة، تحلَّ ببعض الانضباط في التحكم في الإصدارات وسجّل عملك في commit. فلدينا برنامج يعمل، جاهز لتحدي Monzo وBarclays وأمثالهما.

وبعد أن سجّلنا عملنا، صرنا أحرارًا في العبث به وتجربة بعض الأفكار المختلفة في مرحلة إعادة الهيكلة. وإنصافًا، فالكود الذي لدينا ليس سيئًا تمامًا، لكن لأجل هذا التمرين أريد أن أعرض الكود نفسه باستخدام `Reduce`.

```go
func BalanceFor(transactions []Transaction, name string) float64 {
	adjustBalance := func(currentBalance float64, t Transaction) float64 {
		if t.From == name {
			return currentBalance - t.Sum
		}
		if t.To == name {
			return currentBalance + t.Sum
		}
		return currentBalance
	}
	return Reduce(transactions, adjustBalance, 0.0)
}
```

لكن هذا لن يُترجم.

```
./bad_bank.go:19:35: type func(acc float64, t Transaction) float64 of adjustBalance does not match inferred type func(Transaction, Transaction) Transaction for func(A, A) A
```

السبب أننا نحاول الـ reduce إلى نوع _مختلف_ عن نوع المجموعة. يبدو هذا مخيفًا، لكنه في الحقيقة يتطلب فقط تعديل توقيع النوع الخاص بـ `Reduce` ليعمل الأمر. ولن نحتاج إلى تغيير جسم الدالة، ولن نحتاج إلى تغيير أي من مستدعيها الحاليين.

```go
func Reduce[A, B any](collection []A, f func(B, A) B, initialValue B) B {
	var result = initialValue
	for _, x := range collection {
		result = f(result, x)
	}
	return result
}
```

أضفنا قيد نوع ثانٍ أتاح لنا إرخاء القيود على `Reduce`. وهذا يتيح للناس تنفيذ `Reduce` من مجموعة من النوع `A` إلى النوع `B`. وفي حالتنا من `Transaction` إلى `float64`.

يجعل هذا `Reduce` أكثر عمومية وقابلية لإعادة الاستخدام، مع بقائها آمنة الأنواع. وإذا حاولت تشغيل الاختبارات مرة أخرى فينبغي أن تُترجم وتنجح.

## توسيع البنك

على سبيل التسلية، أردت تحسين راحة استخدام كود البنك. وقد حذفت عملية TDD اختصارًا.

```go
func TestBadBank(t *testing.T) {
	var (
		riya  = Account{Name: "Riya", Balance: 100}
		chris = Account{Name: "Chris", Balance: 75}
		adil  = Account{Name: "Adil", Balance: 200}

		transactions = []Transaction{
			NewTransaction(chris, riya, 100),
			NewTransaction(adil, chris, 25),
		}
	)

	newBalanceFor := func(account Account) float64 {
		return NewBalanceFor(account, transactions).Balance
	}

	AssertEqual(t, newBalanceFor(riya), 200)
	AssertEqual(t, newBalanceFor(chris), 0)
	AssertEqual(t, newBalanceFor(adil), 175)
}
```

وهذا هو الكود المحدَّث

```go
package main

type Transaction struct {
	From string
	To   string
	Sum  float64
}

func NewTransaction(from, to Account, sum float64) Transaction {
	return Transaction{From: from.Name, To: to.Name, Sum: sum}
}

type Account struct {
	Name    string
	Balance float64
}

func NewBalanceFor(account Account, transactions []Transaction) Account {
	return Reduce(
		transactions,
		applyTransaction,
		account,
	)
}

func applyTransaction(a Account, transaction Transaction) Account {
	if transaction.From == a.Name {
		a.Balance -= transaction.Sum
	}
	if transaction.To == a.Name {
		a.Balance += transaction.Sum
	}
	return a
}
```

أشعر أن هذا يُظهر حقًا قوة استخدام مفاهيم مثل `Reduce`. فـ `NewBalanceFor` تبدو أكثر _تصريحية_، إذ تصف _ماذا_ يحدث بدلًا من _كيف_. وغالبًا ما نقرأ الكود متنقلين بسرعة بين ملفات كثيرة، ونحاول فهم _ماذا_ يحدث لا _كيف_، وهذا الأسلوب من الكود يسهّل ذلك جيدًا.

وإذا أردت الغوص في التفاصيل أستطيع ذلك، فيمكنني رؤية _منطق العمل_ في `applyTransaction` دون القلق بشأن الحلقات وتغيير الحالة؛ فـ `Reduce` تتولى ذلك على حدة.


### الـ Fold/reduce عالميان إلى حد بعيد

الاحتمالات لا نهائية™️ مع `Reduce` (أو `Fold`). وهو نمط شائع لسبب ما، فليس مقتصرًا على الحساب أو دمج النصوص. جرّب بعض التطبيقات الأخرى.

- لمَ لا تمزج بعض قيم `color.RGBA` في لون واحد؟
- اجمع عدد الأصوات في تصويت، أو العناصر في سلة تسوق.
- كل ما يتعلق بمعالجة قائمة، تقريبًا.

## ‏`Find`

الآن وقد صار في Go الـ generics، وبدمجها مع الدوال ذات الرتبة الأعلى، يمكننا تقليل الكثير من الكود الرتيب المتكرر في مشاريعنا، لتكون أنظمتنا أسهل فهمًا وإدارة.

لم تعد بحاجة إلى كتابة دوال `Find` مخصصة لكل نوع من المجموعات تريد البحث فيه؛ بل أعد استخدام دالة `Find` أو اكتب واحدة. وإذا فهمت دالة `Reduce` أعلاه، فستكون كتابة دالة `Find` أمرًا بالغ السهولة.

وهذا اختبار

```go
func TestFind(t *testing.T) {
	t.Run("find first even number", func(t *testing.T) {
		numbers := []int{1, 2, 3, 4, 5, 6, 7, 8, 9, 10}

		firstEvenNumber, found := Find(numbers, func(x int) bool {
			return x%2 == 0
		})
		AssertTrue(t, found)
		AssertEqual(t, firstEvenNumber, 2)
	})
}
```

وهذا هو التنفيذ

```go
func Find[A any](items []A, predicate func(A) bool) (value A, found bool) {
	for _, v := range items {
		if predicate(v) {
			return v, true
		}
	}
	return
}
```

ومرة أخرى، لأنها تأخذ نوعًا من الـ generics، يمكننا إعادة استخدامها بطرق كثيرة

```go
type Person struct {
	Name string
}

t.Run("Find the best programmer", func(t *testing.T) {
	people := []Person{
		Person{Name: "Kent Beck"},
		Person{Name: "Martin Fowler"},
		Person{Name: "Chris James"},
	}

	king, found := Find(people, func(p Person) bool {
		return strings.Contains(p.Name, "Chris")
	})

	AssertTrue(t, found)
	AssertEqual(t, king, Person{Name: "Chris James"})
})
```

كما ترى، هذا الكود لا تشوبه شائبة.

## الخلاصة

عند استخدامها بذوق رفيع، ستجعل الدوال ذات الرتبة الأعلى كهذه كودك أبسط في القراءة والصيانة، لكن تذكّر القاعدة العامة:

استخدم عملية TDD لاستخلاص سلوك حقيقي ومحدد تحتاجه فعلًا، وفي مرحلة إعادة الهيكلة _قد_ تكتشف بعض التجريدات المفيدة التي تساعد على ترتيب الكود.

تدرّب على الجمع بين TDD وعادات جيدة في التحكم في الإصدارات. سجّل عملك في commit عندما ينجح اختبارك، _قبل_ أن تحاول إعادة الهيكلة. وبهذه الطريقة، إن أحدثت فوضى، يمكنك العودة بسهولة إلى حالتك العاملة.

### الأسماء مهمة

ابذل جهدًا في البحث خارج عالم Go، حتى لا تعيد اختراع أنماط موجودة بالفعل ولها اسم مستقر.

أتكتب دالة تأخذ مجموعة من النوع `A` وتحوّلها إلى النوع `B`؟ لا تسمِّها `Convert`، فهذا اسمه [`Map`](https://en.wikipedia.org/wiki/Map_(higher-order_function)). واستخدام الاسم "الصحيح" لهذه الأمور سيقلل العبء الذهني على الآخرين ويجعل البحث عنها في محركات البحث أسهل لتتعلم المزيد.

### ألا يبدو هذا أسلوبيًا (idiomatic)؟

جرّب أن تكون منفتح الذهن.

ومع أن أساليب Go لن تتغير _جذريًا_ بسبب إصدار الـ generics، ولا ينبغي أن تتغير، إلا أن الأساليب _ستتغير_ — لأن اللغة تتغير! ولا ينبغي أن يكون هذا نقطة خلافية.

أن تقول

> هذا ليس أسلوبيًا

دون أي تفصيل إضافي، ليس قولًا عمليًا ولا مفيدًا، خصوصًا عند مناقشة ميزات لغة جديدة.

ناقش زملاءك في أنماط وأساليب الكود بناءً على مزاياها لا على العقيدة. وطالما لديك اختبارات جيدة التصميم، فستتمكن دائمًا من إعادة الهيكلة وتغيير الأشياء كلما فهمت ما يناسبك ويناسب فريقك.

### مصادر

الـ Fold أساس حقيقي في علوم الحاسوب. وإليك بعض المصادر الشيقة إذا أردت التعمق فيه:
- [ويكيبيديا: Fold](https://en.wikipedia.org/wiki/Fold)
- [درس تعليمي عن عالمية الـ fold وقدرته التعبيرية](http://www.cs.nott.ac.uk/~pszgmh/fold.pdf)
