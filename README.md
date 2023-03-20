# FlashKit-MD-Plus

Продвинутая и расширенная редакция утилиты работы с интерфейсом (программатором) перезаписываемых Sega MegaDrive/Genesis картриджей – FlashKit MD.

![FlashKit](https://user-images.githubusercontent.com/24475390/161397449-f073c6df-1232-4550-8ccc-8e3eef341054.jpg)

Очень рекомендуется выставить минимальную задержку в настройках USB-COM-порта интерфейса FlashKit (особенно актуально для MX29L3211 в сочетании с некоторыми чипсетами на стороне ПК). Даже при «1» задержка между транзакциями остается дико-большой и рандомно-непредсказуемой.

![Setup](https://user-images.githubusercontent.com/24475390/161397487-74491ecd-fec6-49bb-b07d-e522ec5b05ff.jpg)

История версий:

### 1.0.0.0
* Стоковая версия [исходников](https://krikzz.com/pub/support/flashkit-md/) от Krikzz

### 1.0.1.0 (непубличная)
+ Добавлена поддержка работы с микросхемой памяти картриджа Intel PA28F400 (4Mbit);
+ Добавлена функция полного стирания микросхем памяти картриджа и последующей проверкой на чистоту;
* Небольшие косметические редакции в коде и информационных сообщениях.

Intel PA28F400 реализует лишь пословную (16-ти разрядный «байт») запись. Управляющая команда на запись предусматривают один шинный цикл, плюс цикл передачи данных. Состояние регистра статуса автоматически выставляется на линии данных после записи/стирания (не требуется шинный цикл к флэшке с управляющей командой на чтение статуса). На практике для экономии общего времени, можно не инициировать проверку статуса после записи, т.к. накладно это – лишняя usb-транзакция передачи команды и получения ответа от программатора (для каждого слова!). Достаточно того что между вызовом функций отправки usb-транзакцией пакета (массива) данных на запись очередного слова происходит вынужденная (неконтролируемая) пауза (хотя всё и тупо в цикле одно за другим). Результат: ~100kB/min (т.е. 5мин на всю микросхему в 512kB).

### 1.0.2.0 (первая публичная - посвящается проекту [Flash-Cart 4MB](https://github.com/MiGeRA/MD-Flash-Cart-4MB))
+ Добавлена поддержка работы с микросхемой постоянной памяти картриджа MXIC MX29L3211 (32Mbit);
+ Оптимизирован алгоритм работы с Intel PA28F400 (кратное увеличение скорости);
* Дальнейшие оптимизации информационных сообщений о процессе работы с памятью, затраченном времени, контрольных суммах и прочем сопутствующем;
* Корректировка определения размера дампа при чтении: приоритет информации из штатного заголовка (а уж если там размер вне диапазона – то используем базовый метод чтоб прочесть максимум хоть что-то).
* Много что еще по мелочам.

Почему так быстро (меньше минуты все 4MB) пишется стоковая серия 29W? В ней есть команда «байпаса», позволяющая при записи каждого слова не подавать команду записи в три шинных цикла флэшки, а использовать сокращенную команду в 1 цикл, да еще при этом упрощен режим фиксации завершения записи слова: на линии данных выставляется не байт статуса, а само записанное слово. Программатор внутри FPGA реализует логику анализа записанного и прочитанного после результата и продолжает дальше «шпарить» из длинного непрерывного потока: одноцикловую команду с последующим словом данных, используя также функцию автоинкремента адреса. «Пулять» в принципе, с точки зрения микросхемы памяти, можно поток любой длины – хоть за раз весь объем. Алгоритм работы с программатором предусматривает деление на куски максимального размера в 64kB – это дает возможность, в том числе, для визуализации шкалы прогресса. Изящное решение получилось у krikzz'а именно в совокупности архитектуры программатора и картриджа с конкретной моделью микросхемой памяти к нему.

Серия 29L (в отличие от 29W, примененной в стоковом решении) не имеет команды «байпаса», но зато предусматривает страничную запись вплоть до 128 слов (256 байт). Запись каждой страницы должна сопровождаться командой из трех шинных циклов флэшки. Потом нужно быстро, не мешкая передавать данные по нужным последовательным адресам в пределах кратности размеров страницы от младших линий адреса (порядок не имеет значения согласно документации, т.е. можно и от старших адресов заполнять буфер, например). Если замешкался – на запись уходит то, что успел загрузить - остальное остается чистым (FF). Завершение записи страницы предусмотрено чекать чтением статуса, автоматически выставляемого на линии данных после ухода данных на запись страницы. При последующей проверке чтением - нужно выявлять страницы с «недописанными» словами и (без предварительного стирания!) просто дописывать («дожигать») их (повторная запись и проверка). Почему именно MX29L3211? – это одна из немногих доступных сейчас в продаже микросхем памяти в корпусе (P)SOP-44 и объемом в 32Mbit, со страничной записью и ценой меньше доллара за штуку.

Intel PA28F400 удалось оптимизировать, причем очень существенно! Казалось бы «программистский изыск»: поместить все данные в единый массив, а не вызывать подряд в цикле несколько раз функцию отправки данных, но выигрыш во времени колоссальный! – аж микросхема не успевает … Еще бы, ведь от чтения статуса мы оказались (в прошлой версии). Возвращать его? – лишние usb-транзакции с программатором, что в итоге минимум в два раза дольше результата достигнутого в 1.0.1.0 Конфигурация FPGA программатора не предусматривает чтение и парсинг состояния байта статуса внутри себя без usb-транзакций, но зато предусматривает возможность генерации задержки работы своего «автомата» измеряемую во внутренних циклах («попугаях», экспериментально это где-то полмикросекунды на единицу). Генерировать задержку отдельным пакетом (от софта к программатору) – терять время сопоставимое команде чтение статуса. Но задержку можно «завернуть» в общий непрерывный массив команд-данных! Да, максимальная задержка в «системе команд» программатора - 7 единиц, но можно их подряд формировать несколько – экспериментально подобрано: 4 раза по 5 (что по замерам около 10 микросекунд). На время задержки программатор вхолостую тратит свои внутренние циклы бездействуя. Результат – вся микросхема пишется корректно за 10сек.

В версии 1.0.2.0 не реализована поддержка работы с памятью MX29LV320 (под которую изначально создавался проект [Flash-Cart 4MB](https://github.com/MiGeRA/MD-Flash-Cart-4MB)), китайцы ошиблись корпусом ;-\

### 1.0.3.0 (Проект [Flash-Cart 4MB](https://github.com/MiGeRA/MD-Flash-Cart-4MB) полностью завершен!)
+ Добавлена поддержка работы с микросхемой постоянной памяти картриджа MXIC MX29LV320 (32Mbit);
* Косметические правки в интерфейсе, коде и сообщениях.

### 1.0.3.1 (оптимизация)
+ Существенно оптимизирована работа с MXIC MX29L3211 (добавленная в версии 1.0.2.0) в части единообразия скоростных характеристик и успешности результата на различных USB-хост контроллерах, версиях их драйверов и операционных системах (протестированы многие варианты);

К некоторому сожалению, только теперь можно сказать о полной совместимости и гарантированно успешном результате (на всех разнообразных протестированных компах и конфигурациях) для MX29L3211. Вообще же, по факту, программатор FlashKit ориентирован на работу с микросхемами памяти, где запись «пословная», а подтверждением успешного завершения операции записи служит (внутренняя на стороне FlashKit) проверка того, что микросхема памяти выставляет на линии данных переданное ей и успешно записанное слово (в итоге максимум скорости при непрерывном, однонаправленном потоке данных с контролем порционности его получения и записи на стороне программатора). Но есть микросхемы памяти, которые выставляют не переданные (и записанные) данные, а байт статуса. И если для архаичной PA28F400 это скорее можно списать на ее древность и отсутствие (в то время) не только стандартов, но и традиций. То для MX29L3211 чтение байта статуса для фиксации завершения и контроля успешности выполнения операции записи – неизбежность. Запись у данной микросхемы исключительно страничная: за одну транзакцию можно записать до 128 слов (256 байт). FlashKit (со штатной CPLD-конфигурацией) не умеет проверять сам валидность битов в байте статуса и ожидать положенного срока … Варианта два. Запускать обратную транзакцию по USB-соединению на чтение байта статуса (в цикле, до успеха) – это медленно, даже если делать это приходится лишь для каждых 256 байт, плюс к этому задержки между транзакциями носят вероятностный характер и могут сильно отличаться на разных хост-машинах (так было реализовано в версии 1.0.2.0). Причем единичный цикл (с переключением потока по USB запись-чтение-запись) занимает порой больше времени, чем необходимо микросхеме для записи страницы. А до кучи, если команду записи страницы и поток данных страницы отправлять разными транзакциями, то временная пауза между ними может быть такой (больше 100мкс.) что на запись вообще ничего не уйдет («пакет» уйдет пустым). С точки зрения кода на C# две, друг за другом идущие, строчки программы с транзакциями в USB-COM-порт имеют между собой рандомный промежуток времени, и не маленький – а переключение направления передачи еще большую задержку имеет (COM-порт двунаправленный, а USB-соединение поверх которого он эмулируется лишь полудуплексное). Вобщем, если увлекаться чтением байта статуса по USB-соединению – то многим более чем 50% всего потраченного времени будет именно в задержках между сишными процедурами port.Write и/или port.ReadByte (равно как и при немалом количестве однонаправленных), никак не контролируемых нами, как программистами высокого уровня (винда такая, дрова иже с ней и т.п). Вариантом остается только пихать в единый массив много-много команд (к программатору и микросхеме) и отправлять их одной USB-транзакцией. Но какже проверка статуса? А никак – нужно апгреживать прошивку CPLD FlashKit (будет время – может займусь, ресурс там есть). Но с имеющимся в руках железом и конфигом CPLD можно организовать простую аппаратную задержку на стороне программатора, причем с высокой точностью (около 1мкс). Таким образом не читать и чекать состояние байта статуса – а просто ждать время по документации средствами FlashKit, а после установленным порядком отправлять на запись следующую страницу … И такой вариант работает более чем сносно! Вполне стабильно и даже несколько шустрее (нет потерь в рандомных задержках между транзакциями). Да, «дожиг» в ряде случаев (по результатам проверки записи) остается как и раньше неизбежным – но это не занимает много времени и максимум (в худшем варианте из моих тестирований) может иметь объем до 10%. Что такое «дожиг»? - случайным образом (закономерности не выявил) некоторые области памяти после операции записи (даже успешной, согласно байта статуса, если его читать) остаются чистыми (т.е. 0xFF). Такие «не прописавшиеся» участки нужно программировать еще раз, до победного … (без стирания, зачем? – если они и так чистые). Для простоты отправляем кратный размер страницы на запись повторно (повторное присвоение не мешает). Как-то так … Исходник содержит максимум комментариев, а также старого неактивного кода: порой для лучшего понимания в т.ч. того, что так делать не нужно, это пройденный этап ;-))

### 1.0.3.2 (добавлена работа со "[Взломщиком кодов](https://migera.ru/smd/mega-cc.html)")
+ "Взломщик кодов" добавлен отельным разделом. Предполагается, что в этот инструмент («проходной»-картридж) установлена, вместо однократки,  флэш-ПЗУ PA28F400 (формально туда нужна PA28F200, но её в наличии не оказалось – использовал и описал что было). Цель такой модификации – создание и отладка альтернативного ПО для имеющегося «железа». Новый проект назван «[Mega-CC](https://github.com/MiGeRA/Mega-CC)», его описание выходит за рамки данного проекта. Изменения тривиальны: нога #WE убрана с VCC и подключена на одноименную ногу ОЗУ; замкнут тест-поинт «B» для полноценного питания флэш-ПЗУ (необходимо для стирания и программирования); замкнут тест-поинт «A» (питать SRAM паразитным способом – антинаучно!);
+ Исправлен недочет приводящий к ошибке при попытке записи на MXIC MX29L3211 файла размером менее 16кбайт;

«Взломщик кодов» - чит-девайс позволяющий «фризить» значения в оперативной памяти во время игры. Аппаратная модификация «Взломщика кодов» совместно с данной утилитой позволяют менять «прошивку» как самого «взломщика», так и картриджа в него вставленного (в случае его поддержки данной утилитой).

![cc-flashcart|480x640,50%](https://user-images.githubusercontent.com/24475390/185786711-2fec5dc4-dc5a-4656-96b8-8216998d5735.jpg)

### 1.0.3.3 (оптимизации в части удобства отладки)
Добавлено несколько опций в виде чек-боксов. Их действие распространяется на все кнопки. По-умолчанию значение чек-боксов соответствует конфигурации настроек предыдущих версий, так что если их не трогать – можно считать что их как бы и нет, и ничего не изменилось. Но польза от них может проявиться при попытке использования неподдерживаемых напрямую микросхем памяти, так сказать в отладочных целях и без перекомпиляции исходного кода настоящей утилиты:
+	«Checking Device ID» - позволяет отключить проверку аппаратного идентификатора микросхемы при операциях стирания и записи (анализируйте возможные последствия сами – программа просто честно применит алгоритм хоть к однократке). В первом разделе «29W-series» проверок нет в любом случае – это стоковый код без существенных изменений и рассчитанный на неопределенный (в моем представлении) парк микросхем (ограничивать его пока не вижу смысла);
+	«Erasing before Write» - позволяет отключить стирание микросхемы перед записью данных в нее. Может быть полезно, если: микросхема чистая априори, или только что стерта отдельно соответствующей кнопкой (например, ради проверки корректности стирания) и чтоб не расходовать теперь лишний цикл и время, или требуется дозапись;
+	«Read max volume» - позволяет активировать чтение максимально полного объема заданного типа микросхемы, невзирая на заголовок содержимого и зеркала (обусловленные меньшим размером и отсутствием старших линий адреса). Полезно в нестандартных случаях и отладочных целях, в штатном случае получите овердамп;
+	«Verifying after Erase» - позволяет отключить проверку микросхемы на чистоту после стирания. Можно существенно сократить время на операции стирания (по отдельной кнопке) если есть уверенность в ее успешности (например, по итогом многократных экспериментов).

Также исправлены мелкие недочеты, возникающие при стечении совокупности определенных обстоятельств.

### 1.0.3.4 (немного новинок)
Косметические правки в части интерфейса для понимания, что различия не в моделях микросхем как-таковых, а в алгоритмах программирования поддерживаемых ими (одна микросхема может поддерживать, например, как классический пословный алгоритм, так и «ускоренный» байпасный).
+	Добавлена поддержка картриджей с размером 8Мбайт (64Мбита), на примере микросхемы MX26L6420 – подробности в [материале на сайте](https://migera.ru/smd/csp232.html). Громко сказано: изменения в программе формальные – увеличен буфер и счетчики максимального размера для пословного алгоритма;
+	Добавлена поддержка работы (чтения/записи) со встроенной(ными) микросхемами памяти плода русского-народного фабричного творчества под названием Magistr-16 (aka «дисководная сега») через USB-интерфейс Эвердрайва (программатор FlashKit для данной группы опций не используется и не нужен);
+	Добавлена поддержка управления маппером EverDrive-MD v2, да того самого единственно задокументированного и поддерживаемого исходниками в SGDK. Общий размер микросхемы памяти в данном картридже 8МБайт (29W640). В рамках данной утилиты можно выбрать часть адресного пространства (младшую – boot, или старшую - game) для дальнейшей работы с ней (с частью) в байпасном режиме как с целой четырехмегабайтной микросхемой. Учесть, что общее стирание сотрет содержимое обеих частей! Также можно активировать доступ к SRAM и читать/писать содержимое FRAM-микросхемы памяти ее реализующую на данном картридже (доступ к «основной» памяти в режиме активации SRAM сокращается до двух мегабайт – полная совместимость с классической моделью маппинга). Таким образом можно восстановить работоспособность Эвердрайва данной версии из любого состояния при исправном железе, а также вдоволь поэкспериментировать с ним (без пайки!);
+	Выведена наружу (реализованная ранее) тестовая функция проверки оперативной памяти взломщика кодов;

Фактически со временем утилита вырастает уже в среднестатистическую софтину управления одним из программаторов, а стало быть её интерфейс нужно универсализовывать …На будущее в планах полностью переработать интерфейс, но с нуля колдовать лениво – так что пока в поисках и раздумьях. А не сделать ли вообще консольную версию?

Напомню еще раз, что данная утилита носит экспериментальный характер и не ставит перед собой целью «защиту от дурака» во всех немыслимых и нелогичных комбинациях, некоторые защиты есть – но они порой только мешают, поэтому некоторые можно отключить опциями: под ответственность и понимание  пользователя. Скомпилированный бинарник предоставляется «как есть» (что сгенерила IDE из представленных исходников в один клик) - что-то не нравится? Флаг вам в руки – правьте исходник … Буду рад (и думаю не только я) конструктивным дополнениям в код данной утилиты.

Зазипованный экзешник в [разделе релизов](https://github.com/MiGeRA/FlashKit-MD-Plus/releases).

*PS.* Респект автору программатора FlashKit за столь изящный (удобный и функциональный) инструмент для работы с картриджами MegaDrive/Genesis, несомненно он лучший из ныне существующих открытых проектов (доступна топология платы для изготовления, исходник конфигурации CPLD и управляющей утилиты - развитием которой является данный проект).

*PPS.* Однако, авторская редакция программатора от krikzz [продается](https://krikzz.com/our-products/accessories/flashkitmd.html) по завышенной цене, [тут можно купить](https://aliexpress.ru/item/1005001468895818.html) ([или здесь](https://aliexpress.ru/item/1005001468330610.html)) его рестайлинговую версию от KY-tech (имхо более изящно спроектированную, хоть и схемотехника один-в-один) почти на 30% дешевле.
