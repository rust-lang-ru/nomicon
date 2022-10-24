% Границы времени жизни

У нас есть следующий код:

```rust,ignore
struct Foo;

impl Foo {
    fn mutate_and_share(&mut self) -> &Self { &*self }
    fn share(&self) {}
}

fn main() {
    let mut foo = Foo;
    let loan = foo.mutate_and_share();
    foo.share();
}
```

Ожидаем, что он компилируется. Мы вызываем `mutate_and_share`, который временно
заимствует `foo` как изменяемую ссылку, но затем возвращает только как общую
ссылку. Поэтому мы ожидаем, что `foo.share()` выполнится успешно, ведь `foo` уже
не должна быть заимствована как изменяемая ссылка.

Однако, когда мы попытаемся выполнить компиляцию:

```text
<anon>:11:5: 11:8 error: cannot borrow `foo` as immutable because it is also borrowed as mutable
<anon>:11     foo.share();
              ^~~
<anon>:10:16: 10:19 note: previous borrow of `foo` occurs here; the mutable borrow prevents subsequent moves, borrows, or modification of `foo` until the borrow ends
<anon>:10     let loan = foo.mutate_and_share();
                         ^~~
<anon>:12:2: 12:2 note: previous borrow ends here
<anon>:8 fn main() {
<anon>:9     let mut foo = Foo;
<anon>:10     let loan = foo.mutate_and_share();
<anon>:11     foo.share();
<anon>:12 }
          ^
```

Что произошло? Ну, причина все та же, что и в [примере 2 из предыдущей
секции][ex2]. Уберем синтаксический сахар из программы и получим следующее:

```rust,ignore
struct Foo;

impl Foo {
    fn mutate_and_share<'a>(&'a mut self) -> &'a Self { &'a *self }
    fn share<'a>(&'a self) {}
}

fn main() {
	'b: {
    	let mut foo: Foo = Foo;
    	'c: {
    		let loan: &'c Foo = Foo::mutate_and_share::<'c>(&'c mut foo);
    		'd: {
    			Foo::share::<'d>(&'d foo);
    		}
    	}
    }
}
```

Система времени жизни вынуждена продлить время жизни `&mut foo` до времени `'c`
из-за времени жизни `loan` и сигнатуры mutate_and_share. Дальше, когда мы
пытаемся вызвать `share`, и она видит, что мы пытаемся взять ту же ссылку, что и
`&'c mut foo`, все взрывается у нас на глазах!

Программа абсолютно корректна в части семантики ссылок, о которой мы на самом
деле заботимся, но система времени жизни слишком крупнозерниста, чтобы понять
это.


TODO: другие общие проблемы? SEME regions stuff, mostly?




[ex2]: lifetimes.html#example-aliasing-a-mutable-reference
