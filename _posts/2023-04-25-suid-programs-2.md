---
layout: post
published: true
title: SUID Programs Part 2
date: '2023-04-25'
tags:
  - Linux
  - Privilege-Escalation
  - Arabic-Article
---

  <div dir="rtl" markdown="1">

السلام عليكم ورحمة الله وبركاته
 
المقالة هذه إستكمال [للمقالة السابقة](https://0xb1tbyte.github.io/2022-12-31-suid-programs-1/) حول موضوع الـ Set-UID Program 

سنناقش النقاط الآتية في هذه المقالة : 
- حالات إستغلال دالة `system()` في الـ Set-UID Program  
  - كيف تبدو قيم الـ `uid` و الـ `euid` في البرامج العادية وفي برامج الـ Set-UID ؟
  - كيف تعمل دالة `setuid()` في برامج الـ Set-UID ؟
  - كيف تعمل دالة `system()` ؟ هل للدالة حالات خاصة ضمن برامج الـ Set-UID ؟
  
لنبدأ بسم الله 
  
# حالات إستغلال دالة `system()` في الـ Set-UID Program
ذكرنا في المقالة السابقة الكود التالي   
  
  </div>

```c
  #include <stdio.h>
  #include <stdlib.h>
  #include <unistd.h>
  int main () 
  {
        setuid (0);
        system("ls");
  }
```

 <div dir="rtl" markdown="1">
وشرحنا مثال على طريقة إستغلاله في حال تم تفعيل الـ Set-UID bit للبرنامج، لكننا لم نفصّل في الكود نفسه ولم نناقش الحالات التي لا يمكننا فيها إستغلال دالة `system()` في الـ Set-UID Program 

نلاحظ في الكود أعلاه أنه قبل نداء دالة `system()` يوجد نداء لدالة `setuid()` ونلاحظ أننا قمنا بتمرير القيمة `صفر` للدالة

قبل أن نشرح عمل الدالة لنقوم بتعطيلها في الكود ونحاول إستغلال الكود كما فعلنا في المقالة السابقة

سيصبح الكود كالآتي : 

 </div>
 

```c
  #include <stdio.h>
  #include <stdlib.h>
  #include <unistd.h>
  int main () 
  {
        //setuid (0);
        system("ls");
  }
```
 



 <div dir="rtl" markdown="1"> 
 
نلاحظ أنه لم يعد بإمكاننا إستغلال دالة `system()`بدون نداء الدالة `setuid()` 
 
 ![1](https://raw.githubusercontent.com/0xb1tByte/0xb1tbyte.github.io/master/assets/media/suid-2/1.png#center)
 
وفي حال أعدنا الكود لحالته الأولى سنستطيع إستغلال وجود دالة `system()` في الـ Set-UID Program 
 
 ![2](https://raw.githubusercontent.com/0xb1tByte/0xb1tbyte.github.io/master/assets/media/suid-2/2.png#center)
 
يبدو أن وجود الدالة `setuid()` عامل مهم لإستغلال الـ `system()` ضمن الـ Set-UID Program 

لنعود لتعريف الدالة `setuid()` 
 </div>
 
> `setuid()` sets the effective user ID of the calling process.
> 

 

 <div dir="rtl" markdown="1"> 
 
إذًا الدالة تقوم بعمل set لقيمة الـ Effective User ID للعملية
 
لكننا قمنا بالأساس بتفعيل الـ Set-UID bit في البرنامج ويفترض أن يعمل البرنامج وفق صلاحيات الـ owner ( في حالتنا هو الـ root ) 

فلماذا يجب علينا تكرار هذه العملية مرة أخرى عن طريق نداء الدالة `setuid()` في كود الـ Set-UID Program ؟ 

في الحقيقة المثال أعلاه والسؤال هذا نتج خلال رحلة تعلمي للـ Set-UID Program 

في البداية كتبت الكود بدون نداء دالة `setuid()` لأنه حسب فهمي البسيط برامج الـ Set-UID ستعمل ضمن صلاحيات الـ owner فلا حاجة لإعادة تعريف قيمة الـ Effective User ID بشكل صريح ضمن الكود 

حتى وصلت للإستغلال ولم أتمكن من إستغلال الـ `system()` إلا بوجود دالة `setuid()` في الكود 

فلمعرفة سبب وجود دالة `setuid()` على الرغم من أن الـ Set-UID bit مُفعّلة في البرنامج قمت بالبحث وفهم هذه النقاط : 
- كيف تبدو قيم الـ `uid` و الـ `euid` في البرامج العادية وفي برامج الـ Set-UID ؟ 
- كيف تعمل دالة `setuid()` في برامج الـ Set-UID ؟
- كيف تعمل دالة `system()` ؟ هل للدالة حالات خاصة ضمن برامج الـ Set-UID ؟ 
 
لنبدأ الاجابة على هذه الأسئلة

## كيف تبدو قيم الـ `uid` و الـ `euid` في البرامج العادية وفي برامج الـ Set-UID ؟
نستطيع إستخدام الكود التالي للوصول لقيم الـ `uid` و `euid`

</div>


```c
#include <stdio.h>
#include <unistd.h>
  int main () 
  {
  int real = getuid();
  int euid = geteuid();
  printf("The REAL UID =: %d\n", real);
  printf("The EFFECTIVE UID =: %d\n", euid);
  }
```
<div dir="rtl" markdown="1"> 
لنرى قيم  الـ  `uid` و `euid` في البرنامج العادي بدون عمل set للـ Set-UID bit 

 ![3](https://raw.githubusercontent.com/0xb1tByte/0xb1tbyte.github.io/master/assets/media/suid-2/3.png#center)

نلاحظ كلا المتغيرين يحملان نفس القيمة `1000` والتي تعود للمستخدم الحالي `kali` 

لنرى الآن القيم في حالة الـ Set-UID Program 

![4](https://raw.githubusercontent.com/0xb1tByte/0xb1tbyte.github.io/master/assets/media/suid-2/4.png#center)

نلاحظ في حالة الـ Set-UID Program اختلفت القيم ، فقيمة الـ `euid` أصبحت `0` وهذا بديهي لأن قمنا بتفعيل الـ Set-UID bit في البرنامج 

إذًا نستطيع القول : 
- **في البرامج العادية** : ستكون كلا القيمتين `uid` و `euid` متساوية ولا إختلاف بينهما 
- **في الـ Set-UID Program** : ستكون قيمة الـ `uid` قيمة المستخدم الذي أنشأ/بدأ العملية ، بينما قيمة الـ `euid` سترجع لقيمة المستخدم الذي يملك البرنامج ( owner )

لنقوم الآن بالتعديل على الكود وإضافة دالة `system()` ونختبر الكود في كِلا الحالتين ، سيكون الكود كالآتي : 
</div>

```c
  #include <stdio.h>
  #include <unistd.h>
  #include <stdlib.h>
  int main () 
  {
    int real = getuid();
    int euid = geteuid();
    printf("The REAL UID =: %d\n", real);
    printf("The EFFECTIVE UID =: %d\n", euid);
    system("whoami");
  }
```
<div dir="rtl" markdown="1"> 
في حالة البرنامج العادي نرى الآتي : 

![5](https://raw.githubusercontent.com/0xb1tByte/0xb1tbyte.github.io/master/assets/media/suid-2/5.png#center)

النتائج أعلاه تبدو منطقية حتى الآن ، لنرى النتائج في حالة الـ Set-UID Prgoram 

![6](https://raw.githubusercontent.com/0xb1tByte/0xb1tbyte.github.io/master/assets/media/suid-2/6.png#center)

نلاحظ من الصورة أعلاه أن قيمة الـ `euid` تساوي `0` ( البرنامج يفترض أن يعمل بصلاحية الـ root ) لكن الأمر `whoami` الذي قمنا بتمريره للدالة `system()` يخبرنا أن المستخدم للعملية هو `kali`

وكما رأينا سابقًا، لن نستطيع إستغلال دالة `system()` ضمن الـ Set-UID Program إلا في حال وجود دالة `setuid()` 

لنقوم الآن بتعديل الكود مرة أخرى وإضافة دالة `setuid()` ونختبر قيم الـ `uid` و `euid` في كِلا الحالتين 

</div>

 <div dir="rtl" markdown="1"> 

## كيف تعمل دالة `setuid()` في برامج الـ Set-UID ؟

سنقوم بإختبار قيم الـ `uid` و الـ `euid` قبل وبعد نداء دالة `setuid()` وسيكون الكود كالآتي : 

</div> 

```c
  #include <stdio.h>
  #include <unistd.h>
  #include <stdlib.h>
  int main () 
  {
  int real = getuid();
  int euid = geteuid();
  printf("Before calling setuid() \n");  
  printf("The REAL UID =: %d\n", real);
  printf("The EFFECTIVE UID =: %d\n", euid);
  system("whoami");
  setuid (0);
  int real2 = getuid();
  int euid2 = geteuid();
  printf("After calling setuid() \n");
  printf("The REAL UID =: %d\n", real2);
  printf("The EFFECTIVE UID =: %d\n", euid2);
  system("whoami");
  }
```

<div dir="rtl" markdown="1">  

في حالة البرنامج العادي بالتأكيد نداء الدالة `setuid()`  وتمرير القيمة `0` لها لن ينجح لأن البرنامج لا يملك الصلاحية في الأساس ، لكن لمن أراد معرفة القيم ستكون النتائج كالآتي : 

![7](https://raw.githubusercontent.com/0xb1tByte/0xb1tbyte.github.io/master/assets/media/suid-2/7.png#center)

وفي حالة الـ Set-UID Program ستكون النتائج كالآتي : 

![8](https://raw.githubusercontent.com/0xb1tByte/0xb1tbyte.github.io/master/assets/media/suid-2/8.png#center)
 
 الصورة أعلاه تخبرنا بالكثير ، لنبدأ بتفصيل وفهم النتائج : 
 - **قبل نداء `setuid()`** : نلاحظ أن قيمة الـ `euid` فقط تغيّرت ( لأن الـ Set-UID Program تؤثر فقط في قيمة الـ `euid` )
 - **بعد نداء `setuid()`** : نلاحظ أن كلا المتغيرين `uid` و `euid` تغيرت قيمهم وأصبحت `0`

لكن من تعريف الدالة `setuid()` الذي مررنا عليه سابقًا عرفنا أن الدالة تغيّر قيمة الـ `euid` فلماذا في الكود أعلاه بعد نداء الدالة تغيرت قيمة الـ `uid` أيضًا ؟ 

لو عدنا للصفحة الخاصة بالدالة وأكملنا القراءة سنجد الآتي : 

</div> 

> If the calling process is privileged (more precisely: if the process has the CAP_SETUID capability in its user namespace), the real UID and saved set-user-ID are also set.
> 


 <div dir="rtl" markdown="1"> 

تخبرنا العبارة أعلاه أن دالة `setuid()` ستقوم بتغيير قيم الـ `uid` والـ `euid` معًا في حال كانت العملية تعمل بصلاحية عالية ، وهذا يفسّر النتائج السابقة التي حصلنا عليها

إذًا في حال إستخدام دالة `setuid()` وكان البرنامج Set-UID Program فسيتم تغيير كِلا القيمتين `uid` و `euid`

وإذا عدنا لسؤالنا الأول : سبب نداء دالة `setuid()` قبل `system()` -حتى نستطيع إتمام الإستغلال- سنلاحظ أنه تمكننا من تنفيذ الإستغلال في حالة كانت جميع قيم الـ IDs ( `uid` و `euid` ) تساوي الـ ID الخاص بالمستخدم `root`

لكن كما عرفنا في المقالة السابقة الـ access control للعملية مرتبط بقيمة الـ `euid` فقط فلماذا في حالة الـ `system()` إحتجنا أن تكون قيمة الـ `uid` مساوية لـ ID المستخدم `root` كذلك ؟

لنخوض أكثر في دالة `system()`


## كيف تعمل دالة `system()` ؟ هل للدالة حالات خاصة ضمن برامج الـ Set-UID ؟
كما نعرف دالة `system()` ستقوم بتشغيل `/bin/sh` ومن ثم تمرير الـ command ( أطلع على [هذه المقالة](https://0xb1tbyte.github.io/2019-12-16-Injection-Attacks-on-System/) للتفاصيل ) 

 </div>
 
 ```bash
execl("/bin/sh", "sh", "-c", command, (char *) NULL);
```

 <div dir="rtl" markdown="1"> 
 ولو أكملنا القراءة في مرجع الدالة سنجد العبارة الآتية : 

 </div>
 
 > system() will not, in fact, work properly from programs with set-user-ID or set-group-ID privileges on systems on which/bin/sh is bash version 2: as a security measure, bash 2 drops privileges on startup.
> 

 <div dir="rtl" markdown="1"> 
 
 تخبرنا العبارة أن دالة `system()` لن تعمل كما يفترض منها ضمن برامج الـ Set-UID ، السبب في ذلك أن الـ bash ستقوم بتعطيل الصلاحيات
 
 ولو بحثنا في المرجع الخاص بالـ `/bin/sh` سنجد التالي : 
 </div>
 
 ```
 -i               Specify that the shell is interactive; see below. An
                 implementation may treat specifying the -i option as an
                 error if the real user ID of the calling process does
                 not equal the effective user ID or if the real group ID
                 does not equal the effective group ID.
 ```


```
 ENV             shall be ignored if the real and effective user IDs or real and
                 effective group IDs of the process are different.
 ```

 
 <div dir="rtl" markdown="1"> 
يتضح لنا من المرجع أعلاه أنه في حال إختلاف قيم الـ `uid` والـ `euid` سيقوم الـ `/bin/sh` بتعطيل الصلاحية 
 
أعتقد الآن بعد التفصيل الطويل جدًا وجدنا الإجابة النهائية 😄
 
**لنلخص جميع ما سبق:** 
 
لو أردنا إستغلال دالة `system()` ضمن الـ Set-UID Program نحن بحاجة الى نداء دالة `setuid()` مسبقًا حتى تكون قيمة الـ `uid` و الـ `euid` متساوية بالتالي لن يقوم الـ `/bin/sh` بتعطيل الصلاحيات ، ودالة `setuid()` لن تقوم بالتعديل على قيمة الـ `uid` إلا في حال كان البرنامج ذو صلاحية عالية


ختامًا، أود الإشارة بأن هذه المقالة تمّت كتابتها خلال دراسة هذه المواضيع، فكل ما تم ذكره هنا قد يحتمل الخطأ، لكن بالإمكان العودة إلى المراجع التي إستندت عليها هذه المقالة

وإن أصبت فمن توفيق الله وحده، وإن أخطأت فمن نفسي والشيطان

# المراجع 
- [setuid()](https://man7.org/linux/man-pages/man2/setuid.2.html)
- [system()](https://linux.die.net/man/3/system)
- [/bin/sh](https://man7.org/linux/man-pages/man1/sh.1p.html)

</div>
 
