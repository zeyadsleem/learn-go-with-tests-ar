---
title: المؤشرات والأخطاء
weight: 70
---

# المؤشرات والأخطاء

**[يمكنك العثور على كل كود هذا الفصل هنا](https://github.com/quii/learn-go-with-tests/tree/main/pointers)**

تعرّفنا في القسم السابق على الـ structs التي تتيح لنا تجميع عدد من القيم المرتبطة بمفهوم واحد.

وقد ترغب في مرحلة ما في استخدام الـ structs لإدارة الحالة (state)، مع كشف methods تتيح للمستخدمين تغيير تلك الحالة بالطريقة التي تتحكم بها أنت.

**قطاع التقنية المالية (fintech) يعشق Go** و... البيتكوين؟ فلنُظهر إذن ما يمكننا بناؤه من نظام مصرفي مذهل.

لننشئ struct اسمه `Wallet` يتيح لنا إيداع `Bitcoin`.

## اكتب الاختبار أولًا

```go
func TestWallet(t *testing.T) {

	wallet := Wallet{}

	wallet.Deposit(10)

	got := wallet.Balance()
	want := 10

	if got != want {
		t.Errorf("got %d want %d", got, want)
	}
}
```

في [المثال السابق](structs-methods-and-interfaces.md) كنا نصل إلى الحقول مباشرة باسم الحقل، لكننا في _محفظتنا فائقة الأمان_ لا نريد كشف حالتها الداخلية للعالم كله. نريد التحكم في الوصول عبر الـ methods.

## جرّب تشغيل الاختبار

`./wallet_test.go:7:12: undefined: Wallet`

## اكتب أصغر قدر من الكود ليعمل الاختبار وافحص مخرجاته الفاشلة

المترجم لا يعرف ما هو `Wallet`، فلنخبره.

```go
type Wallet struct{}
```

أنشأنا محفظتنا الآن، فجرّب تشغيل الاختبار مرة أخرى

```
./wallet_test.go:9:8: wallet.Deposit undefined (type Wallet has no field or method Deposit)
./wallet_test.go:11:15: wallet.Balance undefined (type Wallet has no field or method Balance)
```

نحتاج إلى تعريف هذه الـ methods.

وتذكّر ألا تفعل إلا ما يكفي ليعمل الاختبار. فنحن نريد التأكد أن اختبارنا يفشل كما ينبغي مع رسالة خطأ واضحة.

```go
func (w Wallet) Deposit(amount int) {

}

func (w Wallet) Balance() int {
	return 0
}
```

إن كانت هذه الصياغة غير مألوفة لديك، فعُد واقرأ فصل الـ structs.

من المفترض أن تترجم الاختبارات الآن وتعمل

`wallet_test.go:15: got 0 want 10`

## اكتب كودًا كافيًا لنجاح الاختبار

سنحتاج إلى متغير ما للرصيد (_balance_) داخل الـ struct لتخزين الحالة

```go
type Wallet struct {
	balance int
}
```

في Go، إذا بدأ الرمز (variables, types, functions وغيرها) بحرف صغير، فهو خاص (private) _خارج الحزمة التي عُرّف فيها_.

وفي حالتنا نريد أن تستطيع الـ methods التعامل مع هذه القيمة، لا أن يفعل ذلك أي أحد آخر.

وتذكّر أننا نستطيع الوصول إلى الحقل `balance` الداخلي في الـ struct عبر متغير "المستقبِل" (receiver).

```go
func (w Wallet) Deposit(amount int) {
	w.balance += amount
}

func (w Wallet) Balance() int {
	return w.balance
}
```

وبعد أن اطمأنّا على مستقبلنا المهني في التقنية المالية، شغّل مجموعة الاختبارات وتنعّم بالاختبار الناجح

`wallet_test.go:15: got 0 want 10`

### هذا ليس صحيحًا تمامًا

هذا محيّر فعلًا؛ فالكود يبدو وكأنه ينبغي أن يعمل. نضيف المبلغ الجديد إلى الرصيد، ثم ينبغي أن تُرجع method الـ balance الحالة الحالية له.

في Go، **عندما تستدعي دالة أو method، تُنسَخ الوسائط** _**نسخًا كاملًا**_.

عند استدعاء `func (w Wallet) Deposit(amount int)` يكون `w` نسخة من الشيء الذي استدعينا الـ method منه.

ودون الدخول كثيرًا في علوم الحاسوب، عندما تنشئ قيمة — مثل محفظة — فسيُخزَّن في مكان ما بالذاكرة. ويمكنك معرفة _العنوان_ (address) لذلك الموضع من الذاكرة باستخدام `&myVal`.

جرّب أن تضيف بعض أوامر الطباعة إلى كودك

```go
func TestWallet(t *testing.T) {

	wallet := Wallet{}

	wallet.Deposit(10)

	got := wallet.Balance()

	fmt.Printf("address of balance in test is %p \n", &wallet.balance)

	want := 10

	if got != want {
		t.Errorf("got %d want %d", got, want)
	}
}
```

```go
func (w Wallet) Deposit(amount int) {
	fmt.Printf("address of balance in Deposit is %p \n", &w.balance)
	w.balance += amount
}
```

يطبع العنصر البديل `%p` عناوين الذاكرة بصيغة الأساس 16 مع بادئة `0x`، ويطبع محرف الهروب (escape character) سطرًا جديدًا. ولاحظ أننا نحصل على المؤشر (عنوان الذاكرة) لأي شيء بوضع محرف `&` في بداية الرمز.

الآن أعد تشغيل الاختبار

```
address of balance in Deposit is 0xc420012268
address of balance in test is 0xc420012260
```

كما ترى، عنوانا الرصيدين مختلفان. فعندما نغيّر قيمة الرصيد داخل الكود، نكون نعمل على نسخة من القيمة التي جاءت من الاختبار. لذلك يبقى الرصيد في الاختبار دون تغيير.

يمكننا إصلاح ذلك باستخدام _المؤشرات_ (pointers). فـ [المؤشرات](https://gobyexample.com/pointers) تتيح لنا الإشارة إلى قيم معينة ثم تغييرها. لذا بدلًا من أخذ نسخة من الـ Wallet بأكملها، نأخذ مؤشرًا إلى تلك المحفظة لنستطيع تغيير القيم الأصلية بداخلها.

```go
func (w *Wallet) Deposit(amount int) {
	w.balance += amount
}

func (w *Wallet) Balance() int {
	return w.balance
}
```

الفرق أن نوع المستقبِل هو `*Wallet` وليس `Wallet`، ويمكنك قراءة ذلك هكذا: "مؤشر إلى محفظة".

جرّب إعادة تشغيل الاختبارات، ومن المفترض أن تنجح.

وربما تتساءل الآن: لماذا نجحت؟ فنحن لم نفك الإشارة عن المؤشر (dereference) داخل الدالة، هكذا:

```go
func (w *Wallet) Balance() int {
	return (*w).balance
}
```

وبدا الأمر وكأننا خاطبنا الكائن مباشرة. والحقيقة أن الكود أعلاه باستخدام `(*w)` صحيح تمامًا. لكن صانعي Go رأوا هذه الصياغة مُرهقة، لذا تسمح لنا اللغة بكتابة `w.balance` دون فك إشارة صريح. بل إن هذه المؤشرات إلى الـ structs لها اسم خاص بها: _مؤشرات الـ structs_، وهي [تُفك إشارتها تلقائيًا](https://golang.org/ref/spec#Method_values).

وتقنيًا، لا تحتاج إلى تغيير `Balance` لتستخدم مستقبِلًا من نوع مؤشر، فأخذ نسخة من الرصيد لا مشكلة فيه. لكن من العُرف أن تُبقي أنواع مستقبِلات methods متطابقة للاتساق.

## إعادة الهيكلة

قلنا إننا نصنع محفظة بيتكوين، لكننا لم نذكر البيتكوين حتى الآن. فقد كنا نستخدم `int` لأنها نوع جيد لعدّ الأشياء!

ويبدو إنشاء `struct` لهذا الغرض مبالغة بعض الشيء. فـ `int` تؤدي الغرض في طريقة عملها، لكنها ليست معبّرة.

تتيح لك Go إنشاء أنواع (types) جديدة من أنواع موجودة.

الصياغة هي `type MyName OriginalType`

```go
type Bitcoin int

type Wallet struct {
	balance Bitcoin
}

func (w *Wallet) Deposit(amount Bitcoin) {
	w.balance += amount
}

func (w *Wallet) Balance() Bitcoin {
	return w.balance
}
```

```go
func TestWallet(t *testing.T) {

	wallet := Wallet{}

	wallet.Deposit(Bitcoin(10))

	got := wallet.Balance()

	want := Bitcoin(10)

	if got != want {
		t.Errorf("got %d want %d", got, want)
	}
}
```

لإنشاء `Bitcoin` تستخدم فقط الصياغة `Bitcoin(999)`.

وبهذه الطريقة ننشئ نوعًا جديدًا يمكننا تعريف _methods_ عليه. وهذا مفيد جدًا عندما تريد إضافة وظائف خاصة بمجال معين فوق أنواع موجودة.

لنُنفّذ [Stringer](https://golang.org/pkg/fmt/#Stringer) على Bitcoin

```go
type Stringer interface {
	String() string
}
```

هذه الواجهة (interface) معرَّفة في حزمة `fmt` وتتيح لك تحديد كيف يُطبع نوعك عند استخدامه مع نص التنسيق `%s` في أوامر الطباعة.

```go
func (b Bitcoin) String() string {
	return fmt.Sprintf("%d BTC", b)
}
```

وكما ترى، صياغة إنشاء method على تعريف نوع هي نفسها الصياغة على struct.

فقدنا الانضباط هنا: أضفنا method دون كتابة اختبار لها أولًا. لا بأس، فلسنا قديسين دائمًا، لكن لا ينبغي أن نترك الأمر يمر أيضًا. تشغيل `go test -cover` سيوضح لنا أن `String` غير مغطاة، وهذا دافع جيد للعودة والسؤال إن كان يستحق اختبارها بأثر رجعي. لا ينبغي أن نسعى إلى تغطية 100% لذاتها، لكن في هذه الحالة لـ `String` منطق خاص بها (`fmt.Sprintf`) يستحق التثبيت، فلنضف اختبارًا.

```go
t.Run("Bitcoin String", func(t *testing.T) {
	btc := Bitcoin(10)
	got := btc.String()
	want := "10 BTC"

	if got != want {
		t.Errorf("got %s want %s", got, want)
	}
})
```

بعد ذلك نحتاج إلى تحديث نصوص التنسيق في اختبارنا لتستخدم `String()` بدلًا من ذلك.

```go
if got != want {
	t.Errorf("got %s want %s", got, want)
}
```

لترى هذا عمليًا، اكسر الاختبار عن قصد لنرى النتيجة

`wallet_test.go:18: got 10 BTC want 20 BTC`

هذا يجعل ما يحدث في اختبارنا أوضح.

المتطلب التالي هو دالة `Withdraw`.

## اكتب الاختبار أولًا

عكس `Deposit()` تقريبًا

```go
func TestWallet(t *testing.T) {

	t.Run("deposit", func(t *testing.T) {
		wallet := Wallet{}

		wallet.Deposit(Bitcoin(10))

		got := wallet.Balance()

		want := Bitcoin(10)

		if got != want {
			t.Errorf("got %s want %s", got, want)
		}
	})

	t.Run("withdraw", func(t *testing.T) {
		wallet := Wallet{balance: Bitcoin(20)}

		wallet.Withdraw(Bitcoin(10))

		got := wallet.Balance()

		want := Bitcoin(10)

		if got != want {
			t.Errorf("got %s want %s", got, want)
		}
	})
}
```

## جرّب تشغيل الاختبار

`./wallet_test.go:26:9: wallet.Withdraw undefined (type Wallet has no field or method Withdraw)`

## اكتب أصغر قدر من الكود ليعمل الاختبار وافحص مخرجاته الفاشلة

```go
func (w *Wallet) Withdraw(amount Bitcoin) {

}
```

`wallet_test.go:33: got 20 BTC want 10 BTC`

## اكتب كودًا كافيًا لنجاح الاختبار

```go
func (w *Wallet) Withdraw(amount Bitcoin) {
	w.balance -= amount
}
```

## إعادة الهيكلة

هناك بعض التكرار في اختباراتنا، فلنُزله بإعادة الهيكلة.

```go
func TestWallet(t *testing.T) {

	assertBalance := func(t testing.TB, wallet Wallet, want Bitcoin) {
		t.Helper()
		got := wallet.Balance()

		if got != want {
			t.Errorf("got %s want %s", got, want)
		}
	}

	t.Run("deposit", func(t *testing.T) {
		wallet := Wallet{}
		wallet.Deposit(Bitcoin(10))
		assertBalance(t, wallet, Bitcoin(10))
	})

	t.Run("withdraw", func(t *testing.T) {
		wallet := Wallet{balance: Bitcoin(20)}
		wallet.Withdraw(Bitcoin(10))
		assertBalance(t, wallet, Bitcoin(10))
	})

}
```

ماذا ينبغي أن يحدث إذا حاولت `Withdraw` مبلغًا أكبر من المتبقي في الحساب؟ متطلبنا الآن هو افتراض عدم وجود خدمة سحب على المكشوف (overdraft).

كيف نُشير إلى وجود مشكلة عند استخدام `Withdraw`؟

في Go، إذا أردت الإشارة إلى خطأ، فمن العُرف أن تُرجع دالتك `err` ليتحقق منها المستدعي ويتصرف بناءً عليها.

لنجرّب ذلك في اختبار.

## اكتب الاختبار أولًا

```go
t.Run("withdraw insufficient funds", func(t *testing.T) {
	startingBalance := Bitcoin(20)
	wallet := Wallet{startingBalance}
	err := wallet.Withdraw(Bitcoin(100))

	assertBalance(t, wallet, startingBalance)

	if err == nil {
		t.Error("wanted an error but didn't get one")
	}
})
```

نريد أن تُرجع `Withdraw` خطأ _إذا_ حاولت سحب أكثر مما لديك، وأن يبقى الرصيد كما هو.

ثم نتحقق من أن خطأ قد أُرجع بإفشال الاختبار إذا كان `nil`.

إن `nil` مرادف لـ `null` في لغات البرمجة الأخرى. ويمكن أن تكون الأخطاء `nil` لأن نوع إرجاع `Withdraw` سيكون `error`، وهي واجهة (interface). وإذا رأيت دالة تأخذ وسائط أو تُرجع قيمًا من نوع واجهة، فيمكن أن تكون قابلة لتكون nil.

ومثل `null`، إذا حاولت الوصول إلى قيمة هي `nil` فستُطلق **panic وقت التشغيل (runtime panic)**. وهذا سيئ! لذا تأكد من فحص القيم بحثًا عن nil.

## جرّب تشغيل الاختبار

`./wallet_test.go:31:25: wallet.Withdraw(Bitcoin(100)) used as value`

الصياغة غير واضحة قليلًا ربما، لكن نيتنا السابقة مع `Withdraw` كانت مجرد استدعائها، فهي لا تُرجع قيمة أبدًا. ولكي يُترجم هذا الكود سنحتاج إلى تغييرها ليكون لها نوع إرجاع.

## اكتب أصغر قدر من الكود ليعمل الاختبار وافحص مخرجاته الفاشلة

```go
func (w *Wallet) Withdraw(amount Bitcoin) error {
	w.balance -= amount
	return nil
}
```

مجددًا، من المهم جدًا ألا تكتب إلا كودًا كافيًا لإرضاء المترجم. نصحّح method الـ `Withdraw` لتُرجع `error`، وعليها الآن أن تُرجع _شيئًا ما_، فلنُرجِع `nil`.

## اكتب كودًا كافيًا لنجاح الاختبار

```go
func (w *Wallet) Withdraw(amount Bitcoin) error {

	if amount > w.balance {
		return errors.New("oh no")
	}

	w.balance -= amount
	return nil
}
```

تذكّر استيراد `errors` في كودك.

تنشئ `errors.New` خطأ (`error`) جديدًا برسالة من اختيارك.

## إعادة الهيكلة

لنصنع دالة مساعدة سريعة لفحص الخطأ لتحسين قراءة الاختبار

```go
assertError := func(t testing.TB, err error) {
	t.Helper()
	if err == nil {
		t.Error("wanted an error but didn't get one")
	}
}
```

وفي اختبارنا

```go
t.Run("withdraw insufficient funds", func(t *testing.T) {
	startingBalance := Bitcoin(20)
	wallet := Wallet{startingBalance}
	err := wallet.Withdraw(Bitcoin(100))

	assertError(t, err)
	assertBalance(t, wallet, startingBalance)
})
```

آمل أنك فكرت، عند إرجاع خطأ "oh no"، أننا _قد_ نحسّنه لاحقًا لأنه لا يبدو مفيدًا بما يكفي لإرجاعه.

بافتراض أن الخطأ سيُعاد في النهاية إلى المستخدم، فلنحدّث اختبارنا ليتحقق من رسالة خطأ معينة بدلًا من مجرد وجود خطأ.

## اكتب الاختبار أولًا

لنحدّث دالتنا المساعدة لتقارن مع `string`.

```go
assertError := func(t testing.TB, got error, want string) {
	t.Helper()

	if got == nil {
		t.Fatal("didn't get an error but wanted one")
	}

	if got.Error() != want {
		t.Errorf("got %q, want %q", got, want)
	}
}
```

وكما ترى، يمكن تحويل الأخطاء (`Errors`) إلى نص عبر method الـ `.Error()`، وهذا ما نفعله لمقارنتها بالنص الذي نريده. ونتأكد أيضًا أن الخطأ ليس `nil` حتى لا نستدعي `.Error()` على `nil`.

ثم نحدّث موضع الاستدعاء

```go
t.Run("withdraw insufficient funds", func(t *testing.T) {
	startingBalance := Bitcoin(20)
	wallet := Wallet{startingBalance}
	err := wallet.Withdraw(Bitcoin(100))

	assertError(t, err, "cannot withdraw, insufficient funds")
	assertBalance(t, wallet, startingBalance)
})
```

قدّمنا هنا `t.Fatal` التي توقف الاختبار إذا استُدعيت. فالسبب أننا لا نريد إجراء مزيد من التحققات على الخطأ المُرجَع إذا لم يكن هناك خطأ أصلًا. وبدون ذلك سيمضي الاختبار إلى الخطوة التالية ويُطلق panic بسبب مؤشر nil.

## جرّب تشغيل الاختبار

`wallet_test.go:61: got err 'oh no' want 'cannot withdraw, insufficient funds'`

## اكتب كودًا كافيًا لنجاح الاختبار

```go
func (w *Wallet) Withdraw(amount Bitcoin) error {

	if amount > w.balance {
		return errors.New("cannot withdraw, insufficient funds")
	}

	w.balance -= amount
	return nil
}
```

## إعادة الهيكلة

لدينا تكرار لرسالة الخطأ في كود الاختبار وكود `Withdraw` معًا.

سيكون مزعجًا جدًا أن يفشل الاختبار لمجرد أن أحدهم أراد إعادة صياغة نص الخطأ، كما أن هذا تفصيل مبالغ فيه لاختبارنا. فنحن لا نهتم _حقًا_ بالصياغة الدقيقة، بل بأن يُرجع خطأ ذو معنى يتعلق بالسحب عند تحقق شرط معين.

في Go، الأخطاء قيم (errors are values)، لذا يمكننا إخراجها في متغير ليكون لدينا مصدر واحد للحقيقة.

```go
var ErrInsufficientFunds = errors.New("cannot withdraw, insufficient funds")

func (w *Wallet) Withdraw(amount Bitcoin) error {

	if amount > w.balance {
		return ErrInsufficientFunds
	}

	w.balance -= amount
	return nil
}
```

تتيح لنا الكلمة المفتاحية `var` تعريف قيم عامة على مستوى الحزمة.

وهذا تغيير إيجابي بحد ذاته، فدالة `Withdraw` الآن تبدو واضحة جدًا.

بعدها يمكننا إعادة هيكلة كود الاختبار ليستخدم هذه القيمة بدلًا من نصوص محددة.

```go
func TestWallet(t *testing.T) {

	t.Run("deposit", func(t *testing.T) {
		wallet := Wallet{}
		wallet.Deposit(Bitcoin(10))
		assertBalance(t, wallet, Bitcoin(10))
	})

	t.Run("withdraw with funds", func(t *testing.T) {
		wallet := Wallet{Bitcoin(20)}
		wallet.Withdraw(Bitcoin(10))
		assertBalance(t, wallet, Bitcoin(10))
	})

	t.Run("withdraw insufficient funds", func(t *testing.T) {
		wallet := Wallet{Bitcoin(20)}
		err := wallet.Withdraw(Bitcoin(100))

		assertError(t, err, ErrInsufficientFunds)
		assertBalance(t, wallet, Bitcoin(20))
	})
}

func assertBalance(t testing.TB, wallet Wallet, want Bitcoin) {
	t.Helper()
	got := wallet.Balance()

	if got != want {
		t.Errorf("got %q want %q", got, want)
	}
}

func assertError(t testing.TB, got, want error) {
	t.Helper()
	if got == nil {
		t.Fatal("didn't get an error but wanted one")
	}

	if got != want {
		t.Errorf("got %q, want %q", got, want)
	}
}
```

والآن أصبح تتبّع الاختبار أسهل أيضًا.

لقد نقلتُ الدوال المساعدة خارج دالة الاختبار الرئيسية كي يبدأ أي شخص يفتح الملف بقراءة التحققات (assertions) أولًا، بدلًا من بعض الدوال المساعدة.

ومن الخصائص المفيدة للاختبارات أنها تساعدنا على فهم _الاستخدام الحقيقي_ لكودنا لنكتب كودًا يتعاطف مع طريقة استخدامه. ونرى هنا أن المطور يمكنه ببساطة استدعاء كودنا وإجراء مقارنة مساواة مع `ErrInsufficientFunds` والتصرف بناءً عليها.

### الأخطاء غير المفحوصة

ورغم أن مترجم Go يساعدك كثيرًا، فأحيانًا تظل هناك أشياء قد تفوتك، وقد تكون معالجة الأخطاء شائكة أحيانًا.

هناك سيناريو واحد لم نختبره. للعثور عليه، شغّل ما يلي في الطرفية لتثبيت `errcheck`، وهو واحد من مدققات الكود (linters) الكثيرة المتاحة لـ Go.

`go install github.com/kisielk/errcheck@latest`

ثم شغّل `errcheck .` داخل المجلد الذي يحوي كودك

ومن المفترض أن تحصل على شيء مثل

`wallet_test.go:17:18: wallet.Withdraw(Bitcoin(10))`

ما تخبرنا به هذه الرسالة أننا لم نفحص الخطأ المُرجَع في ذلك السطر. ويقابل ذلك السطر في جهازي سيناريو السحب العادي؛ لأننا لم نتحقق من أنه إذا نجح `Withdraw` فإن خطأً _لا_ يُرجع.

وهذا هو كود الاختبار النهائي الذي يراعي ذلك.

```go
func TestWallet(t *testing.T) {

	t.Run("deposit", func(t *testing.T) {
		wallet := Wallet{}
		wallet.Deposit(Bitcoin(10))

		assertBalance(t, wallet, Bitcoin(10))
	})

	t.Run("withdraw with funds", func(t *testing.T) {
		wallet := Wallet{Bitcoin(20)}
		err := wallet.Withdraw(Bitcoin(10))

		assertNoError(t, err)
		assertBalance(t, wallet, Bitcoin(10))
	})

	t.Run("withdraw insufficient funds", func(t *testing.T) {
		wallet := Wallet{Bitcoin(20)}
		err := wallet.Withdraw(Bitcoin(100))

		assertError(t, err, ErrInsufficientFunds)
		assertBalance(t, wallet, Bitcoin(20))
	})
}

func assertBalance(t testing.TB, wallet Wallet, want Bitcoin) {
	t.Helper()
	got := wallet.Balance()

	if got != want {
		t.Errorf("got %s want %s", got, want)
	}
}

func assertNoError(t testing.TB, got error) {
	t.Helper()
	if got != nil {
		t.Fatal("got an error but didn't want one")
	}
}

func assertError(t testing.TB, got error, want error) {
	t.Helper()
	if got == nil {
		t.Fatal("didn't get an error but wanted one")
	}

	if got != want {
		t.Errorf("got %s, want %s", got, want)
	}
}
```

## إضافة السياق عبر تغليف الأخطاء

مقارنة `err` بـ `ErrInsufficientFunds` تعمل هنا بشكل جيد لأن `Withdraw` هي الشيء الوحيد الذي يمكن أن ينتجه. لكن البرامج الحقيقية لها طبقات — فقد يُستخدم `Wallet` داخل `Bank`، الذي يُستخدم بدوره داخل معالج HTTP (HTTP handler)، وهكذا. وإذا كانت كل طبقة تُرجع الخطأ الذي استقبلته كما هو دون تغيير، فلن يرى المستدعي الأعلى بطبقات عديدة سوى `insufficient funds`، دون أي فكرة عن الحساب أو العملية التي فشلت فعلًا.

لنفترض أن لدينا دالة تعالج عملية سحب لحساب باسم معين:

```go
func ProcessWithdrawal(wallet *Wallet, accountID string, amount Bitcoin) error {
	if err := wallet.Withdraw(amount); err != nil {
		return fmt.Errorf("processing withdrawal for account %s: %w", accountID, err)
	}
	return nil
}
```

الفعل `%w` (الذي ظهر في Go 1.13) يشبه `%v` في أنه يُدرج نص `err`، لكنه يفعل أيضًا شيئًا لا يفعله `%v`: فهو _يغلّف_ `err` داخل الخطأ الجديد الذي تُرجعه `fmt.Errorf`، بدلًا من مجرد نسخ رسالته إلى نص جديد غير مرتبط. جرّب ذلك:

```go
wallet := Wallet{Bitcoin(10)}
err := ProcessWithdrawal(&wallet, "acc-123", Bitcoin(100))
fmt.Println(err)
// processing withdrawal for account acc-123: cannot withdraw, insufficient funds
```

لقد حافظنا على السياق المفيد ("أي حساب، وأي عملية") دون أن نفقد تفصيل ما حدث فعلًا تحت السطح.

### فحص الأخطاء المغلَّفة

الآن وقد أصبحت `ProcessWithdrawal` تُرجع قيمة خطأ _مختلفة_ عن `ErrInsufficientFunds`، فإن المقارنة بـ `==` (أو بدالتنا المساعدة `assertError` أعلاه) ستفشل، مع أن السبب الكامن هو نفسه. وهذه بالضبط هي المشكلة التي تحلها [`errors.Is`](https://pkg.go.dev/errors#Is) — فهي تفحص سلسلة الأخطاء المغلَّفة، لا الخطأ الخارجي فقط:

```go
if errors.Is(err, ErrInsufficientFunds) {
	// still true, even though err's message now also mentions the account
}
```

تعمل `errors.Is` باستدعاء [`errors.Unwrap`](https://pkg.go.dev/errors#Unwrap) مرارًا على `err` (وهي تعرف كيف تستخرج الخطأ المغلَّف، لأن `fmt.Errorf` مع `%w` تنتج قيمة لها method اسمها `Unwrap() error`) حتى تجد تطابقًا أو تنفد الأخطاء التي يمكن فكها. وسترى `errors.Is` مستخدمة مرة أخرى لهذا السبب نفسه في دالة `assertError` المساعدة في فصل [Maps](maps.md) بدلًا من `==` — فهي خيار افتراضي آمن يعمل أيضًا مع الأخطاء التي لم تُغلَّف أصلًا.

وإذا احتجت إلى استخراج خطأ _ذا نوع_ (typed) (بدلًا من المقارنة مع قيمة إرشادية (sentinel value) مثل `ErrInsufficientFunds`) من سلسلة أخطاء مغلَّفة، فهناك دالة مكافئة لذلك أيضًا: [`errors.As`](https://pkg.go.dev/errors#As). ويغطي فصل [أنواع الأخطاء](../questions-and-answers/error-types.md) ذلك بتعمق أكبر.

## الخلاصة

### المؤشرات

* تنسخ Go القيم عندما تمررها إلى الدوال/الـ methods، لذا إذا كنت تكتب دالة تحتاج إلى تغيير الحالة (mutate state) فعليها أن تأخذ مؤشرًا إلى الشيء الذي تريد تغييره.
* كون Go تنسخ القيم مفيد في كثير من الأحيان، لكنك أحيانًا لا تريد أن ينسخ نظامك شيئًا ما، وعندها تحتاج إلى تمرير مرجع (reference). ومن الأمثلة الإشارة إلى بنى بيانات ضخمة جدًا أو أشياء تكفي منها نسخة واحدة (مثل مجموعات اتصالات قواعد البيانات).

### nil

* يمكن أن تكون المؤشرات nil
* عندما تُرجع دالة مؤشرًا إلى شيء ما، عليك التأكد من فحص ما إذا كان nil، وإلا فقد تُطلق استثناءً وقت التشغيل — ولن يساعدك المترجم هنا.
* مفيد عندما تريد وصف قيمة قد تكون مفقودة

### الأخطاء

* الأخطاء هي طريقة الإشارة إلى الفشل عند استدعاء دالة/method.
* بالإصغاء إلى اختباراتنا استنتجنا أن التحقق من نص داخل خطأ سيؤدي إلى اختبار غير مستقر (flaky test). لذا أعدنا هيكلة تنفيذنا ليستخدم قيمة ذات معنى بدلًا من ذلك، فكان الكود أسهل في الاختبار، وخَلُصنا إلى أن ذلك سيكون أسهل أيضًا لمستخدمي واجهتنا البرمجية (API).
* لم تنتهِ الحكاية مع معالجة الأخطاء عند هذا الحد، فيمكنك فعل أشياء أكثر تطورًا، لكن هذه مجرد مقدمة. وستغطي أقسام لاحقة استراتيجيات أكثر.
* [لا تكتفِ بفحص الأخطاء، بل عالجها برشاقة](https://dave.cheney.net/2016/04/27/dont-just-check-errors-handle-them-gracefully)

### إنشاء أنواع جديدة من الأنواع الموجودة

* مفيد لإضافة معنى أقرب لمجال معين إلى القيم
* يتيح لك تنفيذ الواجهات (interfaces)

المؤشرات والأخطاء جزء كبير من كتابة Go تحتاج إلى الاعتياد عليه. ولحسن الحظ سيساعدك المترجم _عادةً_ إذا أخطأت في شيء ما؛ فقط خذ وقتك واقرأ الخطأ.
