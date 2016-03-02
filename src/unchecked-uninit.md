% Непроверяемая неинициализированная память

Одним интересным исключением из этого правила является работа с массивами.
Безопасный Rust не разрешит вам частично инициализировать массив. При
инициализации массива вы должны или установить всем одно и то же значение `let x
= [val; N]`, или установить каждому члену отдельно `let x = [val1, val2,
val3]`. К сожалению, это довольно негибко, особенно если вам нужно
инициализировать массив более инкрементальным или динамичным способом.

Небезопасный Rust дает вам мощный инструмент для решения этой проблемы:
`mem::uninitialized`. Эта функция делает вид, что возвращает значение, когда в
действительности она вообще ничего не делает. Используя ее, мы можем убедить
Rust в том, что переменная у нас инициализирована, и это позволяет делать хитрые
вещи с условной или инкрементальной инициализацией.

К сожалению, это открывает и кучу проблем. Присвоение имеет разный смысл для
Rust, если он считает, что переменная инициализирована или наоборот. Если
считается, что переменная не инициализирована, то Rust просто семантически
сделает memcopy новых бит на место неинициализированных и ничего больше. Однако,
если Rust считает, что значение инициализировано, он попытается выполнить `Drop`
старого значения! Из-за того, что мы обманули Rust в части того, что значение
инициализировано, мы больше не можем безопасно использовать обычное присвоение.

Эта же проблема возникает, если вы работаете с сырым размещением в памяти,
которое возвращает указатель на неинициализированную память.

Для решения этого мы должны использовать модуль `ptr`. В частности, он
предоставляет три функции, которые позволяют присваивать байты определенному
месту в памяти, не удаляя старое значение: `write`, `copy` и
`copy_nonoverlapping`.

* `ptr::write(ptr, val)` берет `val` и заносит его по адресу `ptr`.
* `ptr::copy(src, dest, count)` копирует биты, занимающие количество `count` 
  из src в dest. (это эквивалент memmove - заметьте, что порядок аргументов 
  перевернут!)
* `ptr::copy_nonoverlapping(src, dest, count)` делает то же, что и `copy`, но 
  немного быстрее, основываясь на предположении, что две области памяти не 
  пересекаются. (это эквивалент memcpy -- заметьте, что порядок аргументов 
  перевернут!)

Надеюсь не надо говорить, что эти функции в случае неправильного использования
приведут к серьезному хаосу или прямиком к Неопределенному Поведению.
*Единственным* требованием этих функций является то, что используемые области
должны находится в памяти. Однако способов, которым запись произвольных бит в
произвольное место в памяти может все сломать, нет числа!

Объединяя все, получаем:

```rust
use std::mem;
use std::ptr;

// длина массива жестко закодирована, но это легко поменять. Это означает, что мы
// не можем использовать синтаксис [a, b, c] для инициализации массива!
const SIZE: usize = 10;

let mut x: [Box<u32>; SIZE];

unsafe {
	// убеждаем Rust, что x Абсолютно Инициализирована
	x = mem::uninitialized();
	for i in 0..SIZE {
		// очень аккуратно переписываем каждый индекс, не читая его
		// Внимание: безопасность исключений не важна; Box не может вызвать панику
		ptr::write(&mut x[i], Box::new(i as u32));
	}
}

println!("{:?}", x);
```

Стоит отметить, что вам не нужно волноваться о махинациях в стиле `ptr::write` с
типами, которые не реализуют или не содержат тип `Drop`, потому что Rust знает
что для них не надо пытаться вызвать деструктор. Аналогично, можно выполнять
присвоения полям частично инициализированных структур напрямую, если эти поля не
содержат типы `Drop`.

Однако, работая с неинициализированной памятью, вам надо постоянно следить,
чтобы Rust не попытался вызвать деструктор значений, которые вы создали, до их
полной инициализации. Каждый контрольный путь, содержащий область видимости этой
переменной, должен инициализировать ее до своего конца, если у нее есть
деструктор. *[This includes code panicking](unwinding.html)*.

Вот и все по работе с неинициализированной памятью! Обычно  неинициализированная
память нигде не обрабатывается, поэтому, если вы собираетесь передавать ее по
кругу всем, вам следует быть *очень* осторожным.