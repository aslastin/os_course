# Процессы

## Процесс - это ...

_Процесс_ - абстракция ОС, соединяющая управление потоками исполнения, памятью и ресурсами. Программа исполняется в виде процесса и для нее логически ситуация выглядит так, что она выполняется монопольно процессором. Задача же ОС обеспечить исполнение процесса на практике. 

- С любым процессом ассоциировано адресное пространство с уникальной виртуальной памятью. Физическая память - более менее непрерывный массив данных, откуда можно читать/писать с помощью команд процессора. При этом концепция адресного пространства - это виртуализация данного массива, таким образом, чтобы каждому процессу казалось, что его память уникальна только для него (например, для каждого процесса адресация начинается с 0 адреса, при этом по факту во всей памяти 0 адрес всего один). Это делается с помощью оборудования __MMU__ (Manager Memory Unit), а логическая абстракция - несколько системных команд/регистров, обеспечивающих поддержку виртуальной памяти. Например, __x86__: регистр __CR3__ (Control Register 3), который содержит указатель на корень таблицы трансляции, обеспечивающей отображение логических адресов (то есть адресов, с которыми работает процесс) в физические. __Arm__: __TTBR0__, __TTBR1__.

- В процессе наличествует один или несколько потоков исполнения. Изначально UNIX подобные системы начинали с единственного потока исполнения. На текущей момент, почти все системы поддерживают многопоточное исполнение. Чаще всего многопоточное исполнение - несколько потоков исполнения, разделяющих одно и то же адресное пространство, вследствие чего способные взаимодействовать с одними и теми же данными.

- Исполнение происходит в непривилегированном контексте процессора, то есть в терминологии процессора __x86__ - __ring 3__, __arm__ - __User__. Модель исполнения __x86__ подразумевает 3 уровня привилегий (3 кольца защиты): __ring 0__ -  уровень привилегий, на котором процессор может исполнять все команды и исполнение любой команды приведет к ожидаемому результату; __ring 3__ - пользовательский уровень, исполнение некоторых команд, например, арифметического сложения 2-х регистров, приведет также к ожидаемому результату, но исполнение другого рода команд, например, запись в привилегированный регистр, тот же __CR3__, или доступ к некоторым видам памяти, приводит к _fault_ - прерывание исполнения и передача управления на ядро ОС, на обработчик для соответствующего fault. __Итог__: на пользовательских уровнях некоторые команды использовать нельзя. __ring 1__, __ring 2__ - режимы, которые поддерживаются в процессорах, но в ОС почти не используются - режимы, в которых можно чуть больше, но не все. У __arm__ есть несколько очень разных привилегированных режимов: один для обработки прерываний, другой - режим супервизора, третий режим для исполнения гипервизоров, то есть набором уровней привилегий еще более богатый, но процесс, вне зависимости от платформы, исполняется в непривилегированном режиме.  

- Стандартизированный интерфейс в ядро (__x86__: `syscall`/`sysenter`/`int`, __arm__: `svc`). Как мы уже поняли, процесс мало как может повлиять на ядро ОС, поэтому необходим какой-то интерфейс взаимодействия. Так в __x86__ - есть специальные инструкции для взаимодействия с ОС. В самых разных ядрах используются разные механизмы для реализации системных вызовов. 

- При этом процесс инкапсулирует в себя список ресурсов, с которыми работает. Например, в UNIX не только ядро, вдобавок реализация - это целая идеология. Символ веры этой идеологии - _«Everything is a file»_, то есть управление любым ресурсом происходит через файлы и с процессом ассоциирован свой набор файлов, которы уникален для этого процесса, и, если процесс завершается, то эти ресурсы каким-то способом освобождаются. В Windows - объектно-ориентированная модель, поэтому у большинства ресурсов есть __handle__. Семантически они не сильно отличаются: есть некий уникальный идентификатор, который имеет смысл только в контексте данного процесса.

- Для практических целей, чтобы люди и другие процессы могли общаться с каким-то процессом, есть концепция как __pid__ (process id), то есть целочисленное значение, которое, заметьте, не уникально идентифицирует процесс, а просто уникально идентифицирует в конкретный момент времени, то есть в некоторый момент времени такой же __pid__ получит другой процесс. __Handle__ - понятие популярное в Windows мире - идентификатор ресурса, с похожей семантикой как у файлового дескриптора.


## Адресное пространство

![2.1](https://github.com/aslastin/os_course/blob/master/images/2.1%20address%20space.png)

У процесса есть виртуальное адресное пространство. На рисунке изображен пример возможного layout виртуального адресного пространства и отображение на физическое адресное пространство. Здесь важно, что с точки зрения процесса существует набор областей, в которых находятся разные классы данных. В верху адресного пространства находится _стек_, растет вниз, где-то находится _сегмент данных_ - данные, адресуемые абсолютным образом. 

_Стек_ можно назвать ареной, в которой происходит аллокация короткоживущих локальных данных для той или иной функции, поэтому он не аллоцируется статически, а выделяется в зависимости от исполнения в процессе. 

_Сегмент данных_ - глобальные указатели, значения, обращение к котором возможно по конкретному адресу. 

_Текст_ - исполняемый код программы, из чего состоит скомпилированный код программы. 

Если рассмотреть реальный процессор и его реальную физическую память, там происходит адресация на линейном физическом устройстве (чем является современный массив данных), то в нем адресация происходит относительно какого-то нуля, страницы, и то, как проставлены стрелочки, то есть какой физической странице соответствует та или иная логическая страница в том или ином процессе - это одна из важных функций ОС, менеджера виртуальной памяти, который создает для разных адресных пространств такого рода таблицы трансляции и при переключении контекстов (то есть исполняется один процесс, затем другой, как только такое переключение происходит) происходит правила трансляции виртуальных адресов в физические. 

Например в команде `move [123] х` используются виртуальные адреса, то есть процессор обычно манипулирует виртуальными адресами, но данные естественно хранятся в физической памяти, адресуемой физически, и контроллер памяти (__MMU__) - то самое устройство, которое обеспечивает трансляцию. Трансляция, как можно заметить по картинке - кусочно-непрерывна. _Квантом_ трансляции называется страница (раньше назывался _frame_), кусок данных от 4 Кбайт до 2 - 4 Мбайт, бывают и другие ранжировки, зависящие от конкретных реализаций __MMU__ и того, как он был запрограммирован для работы той или иной ОС.

_Контроллер памяти_ - микросхема, имеющая несложный интерфейс в форме привилегированного регистра на процессоре, то есть __CR3__ на __x86__, __TTBR__ на __arm__, способ запрограммировать контроллер - в этом месте находятся правила трансляции и далее ответственность ОС предварительно создать осмысленные правила трансляции (это называется __page tables__) и дать возможность контроллеру работать с предоставленными данными.


## Адресное пространство, трансляция

![2.2](https://github.com/aslastin/os_course/blob/master/images/2.2%20address%20space%2C%20translation.png)

Здесь подробнее рассмотрим процесс трансляции.

Если у нас есть виртуальный адрес на процессоре, то этот виртуальный адрес делится на 2 части: обычно по бинарной маске, сколько-то последних бит, например 12, в случае типичных 4 Кбайтных страниц, описывают сдвиг, то есть сдвиг внутри страницы, а вся дальнейшая часть адреса описывает номер виртуальной страницы. 

Это является один входом для правил трансляции, а другим входом для правил трансляции является __page table base register__, что по сути, является физическим адресом того места, где находится таблица трансляции. Если быть занудой :) контроллер памяти и __MMU__ - разные вещи. __MMU__ - часть самого процессора, обеспечивающая физическую трансляцию, а _контролер памяти_ - нечто, что обеспечивает передачу данных от процессора в память. Они часто рассматриваются как единое целое для цели рассуждения об ОС. 

Процессору нужно понять, где находится та или иная физическая страница по заданной виртуальной странице. Пусть адрес - 123. Смотрю, что записано в __page table base register__ (__CR3__ на __x86__) и нахожу по номеру виртуальной. Адрес виртуальной страницы позволяет осуществлять индексацию в многоуровневом дереве - __page tables__, базовый регистр указывает на его корень, от этого корня мы производим индексацию по первым нескольких битам, потом в том месте, куда мы попали, производим индексацию по нескольких следующим битам до тех пор пока не находим листовую запись: _«Ага, физический адрес для данного адреса такой»_ и кроме этого выясняем, что это та самая страница, к которой нам нужно обратиться.  Кроме этой информации, в той же самой листовой записи хранится дополнительная информация о правах доступа, то есть о том, как можно обращаться с этой страницей.

__Page frame__ - номер физической страницы. 

Большинство адресов в ядре - виртуальные, но есть 2 класса адресов, которые не являются виртуальными: _адреса, связанные с физической адресацией памяти_, то есть __page table base register__ и все записи в таблице страниц; _адреса для демо-транзакций_ - транзакций, которые инициируются не процессором, а физическим устройством; мы должны этому устройству также отдавать физический адрес, потому что правил логической трансляции не существует за пределами ЦП. 


## Поток исполнения

![2.3](https://github.com/aslastin/os_course/blob/master/images/2.3%20exec%20thread.png)

Можно сказать, что процесс - поток исполнения. Слева -  однопоточный процесс, справа - многопоточный. У каждого процесса есть свое адресное пространство, то есть данные, код, то есть то, к чему мы обращаемся по конкретному физическому адресу, оно всегда одно и тоже. 

Но есть что-то уникальное для того или иного потока: набор регистров, с разной семантикой - есть целочисленные, векторные, регистры с плавающей точкой (большинство современных физических процессоров - регистровая машина, в которой есть набор из нескольких (от 8 до нескольких сотен) ячеек памяти, адресуемых предсказуемым образом, и они называются _регистрами_. Все манипуляции с ними очень быстрые, так как они находятся напрямую в процессоре и почти все команды манипулируют с содержимыми этих регистров). 

Все эти регистры образуют состояние потока исполнения. Один из эти регистров - указатель на текущую инструкцию (__counter__) - что сейчас делает процессор. Обычно это выделенный регистр, то есть с точки зрения набора команд не совсем такой как остальные регистры. 

_Стек_ - кэш, арена, где хранятся локальные данные для функций. 

Такие вещие как файлы, глобальные данные - разделяемы между потоками исполнения. Потоки выполняются независимо друг от друга, при этом манипулируют одними и теми же ресурсами. 

При смене контекста состояния регистров процессора сохраняются в специальную структуру, ассоциированную с данным логическим потоком, то есть ядро знает все потоки исполнения, и, если мы будем говорить про планировщик, даже в многопоточном процессе есть список всех потоков внутри данного процесса. 


## Системный вызов, syscall

![2.4](https://github.com/aslastin/os_course/blob/master/images/2.4%20syscall.png)

При функционировании программы, ей часто необходимо сделать то, что напрямую она не может. Ей необходимо обратиться с просьбой сделать это что-то к ядру ОС. 

Здесь помогает наличие такой концепции как _системный вызов_. Пусть мы пишем код на __C__ и хотим воспользоваться функцией `write` из библиотеки __libc__ - эта функция, пользуясь интерфейсом ядра, инструкциями, выполняет задуманное. 

Слева - картинка для пользователя (системный вызов -> магия ->  ждем окончания системного вызова, в этот момент поток исполнения заморожен, то есть находится в состоянии системного вызова). 

Теперь давайте войдем в ядро и посмотрим, что там происходит. _Enter the Kernel_. 

Привилегированная инструкция, например `syscall`, работает следующим образом: при вызове этой функции происходит переключение в контекст ядра, а дальше выбирается тот или иной обработчик, в зависимости от значений регистров, во время которых произошел этот вызов. При системном вызове мы всегда можем посмотреть - _а какие значения сейчас у пользовательских регистров?_. 

И здесь, в зависимости от ОС, например в Linux, в регистре __rix__ - номер системного вызова (все доступные операции пронумерованы от 0 до нескольких сот), то есть интерфейс ядра вещь предсказуемая и известная. 

Первое, что мы смотрим в обработчике - какой номер системного вызова, попадает ли он в диапазон системных вызовов доступных системе. Если попадает, то передаем управление на вектор (функция ядра), которая реализует вызов `open`, дальше эта функция валидирует корректность передаваемых значений.

Например, что является аргументами системного вызова `open`? Путь к файлу - строчка с терминированными нулями, то есть с точки зрения процессора - регистр, в котором записан виртуальный адрес в терминологии процесса, который осуществил системный вызов. 

При исполнении системного вызова у ядра всегда есть контекст, то есть знание того, какой конкретный процесс сделал этот вызов. Кроме этого, ядро может читать/писать данные в адресном пространстве процесса. 

В случае `open`, если в первом аргументе написано 123, мы проверили, что это легитимный адрес, в адресном пространстве процессора, дальше мы скопировали в ядро ту терминированную нулем строчку и передали на вход ядерной реализации `open`, которая уже по этому пути попытается найти, есть ли такой файл, имеет ли пользователь, ассоциированный с данным процессом сделать требуемую операцию и.т.д. 

Если все успешно - ядро открывает файл и запись полученного файлового дескриптора в таблице файловых дескрипторов процесса, после чего в каком-то регистре, записывает целое число, являющееся номером файлового дескриптора. 

Внутри ядра реализована нетривиальная машинерия обработки стандартизированного интерфейса, которое предоставляет то или иное ядро. В разных ядрах ОС нумерация вызов и их семантика разная, именно это обеспечивает различие между ОС.


## Сигналы, сихнронные, асинхронные

![2.5](https://github.com/aslastin/os_course/blob/master/images/2.5%20(a)synch%20signals.png)

Что такое сигнал? С точки зрения приложений, _сигналы_ - механизмы синхронной или асинхронной коммуникации с внешним миром, то есть, если мы что-то делаем, то мы можем сделать это каким-то одним образом, либо исполнение может пойти по исключительному пути. 

_Сигнал_ - механизм уведомления о том, что случилось нечто неожиданное. При этом похожие уведомления может прислать и другой процесс. Здесь важно, что у сигнала есть обработчики по умолчанию и у многих сигналов обработчик по умолчанию - завершить исполнение, но можно поставить и пользовательский обработчик, чем пользуются системные ПО, например виртуальные машины.

Допустим мы написали в ту область памяти, куда писать нельзя - результат - прерывание исполнения процесса, но если мы на самом деле специально это сделали, чтобы при записи в некоторый регион нужно прерывать исполнение чего-то (таким образом в __JVM__ реализован механизм _safe pointing_ - остановки исполнения всех потоков __VM__ для сборки мусора).

Тогда мы можем поставить свой обработчик и на действие прерывания и сообщения о том, что я не могу писать в эту память, я хочу вызвать некоторый обработчик, а уже из него решить - продолжать исполнение или также завершить исполнение, если это результат ошибки в программе, а не ожидаемая семантика. 

Сигналы классифицируются на 2 вида: _синхронный сигнал_ - сигнал, который происходит в результате действия пользователя (могли сталкиваться с сигналом __sigsegv__ - segmentation violaton - нарушение сегмента, тех адресов, куда можно писать/читать, __sigbus__ - инициируется когда происходит исполнение невалидной инструкции); _асинхронные_ - происходят по причинам, независимых от человека. 

В UNIX любому процессу можно позвать любой сигнал с помощью __kill id__. Процесс -> syscall -> ОС. ОС -> результат системного вызова, сигнал - посылка какого-то действия на исполнение в контексте процесса -> процесс. 

Как получается, что процесс исполнялся, исполнялся, а потом начинает исполнять совершенно непонятный обработчик? 

С точки зрения ядра, процесс исполняется на процессоре и его никто не тревожит. Потом может случиться то или иное событие (прерывание, отказ от дальнейшего исполнения процесса), и в этот момент ядру ОС нужно сделать 2 вещи: сохранить контекст исполнения данного потока исполнения, то есть набор всех регистров в специально-выделенное для них место и решить, какой код исполнять дальше - другого процесса, того же - поднять новое значение виртуальных регистров, при этом не переключать адресное пространство и так далее. 

Внешнее периферийное оборудование также посылает ядру сигналы, которые должны быть обработаны.

Откуда ОС знает куда писать? Она не проверяет каждую запись/чтение, она описывает правила доступа в виде таблиц трансляций, а потом __MMU__ при каждом доступе к памяти проверяет тип доступа и если соответствия не происходит - происходит __memory fault__. 

Синхронные сигналы - причинно-следственная связь, которая доставлена не в виде кода возврата, а в виде перехода потока исполнения на обработчик сигналов. 

Асинхронный - никак не связан с вашим кодом, а с чем-то внешним.


## Ресурсы, файлы

![2.6](https://github.com/aslastin/os_course/blob/master/images/2.6%20resources%2C%20files.png)

_Процесс_ - не только поток исполнения, адресное пространство, но и некоторые ресурсы. 

Эти ресурсы ассоциированы с процессом: представим, что мы открываем из 2 процессов один и тот же файл, это значит мы создали запрос к ОС с помощью механизма системного вызова, и система создала в таблице файлов некоторую запись, причем создала его в своих собственных структурах, где хранится информация, что это за файл, текущее положение в нем и указатель на _v-node_ - то место, где уже хранится информация о настоящем файле. 

Ресурс файлового дескриптора ассоциирован с файлом, а ресурс конкретной реализации ассоциирован с конкретной файловой системой.


## Освобождение ресурсов 

Одна из ценностей ОС - они качественные ресурс-менеджеры, и позволяют писать людям не очень качественные довольно емкие программы, которые при этом могут считать, что, если они даже забыли освободиться, ресурсы автоматически вернутся системе. 

- В какой момент происходит автоматическое освобождение ресурсов? Можно делать и ручное освобождение (с помощью __close__ -> ассоциированные с ним записи в таблице дескрипторов убираются и ресурсы освобождаются). Кроме этого, есть неявное закрытие при любом завершении процесса (процесс завершился -> уведомил ОС об этом с помощью __exit syscall__), также процессу мог прийти фатальный сигнал -  процесс не зарегистрировал никакого обработчика для данного сигнала, поэтому будет освобождение ресурсов.

- В отличие от языков, таких как C++, где есть такая концепция как деструкторы - кастомизированное освобождение того или иного ресурса, в случае ресурсов ОС, очень важным инвариантом является то, что освобождение ресурсов - это то, что ядро может сделать самом по себе, без помощи процесса, потому что процесс в момент освобождение ресурса уже нельзя исполнять, поэтому вся работа по освобождению должна быть в ядре.

- Освобождаются: виртуальная память, файловые дескрипторы (файлы, сокеты), семафоры (примитивы синхронизации, обеспечивающие синхронизацию внутри процесса на уровне ОС)

- Не освобождаются (необходимо освобождать явно): разделяемая память (`shmget(2)` - типичная терминология в UNIX мире - значит, что надо посмотреть с помощью команды `man` инструкцию по системному вызову, а номер 2 - секция  `man`), коды выхода (`wait(2)`, процессы-зомби) (когда процесс завершается, у него есть свойство, которое остается после его завершения - код выхода, это то, значение, которое является результатом исполнения программы. В UNIX существует такой системный вызов как `wait(2)`, который позволяет передать __pid__ и ждать до момента выхода процесса, а если вышел, получить значение его кода выхода. 

Но у этого есть любопытное свойство: если никого не интересует _status code_ процесса, но тем не менее ОС сохраняет записи про такие процессы, потому что не известно мб кто-то в будущем захочет вызвать `wait` и узнать, что случилось с данным процессом, поэтому в UNIX существует такое состояние у процесса как _процесс-зомби_ - процесс завершился, но значение его кода выхода не прочитано, поэтому ассоциированный __pid__ не завершен => его нельзя переиспользовать, поэтому нужно что бы кто-нибудь вызвал `wait`. 


## fork(2) и exec(2)

![2.7](https://github.com/aslastin/os_course/blob/master/images/2.7%20fork(2)%2C%20exec(2).png)

Как же эти процессы появляются? В случае UNIX, запуск нового процесса похож на деление биологических организмов: существует системный вызов `fork`: в родительском процессе мы получаем идентификатор того процесса, который создался, а в детском процессе получаем `0`, то есть по коду можем отличить, в каком процессе оказались, при этом устройство того или иного процесса не сильно различается: у них один и тот же код, данные (используется оптимизация _copy-on-write_). 

При `fork` появляется 2 одинаковых копии одного и того же, однако на практике нам редко такое нужно, поэтому используются в паре системные вызовы `fork` и `exec`: `fork` создает новый дочерний процесс, а `exec`  замещает сегмент кода и сегмент данных, тем сегментом кода и данных из того исполняемого файла, который передан в качестве аргументов. 

Если говорить высокоуровнево:  `fork` - создание нового адресного пространства с содержимым идентичным оригинальному адресному пространству, а `exec` - замещение содержимого текущего адресного пространства сегментами кода/данных/стека, описанного в том или ином исполняемом файле. 


## posix_spawn(2)

![2.8](https://github.com/aslastin/os_course/blob/master/images/2.8%20posix_spawn(2).png)

В достаточно скором времени появился альтернативный системный вызов - `pspawn`, который позволяет исполнять новый процесс - запускается новый процесс без клонирования существующего адресного пространства и он создается заново. 


## Организация памяти процесса

![2.9](https://github.com/aslastin/os_course/blob/master/images/2.9%20mem%20process%20org.png)

_Текст_ - исполняемый код, который попал в бинарный формат. 

_Дата_ - сегмент данных, которые инициализированы.

_BSS_ - сегмент глобальных, но не инициализированных данных, куда пишутся всякие объекты, например `int array [10]`. 

_Heap_ - та часть, где располагаются динамические аллокации, и в отличие от _BSS_ сегмента, его границы переменны, так как в процессе исполнения мы не знаем, сколько нам данных понадобится - heap растет.

Глобальные константы, если они _read-only_ - попадают в текст. 

В современных ОС разадресация нулевого или близко к нулевому указателю - ошибка, симптом неверного исполнения данной программы, поэтому  и есть пустой блок снизу. 

_Стек_ - место, где сохраняются автоматические переменные программы. Стек растет вниз из=за простоты аллокации. 

_Адресное пространство ядра_ - его могло бы и не быть, но тогда стоимость операции системного вызова была бы высокой - при системном вызове нужно переключать адресное пространство на адресное пространство ядра, а не только уровень привилегий и начинать исполнение и нетривиальным образом читать данные из адресного пространства процесса - поэтому чтобы избежать таких накладок в адресное пространство процесса отображается ядро, причем отображается так, чтобы к адресам в ядре можно было обращаться на `0` уровне привилегий. 

Разберем ошибку переполнения стека: если напрямую сделать ту картинку, которая здесь нарисована, то никакой ошибки не возникнет - мы просто случайным образом начнем считать данные из heap, поэтому реальные системы часто реализую __guard pages__, несколько страниц замапленных в конце стека при доступе к котором приходит ошибка исполнения (тут есть свои тонкости, так как, чисто теоретически их можно перезаписать).


## SMP, NUMA

![2.10](https://github.com/aslastin/os_course/blob/master/images/2.10%20SMP%2C%20NUMA.png)

_Мультипроцессорная система_ - система, в которой много логических процессоров, каждый со своим уникальным кэшем, есть общая шина или набор общих шин, общая память (на больших кластерах используется __NUMA__ - в зависимости от того, к какому конкретно физическому адресу памяти мы обратились - производительность разная - разной группе процессоров  атрибутирован тот или иной кусок памяти).


## Планировщик

![2.11](https://github.com/aslastin/os_course/blob/master/images/2.11%20schedular.png)

Для исполнения процесса нужен некоторый набор функционала, в частности для исполнения кода. Часть ядра ОС, которая отвечает за этот функционал - ___планировщик (scheduler)___. Это важная компонента, которая состоит из 2 взаимодополняющих механизмов:

1. __Preemption__ - позволяет перевести процесс из состояния "_запущен_" (исполняется на процессоре) в состояние "_не запущен_" и наоборот. Этот механизм позволяют ядру ОС контролировать исполнение других программ/потоков исполнения. Это сводится к сохранению и восстановлению значений регистров, регистров общего назначения, где хранятся текущие данные, с которыми работает программа, включая `stack-pointer` и одного или несколько регистров специального назначения, таких как `CR3`.

2. __Выбор того, а какую следующую задачу исполнять.__ Здесь решается задача "предсказания будущего". Планировщик не знает, каким образом процессы, которые он планирует, будут исполняться. Процессы также часто не знает, что они будут делать. Например, если это сетевой сервер, то он не знает какие запросы получит от сети. Чтобы аппроксимировать решение этой "нерешаемой" задачи, была придумана концепция планировщика.

Если посмотреть на картинку __P0__ и __P1__ - 2 процесса _RA_, то есть физически исполняемые элементы, которые позволяют некоторое время исполнять некоторую задачу. Все время в компьютере есть задачи, потенциально готовые к исполнению - _task queue_. Дальше в зависимости от набора некоторых факторов происходит помещение этих задач на исполнение, то есть переключение того или иного процессора на выполнение той или иной задачи. Может происходить и обратное действие - снять задачу с исполнения, поместить назад в очередь исполнения и перейти к другой задаче. Этой общий алгоритм функционирования любого планировщика. 


## Алгоритмы планировщика

![2.12](https://github.com/aslastin/os_course/blob/master/images/2.12%20schedular%20algo%20.png)

Но если мы говорим о реальных системах, например о Linux, то в качестве основного планировщика используется _CFS (Completely fair process scheduling)_. Суть функционирования данного планировщика состоит в том, что он пытается максимально точно аппроксимировать условный идеальный планировщик. Это условный идеальный планировщик алгоритмически из себя представляет процессор, который исполняет все процессы, которые хотят исполняться, пропорционально их потребностям, то есть, если есть 5 процессов, каждый из них хочет исполняться сколько-то, то нужно, чтобы каждому доставалось по 1/5 процессора.

Для реализации этой идеи был предпринят кардинальный дизайн ОС - _шаг_ - создается красно-черное дерево, отсортированное по тому, сколько времени эффективно исполнялась та или иная задача, и просто всегда берем на исполнение ту задачу, которая исполнялась меньше всего и поддерживаем у каждой задачи характеристику _Virtual Run Time_, которая описывает, сколько времени данная задача была на процессоре. В результате мы имеем автоматически отсортированное дерево, которое очень легко модифицировать в процессе работы ядра. 

Почитать об этом поподробнее можно [здесь](https://opensource.com/article/19/2/fair-scheduling-linux).


## Настройка планировщика 

Планировщик кроме базового режима функционирования имеет некоторые способы настройки: 

- __Приоритеты процессов/потоков.__ Неформально, можем сказать, что этот процесс "более важный", либо "менее важный". Если есть выбор, какому из процессов исполняться, отдается предпочтение процессу с наибольшим приоритетом. То же самое верно и для потоков. 

Например, если мы работаем на десктопной ОС с аудио, то поток, декодирующий аудио и видео, имеет высокие требования по отношению к приоритету того, когда он в точности попадает на процессор. Если видео будет дергаться, по причине "некомпетентного" планировщика, будет неприятно. Поэтому для мультимедиа и real-time часто используются высоко-приоритетные задачи. 

- __Группа планирования.__ Те задачи, которым осмысленно исполняться совместно. Наиболее простой пример группы планирования - потоки в одном процессе, то есть, если у нас есть особенно высоко-конкурентный процесс, например jvm runtime, где есть много потоков, которые потенциально конкурируют за одни и те же ресурсы, то разумно делать планирование, чтобы они попадали на процессор более менее одновременно и могли проделать максимально много работы, даже если им нужна взаимная зависимость исполнения.

- __Вычислительно-интенсивные и IO-ориентированные процессы.__ Вторые это те, которые в основном ничего не делают, но пишут и читают данные из сетевых соединений из файлов итд. В практических целях это значит, что вычислительные процессы чаще всего используют полностью свой _time-slice_ (выделенный кусочек времени процессу), а IO-ориентированные процессы что-либо чуть-чуть делают, а потом сами отказываются от дальнейшего исполнения, потому что заблокированы на ввод/вывод. 

- __Process afinity и кеши.__ Бывают ситуации, когда разработчик системы лучше знает, каким-либо образом должен исполняться тот или иной процесс, нежели планировщик. Например, есть очень важная задача, под которую мы готовы выделить 1 процессор, так чтобы на нем ничего не другого не исполнялось, а наша задача исполнялась только на этом процессоре. Такое поведение называется _process afinity_ (привязка процесса к процессору). Это важно для задач, связанных с интенсивной работой с памятью. Есть много данных в кешах, но при переключении процессов кеши инвалидируются, однако при привязке процесса к процессору можно добиться лучшего использования кеша. 

- __Процессы реального времени и sched_setscheduler(2).__ Процессы, которые отвечают за манипуляцию с физическими устройствами - процессы реального времени - каждый раз, когда они хотят исполняться, они должны исполняться. 

- __Потенциальные проблемы: starvation, priority inversion.__ Планировщик решает сложную задачу, поэтому в нем есть несколько классических проблем, таких как _starvation_ - когда процессу не достается исполнение, _priority inversion_ - у одного процесса более высокий приоритет, чем у другого, но при этом он зависит от более низко приоритетного процесса для продолжения выполнения, поэтому ничего не может сделать :(. В Windows для решения __priority inversion__ есть группа приоритетов и программные проверки, которые исключают возможность взять ресурс, за который отвечает процесс более низкого приоритета, процессу с более высоким уровневым приоритета.
