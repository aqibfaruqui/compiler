# Peri

A programming language for embedded systems to enforce hardware peripheral state safety at compile time.

## Language Features

### Peripheral Access Model

Peripherals require a state machine of their usage protocol.

```rust
peripheral Timer at 0x4000_0000 {
    states: Disabled, Enabled, Running;
    initial: Disabled;
    
    registers u32 {
        CTRL at 0x00;
        COUNT at 0x04;
    }
}
```

### Typestate Verification Model

Peripheral drivers are tagged with state transitions, these are enforced at compile time.

```rust
fn enable_timer() :: Timer<Disabled> -> Timer<Enabled> {
    Timer.CTRL = 1;
    return 0;
}

fn start_timer() :: Timer<Enabled> -> Timer<Running> {
    Timer.CTRL = 2;
    return 0;
}

fn boot_timer() :: Timer<Disabled> -> Timer<Running> {
    start_timer();
    enable_timer();
    return 0;
}

fn main() {
    boot_timer();
}
```


```
Error: Typestate violation in function 'boot_timer'
  --> example.peri:22
    |
 22 |     start_timer();
    |     ^^^^^^^^^^^^^ expected Timer<Enabled>, found Timer<Disabled>
```

Peripheral driver functions that do not call other driver functions are implicitly trusted, and any function that calls a driver function is verified by the compiler. More formally, Peri's typestate verification is based on [type](https://en.wikipedia.org/wiki/Type_system) and [effect](https://en.wikipedia.org/wiki/Effect_system) systems:

```
Σ  = Context
F  = Function
P  = Peripheral
Sᵢ = States

Axiom:
      sig(F) = P<S₁> → P<S₂>
    ───────────────────────────
    Σ[P ↦ S₁] ⊢ f() : Σ[P ↦ S₂]

Composition:
    Σ ⊢ s₁ : Σ₁    Σ₁ ⊢ s₂ : Σ₂
    ────────────────────────────
         Σ ⊢ s₁; s₂ : Σ₂

Branch:
    Σ ⊢ then : Σ₁    Σ ⊢ else : Σ₂    Σ₁ = Σ₂
    ──────────────────────────────────────────
       Σ ⊢ if e { then } else { else } : Σ₁
```

This maps to derivations in the [Simply Typed Lambda Calculus](https://en.wikipedia.org/wiki/Simply_typed_lambda_calculus) and [System F](https://en.wikipedia.org/wiki/System_F): peripheral state environments are typing contexts, typestate signatures are function types, parametric typestates correspond to bounded type variables, and verification is type derivation.

Future work includes formalising this in Lean.

### Parametric Typestates

Peri supports parametric typestates, allowing functions to express their state transitions on a range of states.

```rust
typestate CanSetDLAB = LCRSet & !DLABSet | Ready & !DLABSet;

fn uart_set_lcr_dlab()<S as CanSetDLAB> :: UART<S> -> UART<S & DLABSet> {
    UART::LCR = UART::LCR | LCR_DLAB;
}

fn uart_clear_dlab()<S includes DLABSet> :: UART<S> -> UART<S & !DLABSet> {
    UART::LCR = UART::LCR & ~LCR_DLAB;
}
```

- `S` represents the peripheral's current state
- The bounds `as` and `includes` constrain which states a function may be called with
- `&`, `|` and `!` create state combinations over a Boolean algebra

```
X = State variable
B = State bound
S = Current state

Parametric Call:
    Σ(P) = S    S ⊨ B(X)    sig(F) = ⟨X : B⟩ P<X> → P<out(X)>
    ─────────────────────────────────────────────────────────
              Σ[P ↦ S] ⊢ f() : Σ[P ↦ out(S)]
```

These allow drivers to declare state-dependent operations much more expressively than other languages, very cool 😎

## Installation and Usage

Requires Rust toolchain (rustc + cargo).

```sh
git clone https://github.com/aqibfaruqui/peri
cd peri
cargo install --path .
peric -S input.peri output.s
```
