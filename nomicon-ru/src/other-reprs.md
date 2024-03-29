% Альтернативные представления

Rust позволяет вам определять альтернативные стратегии расположения данных в
памяти, отличающиеся от стратегии по умолчанию.

# repr(C)

Это самая важная `repr`. У нее простая цель: делать то, что делает Си. Порядок,
размер и выравнивание полей полностью соответствуют тому, что вы ожидаете от Си
или C++. Все типы, проходящие через границы FFI, должны иметь `repr(C)`. Связано
это с тем, что Си - это лингва-франка в мире программирования. Она также
необходима, чтобы правильно делать некоторые сложные трюки с размещением данных
в памяти вроде интерпретации чисел, как значений других типов.

Но надо держать в голове особенности взаимодействия с более экзотическим
размещением данных Rust в памяти. Из-за своего двойственного назначения - "для
FFI" и "для контроля размещения в памяти", `repr(C)` можно применять к типам,
которые нечувствительны или, наоборот, проблемны при прохождении границ
FFI.

* ТНР не занимают места, даже учитывая, что это не стандартное поведение в Си, и 
что это явно противоречит поведению пустых типов в C++, которые занимают байт 
свободного места.

* ТДР, кортежи и типы-суммы не существуют в Си и, следовательно, всегда 
являются FFI небезопасными.

* Кортежные структуры похожи на структуры, если смотреть относительно `repr(C)`,
 с единственным отличием, заключающимся в отсутствии имен у полей.

* **Если у типа есть [drop flags], они все равно будут добавлены**

* Для перечислений это эквивалентно одному из `repr(u*)` (смотри следующий
раздел). Выбранный размер является размером по умолчанию перечислений для
С ABI целевой платформы. Помните, что представление перечислений в Си зависит от
реализации, поэтому все это на самом деле только догадка. В частности, это может
быть не так, если интересующий код на Си компилировать с определенными флагами.



# repr(u8), repr(u16), repr(u32), repr(u64)

Для создания Си-подобных перечислений нужно указать размер. Если дискриминант
переполняет целое, в который его нужно уместить, возникнет ошибка компиляции. Вы
можете вручную попросить Rust явно заменять переполняющийся элемент на 0. Но
Rust не разрешит вам создать перечисление, в котором два варианта будут иметь
одинаковый дискриминант.

Для не-Си-подобных перечислений, это запретит выполнять определенные оптимизации,
такие как оптимизации нулевого указателя.

Эти представления никак не влияют на структуры.




# repr(packed)

`repr(packed)` заставит Rust убрать любой паддинг и выравнять тип по байту. Это
 улучшит использование памяти, но появятся негативные побочные последствия.

В частности, большинство архитектур *строго* требуют выравнивать значения. Это
означает, что обращение к памяти по невыровненному адресу выполняется дольше (x86),
или прервется с ошибкой (на некоторых чипах ARM). Для простых случаев, как прямая 
загрузка или сохранение упакованного поля, компилятору удастся сгладить проблемы
выравнивания сдвигами и масками. Но если вы создадите ссылку на упакованное
поле, очень маловероятно, что компилятору удастся избежать невыровненной
загрузки.

**[Для Rust 1.0 это может вызвать неопределенное поведение.][ub loads]**

`repr(packed)` использовать непросто. Если у вас нет строгих требований, лучше
ее не использовать.

Это - модификатор для `repr(C)` и `repr(rust)`.

[drop flags]: drop-flags.md
[ub loads]: https://github.com/rust-lang/rust/issues/27060
