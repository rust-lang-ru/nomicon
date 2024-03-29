# Безопасность исключений

Хотя программы и должны использовать размотку редко, есть много кода, который *может* запаниковать. Если вы делаете unwrap у None, индекс вне границ массива или делите на 0, ваша программа вызовет панику. В режиме отладки каждая арифметическая операция при переполнении может вызвать панику. Если не быть очень аккуратным и не контролировать строго, какой код исполняется, нужно быть к этому готовым.

Готовность к размотке часто называется *безопасностью исключений* в остальном мире программирования. В Rust присутствуют два уровня безопасности исключений, с которыми можно столкнуться:

- В небезопасном коде *необходимо* соблюдать безопасность исключений в том смысле, что не позволять нарушать безопасность памяти. Назовём это *минимальной* безопасностью исключений.

- В безопасном коде *хорошо бы* соблюдать безопасность исключений до тех пор, пока программа работает правильно. Назовём это *максимальной* безопасностью исключений.

Как и в многих других ситуациях в Rust, небезопасный код должен быть готов к работе с плохим безопасным кодом в случае размотки. Код, который временно создаёт неправильные состояния, должен заботиться, чтобы не вызвалась паника в этом состоянии. В общем смысле это означает, что необходимо гарантировать, что только код, не вызывающий панику, выполняется пока все находится в неправильном состоянии или необходимо создать охранное значение, которое почистит это состояние в случае паники. Это не обязательно означает, что состояние во время паники должно быть полностью вразумительным. Мы должны только гарантировать, что это *безопасное* состояние.

Большая часть небезопасного кода является листовидной, и поэтому легко делается безопасной от исключений. Она контролирует весь запускаемый код, и большинство этого кода не вызовет панику. Однако работа с массивами временно неинициализированных данных путём вызова обработчика, предоставленного вызывающей стороной, не является чем-то необычным для небезопасного кода. Такой код должен быть аккуратным и подразумевать безопасность исключений.

## Vec::push_all

`Vec::push_all` - это временный хак, позволяющий очень эффективно увеличить Vec, используя срез обобщённых данных. Вот простая реализация:

<!-- ignore: simplified code -->

```rust,ignore
impl<T: Clone> Vec<T> {
    fn push_all(&mut self, to_push: &[T]) {
        self.reserve(to_push.len());
        unsafe {
            // не может переполниться, потому что мы только что зарезервировали его
            self.set_len(self.len() + to_push.len());

            for (i, x) in to_push.iter().enumerate() {
                self.ptr().offset(i as isize).write(x.clone());
            }
        }
    }
}
```

Мы обходим `push` для избежания избыточных проверок размера и `len` Vec, которые мы точно знаем. Логика абсолютно правильна, кроме маленькой проблемы: код не безопасен от исключений! `set_len`, `offset` и `write` - надёжны; `clone` - бомба, которую мы просмотрели.

Clone абсолютно не контролируется нами и свободно может вызвать панику. Если так случится, наша функция выйдет раньше времени и длина Vec будет слишком большой. Если к нему обратятся или удалят его, произойдёт чтение неинициализированной памяти!

Исправление тут очень простое. Если мы хотим гарантировать, что значения, которые мы *на самом деле* клонировали, удаляются, мы можем устанавливать `len` на каждом цикле итераций. Если мы просто хотим гарантировать, что не будет прочтена неинициализированная память, мы можем установить `len` после цикла.

## BinaryHeap::sift_up

Поднять наверх элемент в куче чуть сложнее, чем расширить Vec. Псевдокод будет следующим:

```text
bubble_up(heap, index):
    while index != 0 && heap[index] < heap[parent(index)]:
        heap.swap(index, parent(index))
        index = parent(index)

```

Буквальное переписывание этого кода на Rust абсолютно нормально, но характеристики производительности раздражают: исходный элемент постоянно меняется местами с соседним. Исправим на следующее:

```text
bubble_up(heap, index):
    let elem = heap[index]
    while index != 0 && elem < heap[parent(index)]:
        heap[index] = heap[parent(index)]
        index = parent(index)
    heap[index] = elem
```

Код гарантирует, что каждый элемент копируется максимально малое количество раз (на самом деле `elem` скопируется дважды в общем случае). Но теперь появились проблемы с безопасностью исключений! Все время существуют две копии одного значения. Если вызовется паника в функции, что-то будет дважды удалено. К сожалению, у нас также нет полного контроля над кодом: сравнение определяется пользователем!

В отличие от Vec исправление здесь не такое простое. Первым вариантом будет разбить пользовательский код и небезопасный код на две отдельные фазы:

```text
bubble_up(heap, index):
    let end_index = index;
    while end_index != 0 && heap[end_index] < heap[parent(end_index)]:
        end_index = parent(end_index)

    let elem = heap[index]
    while index != end_index:
        heap[index] = heap[parent(index)]
        index = parent(index)
    heap[index] = elem
```

Поломка пользовательского кода теперь больше не проблема, потому что мы ещё не меняли состояние кучи. Во время обращения к куче мы работаем только с данными и функциями, которым доверяем, поэтому не надо волноваться о возникновении паники.

Возможно, вы недовольны такой конструкцией. Конечно, это обман! Нам приходится выполнять сложное прохождение кучи *дважды*! Ладно, давайте стиснем зубы. Давайте *по-настоящему* смешаем ненадёжный и небезопасный код вместе.

Если б у Rust были `try` и `finally` как в Java, мы могли бы сделать следующее:

```text
bubble_up(heap, index):
    let elem = heap[index]
    try:
        while index != 0 && elem < heap[parent(index)]:
            heap[index] = heap[parent(index)]
            index = parent(index)
    finally:
        heap[index] = elem
```

Базовая идея проста: если сравнение вызывает панику, мы просто присваиваем потерянный элемент по логически неинициализированному индексу в куче и катапультируемся. Каждый, кто проходит кучу, видит её потенциально *несогласованной*, но по крайней мере мы избавились от двойного удаления! Если алгоритм нормально завершится, то независимо ни от чего в конце выполнится операция присвоения элемента по индексу.

Жалко, что у Rust нет такой конструкции, придётся накатать свою! Сделаем её одним из возможных способов - будем хранить состояние алгоритма в отдельной структуре и создадим для логики "finally" деструктор. Независимо от того, вызовется паника или нет, он выполнится и почистит все за нами.

<!-- ignore: simplified code -->

```rust,ignore
struct Hole<'a, T: 'a> {
    data: &'a mut [T],
    /// `elt` всегда `Some` от создания до удаления.
    elt: Option<T>,
    pos: usize,
}

impl<'a, T> Hole<'a, T> {
    fn new(data: &'a mut [T], pos: usize) -> Self {
        unsafe {
            let elt = ptr::read(&data[pos]);
            Hole {
                data: data,
                elt: Some(elt),
                pos: pos,
            }
        }
    }

    fn pos(&self) -> usize { self.pos }

    fn removed(&self) -> &T { self.elt.as_ref().unwrap() }

    unsafe fn get(&self, index: usize) -> &T { &self.data[index] }

    unsafe fn move_to(&mut self, index: usize) {
        let index_ptr: *const _ = &self.data[index];
        let hole_ptr = &mut self.data[self.pos];
        ptr::copy_nonoverlapping(index_ptr, hole_ptr, 1);
        self.pos = index;
    }
}

impl<'a, T> Drop for Hole<'a, T> {
    fn drop(&mut self) {
        // заполним заново hole
        unsafe {
            let pos = self.pos;
            ptr::write(&mut self.data[pos], self.elt.take().unwrap());
        }
    }
}

impl<T: Ord> BinaryHeap<T> {
    fn sift_up(&mut self, pos: usize) {
        unsafe {
            // Вытащим значение по `pos` и создадим hole.
            let mut hole = Hole::new(&mut self.data, pos);

            while hole.pos() != 0 {
                let parent = parent(hole.pos());
                if hole.removed() <= hole.get(parent) { break }
                hole.move_to(parent);
            }
            // Hole будет безусловно заполнена здесь; с паникой или нет!
        }
    }
}
```
