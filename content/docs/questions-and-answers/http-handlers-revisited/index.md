---
title: إعادة زيارة معالجات HTTP
weight: 370
---

# إعادة زيارة معالجات HTTP

**[يمكنك العثور على كل كود هذا الفصل هنا](https://github.com/quii/learn-go-with-tests/tree/main/q-and-a/http-handlers-revisited)**

يحتوي هذا الكتاب بالفعل على فصل عن [اختبار معالج HTTP](../build-an-application/http-server.md)، لكن هذا الفصل سيقدّم نقاشًا أوسع حول تصميم المعالجات، ليكون اختبارها بسيطًا.

سنلقي نظرة على مثال واقعي، وكيف يمكننا تحسين طريقة تصميمه بتطبيق مبادئ مثل مبدأ المسؤولية الواحدة (single responsibility principle) وفصل الاهتمامات (separation of concerns). ويمكن تحقيق هذه المبادئ باستخدام [الواجهات (interfaces)](../go-fundamentals/structs-methods-and-interfaces.md) و[حقن الاعتماديات (dependency injection)](../go-fundamentals/dependency-injection.md). وبهذا سنوضح أن اختبار المعالجات أمر بسيط للغاية في الحقيقة.

![رسم يوضح سؤالًا شائعًا في مجتمع Go](amazing-art.png)

يبدو أن اختبار معالجات HTTP سؤال متكرر في مجتمع Go، وأعتقد أنه يشير إلى مشكلة أوسع هي أن الناس يسيئون فهم كيفية تصميمها.

كثيرًا ما تنبع صعوبات الناس في الاختبار من تصميم كودهم لا من كتابة الاختبارات نفسها. وكما أؤكد كثيرًا في هذا الكتاب:

> إذا كانت اختباراتك تسبب لك الألم، فأصغِ إلى هذه الإشارة وفكّر في تصميم كودك.

## مثال

[غرّد لي Santosh Kumar على تويتر](https://twitter.com/sntshk/status/1255559003339284481)

> كيف أختبر معالج HTTP يعتمد على mongodb؟

إليك الكود

```go
func Registration(w http.ResponseWriter, r *http.Request) {
	var res model.ResponseResult
	var user model.User

	w.Header().Set("Content-Type", "application/json")

	jsonDecoder := json.NewDecoder(r.Body)
	jsonDecoder.DisallowUnknownFields()
	defer r.Body.Close()

	// check if there is proper json body or error
	if err := jsonDecoder.Decode(&user); err != nil {
		res.Error = err.Error()
		// return 400 status codes
		w.WriteHeader(http.StatusBadRequest)
		json.NewEncoder(w).Encode(res)
		return
	}

	// Connect to mongodb
	client, _ := mongo.NewClient(options.Client().ApplyURI("mongodb://127.0.0.1:27017"))
	ctx, _ := context.WithTimeout(context.Background(), 10*time.Second)
	err := client.Connect(ctx)
	if err != nil {
		panic(err)
	}
	defer client.Disconnect(ctx)
	// Check if username already exists in users datastore, if so, 400
	// else insert user right away
	collection := client.Database("test").Collection("users")
	filter := bson.D{{"username", user.Username}}
	var foundUser model.User
	err = collection.FindOne(context.TODO(), filter).Decode(&foundUser)
	if foundUser.Username == user.Username {
		res.Error = UserExists
		// return 400 status codes
		w.WriteHeader(http.StatusBadRequest)
		json.NewEncoder(w).Encode(res)
		return
	}

	pass, err := bcrypt.GenerateFromPassword([]byte(user.Password), bcrypt.DefaultCost)
	if err != nil {
		res.Error = err.Error()
		// return 400 status codes
		w.WriteHeader(http.StatusBadRequest)
		json.NewEncoder(w).Encode(res)
		return
	}
	user.Password = string(pass)

	insertResult, err := collection.InsertOne(context.TODO(), user)
	if err != nil {
		res.Error = err.Error()
		// return 400 status codes
		w.WriteHeader(http.StatusBadRequest)
		json.NewEncoder(w).Encode(res)
		return
	}

	// return 200
	w.WriteHeader(http.StatusOK)
	res.Result = fmt.Sprintf("%s: %s", UserCreated, insertResult.InsertedID)
	json.NewEncoder(w).Encode(res)
	return
}
```

لنُعدد فقط كل الأشياء التي يجب أن تفعلها هذه الدالة الواحدة:

1. كتابة استجابات HTTP وإرسال الترويسات (headers) وأكواد الحالة (status codes) وما إلى ذلك.
2. فك ترميز جسم الطلب (request body) إلى `User`
3. الاتصال بقاعدة بيانات (وكل التفاصيل المتعلقة بذلك)
4. الاستعلام من قاعدة البيانات وتطبيق بعض منطق العمل (business logic) حسب النتيجة
5. توليد كلمة مرور
6. إدراج سجل (record)

هذا كثير جدًا.

## ما هو معالج HTTP وماذا ينبغي أن يفعل؟

إذا نسيّنا تفاصيل Go المحددة للحظة، فمهما كانت اللغة التي عملت بها، ما خدمني دائمًا هو التفكير في [فصل الاهتمامات](https://en.wikipedia.org/wiki/Separation_of_concerns) و[مبدأ المسؤولية الواحدة](https://en.wikipedia.org/wiki/Single-responsibility_principle).

قد يكون تطبيق ذلك صعبًا بعض الشيء حسب المشكلة التي تحلها. فما هي المسؤولية _بالضبط_؟

قد تتلاشى الحدود حسب مستوى التجريد الذي تفكر به، وأحيانًا قد لا يكون تخمينك الأول صحيحًا.

لحسن الحظ، ففي حالة معالجات HTTP أشعر أن لديّ فكرة جيدة إلى حد كبير عما ينبغي أن تفعله، مهما كان المشروع الذي عملت عليه:

1. استقبال طلب HTTP وتحليله والتحقق من صحته.
2. استدعاء شيء ما مثل `ServiceThing` لتنفيذ `ImportantBusinessLogic` على البيانات التي حصلت عليها من الخطوة الأولى.
3. إرسال استجابة `HTTP` مناسبة حسب ما يرجعه `ServiceThing`.

لا أقول إن كل معالج HTTP _على الإطلاق_ ينبغي أن يكون بهذا الشكل تقريبًا، لكن في 99 حالة من كل 100 يبدو الأمر كذلك بالنسبة لي.

عندما تفصل هذه الاهتمامات:

* يصبح اختبار المعالجات سهلًا جدًا ويركز على عدد صغير من الاهتمامات.
* والأهم أن اختبار `ImportantBusinessLogic` لم يعد مضطرًا للانشغال بـ `HTTP`، فيمكنك اختبار منطق العمل بنقاء.
* ويمكنك استخدام `ImportantBusinessLogic` في سياقات أخرى دون الحاجة إلى تعديله.
* وإذا غيّر `ImportantBusinessLogic` ما يفعله، فلن تحتاج إلى تغيير معالجاتك ما دامت الواجهة (interface) كما هي.

## معالجات Go

[`http.HandlerFunc`](https://golang.org/pkg/net/http/#HandlerFunc)

> النوع HandlerFunc هو مهايئ (adapter) يسمح باستخدام الدوال العادية كمعالجات HTTP.

`type HandlerFunc func(ResponseWriter, *Request)`

أيها القارئ، خذ نفسًا عميقًا وانظر إلى الكود أعلاه. ماذا تلاحظ؟

**إنها دالة تأخذ بعض الوسائط**

لا يوجد سحر أطر عمل، ولا تعليقات توضيحية (annotations)، ولا حبوب سحرية، ولا شيء.

إنها مجرد دالة، _ونحن نعرف كيف نختبر الدوال_.

وهي تنسجم تمامًا مع التعليق أعلاه:

* تأخذ [`http.Request`](https://golang.org/pkg/net/http/#Request) وهي مجرد حزمة من البيانات لنفحصها ونحللها ونتحقق من صحتها.
* > [تُستخدم واجهة `http.ResponseWriter` من قبل معالج HTTP لبناء استجابة HTTP.](https://golang.org/pkg/net/http/#ResponseWriter)

### مثال اختبار بسيط للغاية

```go
func Teapot(res http.ResponseWriter, req *http.Request) {
	res.WriteHeader(http.StatusTeapot)
}

func TestTeapotHandler(t *testing.T) {
	req := httptest.NewRequest(http.MethodGet, "/", nil)
	res := httptest.NewRecorder()

	Teapot(res, req)

	if res.Code != http.StatusTeapot {
		t.Errorf("got status %d but wanted %d", res.Code, http.StatusTeapot)
	}
}
```

لاختبار دالتنا، _نستدعيها_.

في اختبارنا نمرر `httptest.ResponseRecorder` كوسيط `http.ResponseWriter`، وستستخدمه دالتنا لكتابة استجابة `HTTP`. وسيسجّل المسجّل (recorder) ما أُرسل (أو _يتجسس_ عليه)، ثم يمكننا إجراء تحققاتنا.

## استدعاء `ServiceThing` في معالجنا

من الشكاوى الشائعة عن دروس التطوير الموجه بالاختبار أنها دائمًا "بسيطة جدًا" وليست "واقعية بما يكفي". وجوابي على ذلك هو:

> ألن يكون جميلًا لو كان كل كودك بسيطًا في قراءته واختباره مثل الأمثلة التي تذكرها؟

هذا أحد أكبر التحديات التي نواجهها، لكن علينا أن نواصل السعي إليه. فمن _الممكن_ تصميم الكود (وإن لم يكن ذلك سهلًا بالضرورة) ليكون بسيطًا في قراءته واختباره، إذا تدرّبنا وطبّقنا مبادئ هندسة برمجيات جيدة.

لنستعرض سريعًا ما يفعله المعالج من قبل:

1. كتابة استجابات HTTP وإرسال الترويسات وأكواد الحالة وما إلى ذلك.
2. فك ترميز جسم الطلب إلى `User`
3. الاتصال بقاعدة بيانات (وكل التفاصيل المتعلقة بذلك)
4. الاستعلام من قاعدة البيانات وتطبيق بعض منطق العمل حسب النتيجة
5. توليد كلمة مرور
6. إدراج سجل

وباعتناق فكرة فصل الاهتمامات المثالي أكثر، كنت أرغب في أن يكون أشبه بهذا:

1. فك ترميز جسم الطلب إلى `User`
2. استدعاء `UserService.Register(user)` (وهذا هو `ServiceThing` لدينا)
3. إذا حدث خطأ فنتصرف بناءً عليه (فالمثال يرسل دائمًا `400 BadRequest` ولا أظن ذلك صحيحًا)، وسأكتفي _في الوقت الحالي_ بمعالج شامل يعيد `500 Internal Server Error`. ويجب أن أؤكد أن إرجاع `500` لكل الأخطاء يجعل الواجهة البرمجية (API) سيئة للغاية! ويمكننا لاحقًا جعل معالجة الأخطاء أكثر تطورًا، ربما باستخدام [أنواع الأخطاء](error-types.md).
4. إذا لم يحدث خطأ، نعيد `201 Created` مع المعرّف (ID) كجسم للاستجابة (مرة أخرى للاختصار/الكسل)

اختصارًا للوقت لن أستعرض عملية التطوير الموجه بالاختبار المعتادة، فراجع بقية الفصول للأمثلة.

### تصميم جديد

```go
type UserService interface {
	Register(user User) (insertedID string, err error)
}

type UserServer struct {
	service UserService
}

func NewUserServer(service UserService) *UserServer {
	return &UserServer{service: service}
}

func (u *UserServer) RegisterUser(w http.ResponseWriter, r *http.Request) {
	defer r.Body.Close()

	// request parsing and validation
	var newUser User
	err := json.NewDecoder(r.Body).Decode(&newUser)

	if err != nil {
		http.Error(w, fmt.Sprintf("could not decode user payload: %v", err), http.StatusBadRequest)
		return
	}

	// call a service thing to take care of the hard work
	insertedID, err := u.service.Register(newUser)

	// depending on what we get back, respond accordingly
	if err != nil {
		//todo: handle different kinds of errors differently
		http.Error(w, fmt.Sprintf("problem registering new user: %v", err), http.StatusInternalServerError)
		return
	}

	w.WriteHeader(http.StatusCreated)
	fmt.Fprint(w, insertedID)
}
```

تطابق دالتنا (method) `RegisterUser` شكل `http.HandlerFunc`، لذا نحن جاهزون. وقد أرفقناها كـ method على نوع جديد هو `UserServer` يحتوي على اعتمادية على `UserService` مُلتقَطة كواجهة (interface).

الواجهات (interfaces) طريقة رائعة لضمان فصل اهتماماتنا المتعلقة بـ `HTTP` عن أي تنفيذ (implementation) محدد؛ فيمكننا فقط استدعاء الـ method على الاعتمادية، دون أن يهمنا _كيف_ يُسجَّل المستخدم.

وإذا أردت استكشاف هذا الأسلوب بمزيد من التفصيل عبر التطوير الموجه بالاختبار، فاقرأ فصل [حقن الاعتماديات](../go-fundamentals/dependency-injection.md) وفصل [خادم HTTP من قسم "ابنِ تطبيقًا"](../build-an-application/http-server.md).

وبعد أن فصلنا أنفسنا عن أي تفصيل تنفيذي محدد يتعلق بالتسجيل، أصبحت كتابة كود معالجنا مباشرة وتتبع المسؤوليات الموصوفة سابقًا.

### الاختبارات!

تتجلّى هذه البساطة في اختباراتنا.

```go
type MockUserService struct {
	RegisterFunc    func(user User) (string, error)
	UsersRegistered []User
}

func (m *MockUserService) Register(user User) (insertedID string, err error) {
	m.UsersRegistered = append(m.UsersRegistered, user)
	return m.RegisterFunc(user)
}

func TestRegisterUser(t *testing.T) {
	t.Run("can register valid users", func(t *testing.T) {
		user := User{Name: "CJ"}
		expectedInsertedID := "whatever"

		service := &MockUserService{
			RegisterFunc: func(user User) (string, error) {
				return expectedInsertedID, nil
			},
		}
		server := NewUserServer(service)

		req := httptest.NewRequest(http.MethodGet, "/", userToJSON(user))
		res := httptest.NewRecorder()

		server.RegisterUser(res, req)

		assertStatus(t, res.Code, http.StatusCreated)

		if res.Body.String() != expectedInsertedID {
			t.Errorf("expected body of %q but got %q", res.Body.String(), expectedInsertedID)
		}

		if len(service.UsersRegistered) != 1 {
			t.Fatalf("expected 1 user added but got %d", len(service.UsersRegistered))
		}

		if !reflect.DeepEqual(service.UsersRegistered[0], user) {
			t.Errorf("the user registered %+v was not what was expected %+v", service.UsersRegistered[0], user)
		}
	})

	t.Run("returns 400 bad request if body is not valid user JSON", func(t *testing.T) {
		server := NewUserServer(nil)

		req := httptest.NewRequest(http.MethodGet, "/", strings.NewReader("trouble will find me"))
		res := httptest.NewRecorder()

		server.RegisterUser(res, req)

		assertStatus(t, res.Code, http.StatusBadRequest)
	})

	t.Run("returns a 500 internal server error if the service fails", func(t *testing.T) {
		user := User{Name: "CJ"}

		service := &MockUserService{
			RegisterFunc: func(user User) (string, error) {
				return "", errors.New("couldn't add new user")
			},
		}
		server := NewUserServer(service)

		req := httptest.NewRequest(http.MethodGet, "/", userToJSON(user))
		res := httptest.NewRecorder()

		server.RegisterUser(res, req)

		assertStatus(t, res.Code, http.StatusInternalServerError)
	})
}
```

وبما أن معالجنا لم يعد مرتبطًا بتنفيذ محدد للتخزين، يصبح من السهل جدًا أن نكتب `MockUserService` ليساعدنا على كتابة اختبارات وحدة بسيطة وسريعة تمارس المسؤوليات المحددة التي يتحملها.

### ماذا عن كود قاعدة البيانات؟ أنت تغش!

كل هذا مقصود تمامًا. فنحن لا نريد أن تنشغل معالجات HTTP بمنطق العمل أو قواعد البيانات أو الاتصالات وما إلى ذلك.

وبفعل ذلك حرّرنا المعالج من التفاصيل الفوضوية، و_كذلك_ جعلنا اختبار طبقة الاستمرارية (persistence layer) ومنطق العمل أسهل، لأنها أيضًا لم تعد مرتبطة بتفاصيل HTTP غير ذات صلة.

كل ما علينا فعله الآن هو تنفيذ `UserService` لدينا باستخدام أي قاعدة بيانات نريد.

```go
type MongoUserService struct {
}

func NewMongoUserService() *MongoUserService {
	//todo: pass in DB URL as argument to this function
	//todo: connect to db, create a connection pool
	return &MongoUserService{}
}

func (m MongoUserService) Register(user User) (insertedID string, err error) {
	// use m.mongoConnection to perform queries
	panic("implement me")
}
```

يمكننا اختبار هذا على حدة، وبعد أن نطمئن، نجمع هاتين الوحدتين معًا في `main` لنحصل على تطبيقنا العامل.

```go
func main() {
	mongoService := NewMongoUserService()
	server := NewUserServer(mongoService)
	http.ListenAndServe(":8000", http.HandlerFunc(server.RegisterUser))
}
```

### تصميم أكثر متانة وقابلية للتوسع بجهد قليل

هذه المبادئ لا تسهّل حياتنا على المدى القصير فحسب، بل تجعل توسيع النظام في المستقبل أسهل.

لن يكون مستغرَبًا أن نرغب، في تكرارات لاحقة لهذا النظام، في إرسال بريد إلكتروني إلى المستخدم يؤكد تسجيله.

في التصميم القديم كنا سنضطر إلى تغيير المعالج _و_ الاختبارات المحيطة به. وهكذا غالبًا تصبح أجزاء من الكود غير قابلة للصيانة، إذ تتسلل وظائف أكثر وأكثر لأنها _مصممة_ أصلًا بهذه الطريقة؛ بحيث يتولى "معالج HTTP"... كل شيء!

وبفصل الاهتمامات باستخدام واجهة (interface)، لا نضطر إلى تعديل المعالج _إطلاقًا_ لأنه غير معني بمنطق العمل المتعلق بالتسجيل.

## الخلاصة

اختبار معالجات HTTP في Go ليس بالأمر الصعب، أما تصميم برمجيات جيدة فقد يكون تحديًا!

يقع الناس في خطأ اعتبار معالجات HTTP حالة خاصة، فيتخلون عن ممارسات هندسة البرمجيات الجيدة عند كتابتها، ما يجعل اختبارها صعبًا بعد ذلك.

وللتأكيد مرة أخرى: **معالجات http في Go هي مجرد دوال**. وإذا كتبتها كما تكتب أي دالة أخرى، بمسؤوليات واضحة وفصل جيد للاهتمامات، فلن تجد أي مشكلة في اختبارها، وستصبح قاعدة كودك أكثر صحة بفضل ذلك.
