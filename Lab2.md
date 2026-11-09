## Комп'ютерні системи імітаційного моделювання
## СПм-25-2, Топорець Дмитро Олексійович
### Лабораторна робота №2. Редагування імітаційних моделей у середовищі NetLogo

### Варіант 2, модель у середовищі NetLogo:
[Climate Change](http://www.netlogoweb.org/launch#http://www.netlogoweb.org/assets/modelslib/Sample%20Models/Earth%20Science/Climate%20Change.nlogo)

### Внесені зміни у вихідну логіку моделі, за варіантом:

**Додано можливість встановлення кількості хмар**, для спрощення процесу початкового налаштування моделі. Імовірність встановлюється користувачем через інтерфейс середовища моделювання.

Для цього було додано процедуру підрахунку хмар
<pre>
to-report cloud-cluster-amount
  report length (remove-duplicates [cloud-id] of clouds)
end
</pre>
відповідну змінну
<pre>
to setup
  ...
  set cloud-amount 0
  reset-ticks
end
</pre>
та процедуру для застосування введеного значення
<pre>
to apply-cloud-amount
  let target max (list 0 (round cloud-amount))
  let current cloud-cluster-amount

  if target > current [
    repeat (target - current) [
      add-cloud
    ]
  ]

  if target < current [
    repeat (current - target) [
      remove-cloud
    ]
  ]

  set cloud-count cloud-cluster-count
end
</pre>
Для узгодження з роботою вже наявного функціоналу, а саме кнопок "add cloud" та "remove cloud", було оновлено відповідні процедури:
<pre>
to add-cloud
  ...
  set cloud-amount cloud-cluster-amount
end
</pre>
<pre>
to remove-cloud
  ...
  set cloud-amount cloud-cluster-amount
end
</pre>

**Додано зміну кута падіння нових сонячних променів з часом.**

Для реалізації логіки врахування зміни умов протягом доби, було прийнято, що наявна в симуляції поверхня планети є "візуально пласкою частиною". Відповідно до такого масштабу, сонячне світло має розповсюджуватись атмосферою в момент часу приблизно паралельно. Щоб уникнути переускладнення моделі, воно приймається повністю паралельним. Для цього було замінено наявну процедуру "create-sunshine" на
<pre>
to create-sunshine
  let day-length 8640 ;; hours * minutes * seconds / 10
  let phase (ticks mod day-length) / day-length

  ;; daylight: 25% - 75% of the daily cycle
  if phase >= 0.25 and phase <= 0.75 [

    ;; progress of the sun from morning to evening
    let day-progress (phase - 0.25) / 0.5

    ;; change the direction of sunlight
    let solar-heading 135 + 90 * day-progress

    ;; move the emission area from left to right
    let source-x min-pxcor + 2 +
      day-progress * (max-pxcor - min-pxcor - 4)
    
    ;; create new rays depending on sun brightness
    if 10 * sun-brightness > random 50
      create-rays 1 [
        set heading solar-heading
        set color yellow
        ;; (source-x + random-float 2 - 1) -> setxy random-xcor
        setxy setxy random-xcor max-pycor
      ]
    ]
  ]
end 
</pre>
Зазначене врахування масштабу також дозволяє вважати **не помилкою** рівномірність albedo на всій ділянці поверхні, оскільки масштаб, в якому розглядається тільки один біом (ліс, пустеля, тощо), має приблизно однаковий показник albedo.


**Додано можливість встановлення кількості CO2**,

<pre>
to setup
  ...
  set cloud-amount 0
  set CO2-amount 0
  reset-ticks
end
</pre>

<pre>
to apply-co2-amount
  let target max (list 0 (round CO2-amount))
  let current count CO2s

  let difference abs (target - current)

  if target > current [
    let sky-height sky-top - earth-top

    create-CO2s difference [
      set color green
      setxy random-xcor
            earth-top + random-float sky-height
    ]
  ]

  if target < current [
    repeat difference [
      ask one-of CO2s [
        die
      ]
    ]
  ]

  set CO2-amount count CO2s
end
</pre>


<pre>
to add-CO2
  ...
  set CO2-amount count CO2s
end

to remove-CO2
  ...
  set CO2-amount count CO2s
end
</pre>


### Внесені зміни у вихідну логіку моделі, на власний розсуд:

**Додано можливість зміни руху хмар на конкретній висоті**

Для наближення умов моделі до реальних, було додано можливість руху хмар не тільки "наліво" а й "направо":
<pre>
to add-cloud            ;; erase clouds and then create new ones, plus one
  ...
  ;; move left or right
  let cloud-heading one-of [90 270]
  ...
  create-clouds 3 + random 20
  [
    ...
    ;; randomly pick direction
    set heading cloud-heading
    
  ]
  set cloud-amount cloud-cluster-amount
end
</pre>
В даній реалізації винесення вибору напрямку за межі власне створення хмари, аналогічно до визначення її швидкості, є необхідним, оскільки хмари фактично складаються з декількох "turtle" агентів.

**Додано можливість врахування довжини світлового дня**

В якості значень за замовчуванням береться шоста ранку та шоста вечора, що дає 12-годинний світловий день. Ці змінні задаються елементами "слайдер", тому явно визначати їх в коді не потрібно. Також в даному випадку надається можливість встановлення часу сходу та заходу сонця асиметрично до полуночі/полудня. Це потрібно для проведення симуляції згідно часу "часового поясу", замість "сонячного" часу.
<pre>
to create-sunshine
  let ticks-per-day 8640
  let ticks-per-hour 360

  ;; Calculate current time of day
  let current-hour
    ((ticks mod ticks-per-day) / ticks-per-hour)

  ;; Check if it is daytime
  if current-hour > time-sunrise and
     current-hour < time-sunset [

    ;; Progress from sunrise to sunset (0 - 1)
    let day-progress
      (current-hour - time-sunrise) /
      (time-sunset - time-sunrise)

    ;; Calculate solar direction
    let solar-heading (90 + 180 * day-progress)
    
    ;; sun factor. Used to include angle
    ;;let sun-factor cos (solar-heading - 180)
    ;; if 10 * sun-brightness * sun-factor > random 50 [

    ;; Create a new ray depending on sun brightness
    if 10 * sun-brightness > random 50 [
      create-rays 1 [
        set heading solar-heading
        set color yellow
        setxy random-xcor max-pycor
      ]
    ]
  ]
end
</pre>
В даній реалізації під час перевірки наявності світлового дня були використані суворі нерівності. Це потрібно для того щоб уникнути створення сонячних променів, які рухаються горизонтально.

<img width="720" height="313" alt="image" src="https://github.com/user-attachments/assets/75405829-f634-4d15-8394-aa0aa5ff2cde" />

**Відкинуті ідеї**
1) Часові масштаби моделі не відтворюють безпосередньо фізичну тривалість перебігу процесів. Зокрема, швидкість переміщення агентів випромінювання має умовний характер і використовується для відображення їх просторового поширення та взаємодії з іншими. Фактичний час проходження світла через атмосферу є дуже незначним порівняно з тривалістю добового циклу та атмосферних процесів, які симулюються. Водночас спроба узгодити ці часові масштаби вимагала б не лише зміни швидкості переміщення агентів, а й уточнення просторового масштабу моделі, фізичного значення окремих одиниць енергії, їх кількісного співвідношення з енергетичними потоками та правил передавання, поглинання і перетворення енергії.
2) Для уникнення надмірного ускладнення моделі, рух хмар реалізовано незалежно від їх висоти, типу, часу доби та інших метеорологічних умов та явищ. Детальніше врахування атмосферних умов потребувало б введення додаткових параметрів і залежностей: впливу висоти на напрямок та швидкість руху, розподілу ймовірностей появи різних типів хмар, залежності їх утворення від інших погодних явищ, а також правил взаємодії між ними. Також не всі явища можливо перенести на двовимірну модель без використання надмірно складних розрахунків.
3) Можливість задавати незалежний час сходу та заходу Сонця потенційно дозволяє описати випадки, коли їхні часові відхилення відносно середини сонячного дня не є симетричними. Зокрема, така особливість може бути пов'язана не лише з умовами освітлення горизонтальної поверхні, а й з її просторовою орієнтацією, наприклад для моделювання похилої ділянки льодовика або схилу гори. Проте коректне відтворення таких умов потребувало б додаткового дослідження впливу нахилу та орієнтації поверхні на тривалість її освітлення.
4) На поточному етапі зазначені особливості недостатньо досліджені для їх формалізації та реалізації. Відповідно, цей аспект не входить до поточної області дослідження та не підлягає розгляду в межах реалізованої моделі.

Фінальний код моделі та її інтерфейс доступні за [посиланням](https://drive.google.com/file/d/1NDqYxZbG4A60mjdARIr5BbAT_8DlOZ1Z/view?usp=drive_link).
