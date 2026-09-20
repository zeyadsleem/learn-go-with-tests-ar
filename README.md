# تعلّم Go بالاختبارات

<p align="center">
  <img src="https://raw.githubusercontent.com/quii/learn-go-with-tests/main/red-green-blue-gophers-smaller.png" />
</p>

الترجمة العربية لكتاب [Learn Go with Tests](https://github.com/quii/learn-go-with-tests)
للمؤلف [Chris James](https://github.com/quii).

**[اقرأ الكتاب من هنا](https://zeyadsleem.github.io/learn-go-with-tests-ar/)**

## عن الكتاب

يعلّم الكتاب لغة Go من خلال كتابة الاختبارات، مع تركيز على التطوير الموجه
بالاختبار (TDD). يبدأ كل فصل باختبار فاشل ونسير خطوة بخطوة حتى يعمل الكود،
ثم نعيد ترتيبه (refactor) بثقة، لأن الاختبارات تحمينا.

## حالة الترجمة

الترجمة عمل مستمر. الفصول المتاحة حاليًا:

- [تثبيت Go وتجهيز بيئة عمل منتجة](content/docs/install-go.md)
- [مرحبًا بالعالم](content/docs/hello-world.md)
- [الأعداد الصحيحة](content/docs/integers.md)
- [التكرار](content/docs/iteration.md)
- [المصفوفات والشرائح](content/docs/arrays-and-slices.md)
- [Structs والـ Methods والـ Interfaces](content/docs/structs-methods-and-interfaces.md)

ولمتابعة الجديد، ضع نجمة على المستودع أو تابع [الموقع](https://zeyadsleem.github.io/learn-go-with-tests-ar/).

## المساهمة

المساهمات مرحّب بها، سواء في تصحيح ترجمة موجودة أو في ترجمة فصل جديد.
افتح Issue أو أرسل Pull Request مباشرة.

## البناء محليًا

المشروع مبني بـ [Hugo](https://gohugo.io) وثيم [Hugo Book](https://github.com/alex-shpak/hugo-book).

```sh
git clone --recursive https://github.com/zeyadsleem/learn-go-with-tests-ar.git
cd learn-go-with-tests-ar
hugo server
```

سيفتح الموقع على `http://localhost:1313`.

## الترخيص والإسناد

كل المحتوى مشتق من كتاب [Learn Go with Tests](https://github.com/quii/learn-go-with-tests)
للمؤلف Chris James، والكود الأصلي والشرح منشوران تحت [ترخيص MIT](LICENSE.md).
وتشير روابط الكود في الفصول إلى المستودع الأصلي.

الترجمة العربية منشورة بالترخيص نفسه، والحقوق محفوظة لمترجميها.
