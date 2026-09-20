# تعلّم Go بالاختبارات

<p align="center">
  <img src="https://raw.githubusercontent.com/quii/learn-go-with-tests/main/red-green-blue-gophers-smaller.png" />
</p>

الترجمة العربية لكتاب [Learn Go with Tests](https://github.com/quii/learn-go-with-tests)
للمؤلف [Chris James](https://github.com/quii).

**[اقرأ الكتاب من هنا](https://zeyadsleem.github.io/learn-go-with-tests-ar/)**

## عن الكتاب

الكتاب بيعلّمك لغة Go من خلال كتابة الاختبارات، مع تركيز على تطوير البرامج
بالاختبارات (TDD). كل فصل يبدأ من اختبار فاشل وبنمشي خطوة بخطوة لحد ما الكود يشتغل،
وبعدها بنعيد ترتيبه (refactor) بثقة لأن الاختبارات بتحمينا.

## حالة الترجمة

الترجمة شغل مستمر. الفصول المتاحة حاليًا:

- [تثبيت Go وتجهيز بيئة العمل](content/docs/install-go.md)
- [مرحبًا بالعالم](content/docs/hello-world.md)
- [الأعداد الصحيحة](content/docs/integers.md)
- [التكرار](content/docs/iteration.md)
- [المصفوفات والشرائح](content/docs/arrays-and-slices.md)
- [Structs والـ Methods والـ Interfaces](content/docs/structs-methods-and-interfaces.md)

لو عايز تتابع الجديد، اعمل نجمة للريبو أو تابع [الموقع](https://zeyadsleem.github.io/learn-go-with-tests-ar/).

## المساهمة

المساهمات مرحّب بها، سواء في تصحيح ترجمة موجودة أو ترجمة فصل جديد.
افتح Issue أو ابعت Pull Request مباشرة.

## البناء محليًا

المشروع مبني بـ [Hugo](https://gohugo.io) وثيم [Hugo Book](https://github.com/alex-shpak/hugo-book).

```sh
git clone --recursive https://github.com/zeyadsleem/learn-go-with-tests-ar.git
cd learn-go-with-tests-ar
hugo server
```

الموقع هيفتح على `http://localhost:1313`.

## الترخيص والإسناد

كل المحتوى مشتق من كتاب [Learn Go with Tests](https://github.com/quii/learn-go-with-tests)
للمؤلف Chris James، والكود الأصلي والشرح منشورين تحت [ترخيص MIT](LICENSE.md).
روابط الكود في الفصول بتشاور على المستودع الأصلي.

الترجمة العربية منشورة بنفس الترخيص، والحقوق محفوظة لمترجميها.
