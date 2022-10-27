# Ограничения типажей высшего порядка (ОТВП, Higher-Rank Trait Bounds (HRTBs))

Типажи `Fn` в Rust - это уличная магия. Например, мы можем написать следующий код:

```rust
struct Closure<F> {
    data: (u8, u16),
    func: F,
}

impl<F> Closure<F>
    where F: Fn(&(u8, u16)) -> &u8,
{
    fn call(&self) -> &u8 {
        (self.func)(&self.data)
    }
}

fn do_it(data: &(u8, u16)) -> &u8 { &data.0 }

fn main() {
    let clo = Closure { data: (0, 1), func: do_it };
    println!("{}", clo.call());
}
```

Если мы попытаемся убрать синтаксический сахар так же, как мы делали в [главе про времена жизни], у нас возникнут проблемы:

<!-- ignore: desugared code -->

```rust,ignore
// Обратите внимание, что синтаксис `&'b data.0` и `'x: {` не валиден!
struct Closure<F> {
    data: (u8, u16),
    func: F,
}

impl<F> Closure<F>
    // where F: Fn(&'??? (u8, u16)) -> &'??? u8,
{
    fn call<'a>(&'a self) -> &'a u8 {
        (self.func)(&self.data)
    }
}

fn do_it<'b>(data: &'b (u8, u16)) -> &'b u8 { &'b data.0 }

fn main() {
    'x: {
        let clo = Closure { data: (0, 1), func: do_it };
        println!("{}", clo.call());
    }
}
```

Каким же образом нам выразить границы времени жизни типажа `F`? Мы должны предложить какое-нибудь время жизни, однако, оно не будет известно до тех пор пока мы не войдем в тело `call`! К тому же, это не какое-то фиксированное время; `call` работает с *любым* временем жизни, которое будет у `&self` в этот момент.

Такая работа требует магии ограничения типажей высшего порядка (ОТВП). Убрать синтаксический сахар можно так:

<!-- ignore: simplified code -->

```rust,ignore
where for<'a> F: Fn(&'a (u8, u16)) -> &'a u8,
```

Иначе:

<!-- ignore: simplified code -->

```rust,ignore
where F: for<'a> Fn(&'a (u8, u16)) -> &'a u8,
```

(где `Fn(a, b, c) -> d` - это сам по себе сахар для нестабильного *настоящего* типажа `Fn`)

`for<'a>` можно прочитать как "для всех возможных `'a`", и в общем случае это создаст *бесконечный список* границ типажа, которым должен соответствовать F. Сильно. Помимо типажей `Fn` есть не так уж много мест, где мы можем встретить ОТВП. И даже в этих случаях чаще всего нам поможет синтаксический сахар.

В итоге, мы можем переписать оригинальный код более явно:

```rust
struct Closure<F> {
    data: (u8, u16),
    func: F,
}

impl<F> Closure<F>
    where for<'a> F: Fn(&'a (u8, u16)) -> &'a u8,
{
    fn call(&self) -> &u8 {
        (self.func)(&self.data)
    }
}

fn do_it(data: &(u8, u16)) -> &u8 { &data.0 }

fn main() {
    let clo = Closure { data: (0, 1), func: do_it };
    println!("{}", clo.call());
}
```


[главе про времена жизни]: lifetimes.html