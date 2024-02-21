---
layout: post
published: true
title: The Stack and Windows x86 Calling Conventions Part 3
date: '2024-01-25'
tags:
- Windows
- Windows Internals
- Arabic-Article
- System Security
---

<div dir="rtl" markdown="1">

السلام عليكم ورحمة الله وبركاته 

هذه المقالة استكمال للسلسلة السابقة ، وستكون الأخيرة ( إلا إن اكتشفنا موضوع مُقارب للمواضيع هذه ورهيب 😁 ) 

المقالات السابقة كانت تمهيد -ومقدمة لابد منها- قبل الحديث عن مواضيع المقالة الأخيرة، والنقاط التي سنناقشها في مقالتنا هذه هي : 
* C Declaration (`__cdecl`) Calling Convention
* Standard Call (`__stdcall`) Calling Convention
* Fast Call (`__fastcall`) Calling Convention

المراجع في هذه المقالة بشكل أساسي هي الكتاب الرائع `Practical Malware Analysis` والمرجع الرسمي من مايكروسوفت ، كوننا نتحدث هنا عن الـ Calling Convention في أنظمة مايكروسوفت 

لنبدأ بسم الله ،

# ملاحظة

في كتاب `Practical Malware Analysis` قبل الحديث عن تفاصيل الـ Calling Conventions نجد هذه الملاحظة المهمة

</div> 

> NOTE Although the same conventions can be implemented differently between compilers, we’ll focus on the most common ways they are used.



<div dir="rtl" markdown="1">
بايجاز، نستطيع الفهم من الملاحظة السابقة أن الـ Calling Conventions قد يختلف تنفيذها ( implementation ) من Compiler لآخر ، لذلك مثل ما ذكر الكتاب ، سنركّز في المقالة هذه على المفاهيم العامة لكل Calling Convention 


أيضًا هذا هو الـ Pseudocode الذي سنعتمد عليه في شرح الجزئيات القادمة 

</div> 


```c
 1 | int test(int x, int y, int z);
 2 | int a, b, c, ret;
 3 |
 4 | ret = test(a, b, c);
```

<div dir="rtl" markdown="1">

# C Declaration (`__cdecl`) Calling Convention

نجد في المرجع الخاص [بمايكروسوفت](https://learn.microsoft.com/en-us/cpp/cpp/cdecl?view=msvc-170) أن هذا الـ Calling Convention هو الـ default في لغات C & C++ 


</div> 

> __cdecl is the default calling convention for C and C++ programs.

<div dir="rtl" markdown="1">

طيب ماهي القوانين في هذا الـ Calling Convention ؟ 

إذا تم استخدام الـ `__cdecl` فسيتم الآتي : 

١ - سيتم تمرير الـ `Parameters` الى الـ `Stack` **من اليمين إلى اليسار**

٢ - الدالة المُنادية ( `Caller` ) ستكون هي المسؤولة عن عملية إعادة تهيئة الـ Stack لوضعه الأولي بعد أن تُنهي الدالة المُناداة ( `Callee` ) عملها 

لنعود للـ Pseudocode السابق ونربط النقاط التي ذكرناها أعلاه بالكود 

**النقطة رقم ١**

في السطر `4` نجد نداء الدالة `test` ، ونلاحظ أنه تم تمرير الـ `Parameters` التالية : `a` , `b` , `c`

في حالة الـ `__cdecl` سيتم أولًا تخزين الـ `c` في الـ `Stack` ومن ثم الـ `b` وآخرًا الـ `a`

**النقطة رقم ٢** 

الدالة التي ستقوم بنداء الدالة `test` ستكون هي المسؤولة عن تنظيف الـ `Stack`  وإعادة الـ `registers` لوضعها بعد أن تُنهي الدالة `test` عملها 

كود الأسمبلي الآتي لعله يوضّح آلية الـ `__cdecl`


</div>

```nasm
 1 | push c
 2 | push b
 3 | push a
 4 | call test
 5 | add esp, 12
 6 | mov ret, eax
```



<div dir="rtl" markdown="1">

لاحظ في السطر `1` تم عمل `push` للـ `c` ، والـ `c` هو آخر `parameter` تم تمريره وقت نداء الدالة `test` 
</div>

```c
 4 | ret = test(a, b, c);
```

<div dir="rtl" markdown="1">

وفي السطر `2` تم تمرير قيمة الـ `b` للـ `Stack` وبعده الـ `a` 

وفي السطر رقم `4` تم نداء الدالة `test` بالتعليمة `CALL` والتي ناقشناها في مقالة سابقة 

عند هذه الخطوة سينتقل التحكم للدالة `test` وسيتم انشاء `Stack Frame` جديد خاص بها 

وفي حال انتهت الدالة `test` من عملها سيتم إعادة التحكم للدالة المُنادية ( `Caller` ) 

وستتكفل الدالة المُنادية ( `Caller` ) بإعادة الـ `Stack` لحالته الأولى ، هذه الخطوات يتم تنغيذها في السطر رقم `5`

بتفصيل أكثر : الكود في السطر رقم `5` يقوم باضافة `12` الى الـ `ESP` و كما عرفنا في مقالة سابقة، الـ `Stack` ينمو **بشكل عكسي** ، فعملية الإضافة هنا لأنه نريد إعادة الـ `Stack`  للحالة الأولى

وبالنسبة للقيمة التي قمنا بإضافتها ( `12` ) لأنه تم تمرير `3` متغيرات ( `Parameters` ) وكل متغيّر يأخذ مساحة `4` بايت ( نتحدّث هنا عن الـ 32bit systems )

# Standard Call (`__stdcall`) Calling Convention

الـ `__stdcall` مثل الـ `__cdecl` لكن يختلف عنه في أن الدالة المُناداة ( `Callee` ) هي المسؤولة عن تنظيف وإعادة تهيئة الـ `Stack` 

ولو عدنا لمثالنا السابق ستكون الدالة `test` هي المسؤولة عن تنظيف الـ `Stack` ، وعلى وجه التحديد، يتم تنفيذ هذه العملية في الـ `Epilogue` الخاص بالدالة `test` 

كذلك الـ `__stdcall` هو الـ Calling Convention الذي يتم استخدامه في دوال الـ `Win32 API` ، وبلغة أخرى، أي دوال تقوم بنداء دوال الـ `Win32 API` لن تكون مسؤولة عن إعادة تهيئة الـ `Stack` ، لأنها مسؤولية دوال الـ `Win32 API`

# Fast Call (`__fastcall`) Calling Convention

في الـ `__fastcall` سيتم الآتي : 

١ - الـ Parameters الأولى ( عادةً المتغيرين الأوليين ) سيتم تخزينهم في registers ( عادةً في الـ `ECX` و `EDX` ) وبقية الـ Parameters يتم تمريرها للـ `Stack` **من اليمين إلى اليسار**

٢ - عملية إعادة تهيئة الـ `Stack` من مسؤولية الدالة المُناداة ( `Callee` ) 

الجدول التالي يقارن بين الأنواع الثلاث 

| Calling Convention | Stack Cleanup | Parameter Passing |
|-------------------|---------------|-------------------|
| `__cdecl`         | الدالة المُنادية ( `Caller` ) | يتم تمريرها للـ `Stack` **من اليمين إلى اليسار** |
| `__stdcall`       | الدالة المُناداة (`Callee` ) | يتم تمريرها للـ `Stack` **من اليمين إلى اليسار** |
| `__fastcall`      | الدالة المُناداة (`Callee` ) | يتم تخزين أول متغييرن في registers والبقية يتم تمريرهم للـ `Stack` |


</div>


> ℹ️ [ملاحظة]
> هذه المقالة تمّت كتابتها خلال دراسة هذه المواضيع، فكل ما تم ذكره هنا قد يحتمل الخطأ، لكن بالإمكان العودة إلى المراجع التي إستندت عليها هذه المقالة 
