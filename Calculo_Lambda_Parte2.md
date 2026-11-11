# Cálculo Lambda — Parte 2

**EIF400 Paradigmas de Programación · Repaso para el II Parcial (viernes 13/11, 10:00 am)**
Temas del SPEC: Parte 2, secciones 2c–2e (reducción, estrategias, aritmética y lógica, combinador Y). Ejercicios e–i de la sesión 32.

---

## 1. Redex y forma normal

### Redex
**Redex** (*reducible expression*) es el único lugar donde se puede hacer una β-reducción. Tiene la forma exacta:
```
(λx. M) N        ← una λ escrita, entre paréntesis, pegada a su argumento
```
En JS: una flecha que se llama en el mismo lugar donde se escribe: `(x => x + 1)(5)`.

> **Analogía:** una receta escrita con el ingrediente al lado, lista para cocinar.

| Expresión | ¿Redex? | Por qué |
|---|---|---|
| `(λx. x) a` | ✅ | λ escrita + argumento |
| `f a` | ❌ | `f` es un nombre, no una λ escrita |
| `λx. x` | ❌ | nadie le pasa argumento |
| `λx. x y` | ❌ | es `λx. (x y)`: el cuerpo llega hasta el final; adentro `x` es variable, no λ |
| `a (λx. x)` | ❌ | la λ está a la derecha (es el argumento) |
| `g (λx. x)` | ❌ | `g` es un nombre |

**Truco:** busca un `(λ...)` cerrado seguido de algo. Los redex pueden estar **escondidos adentro**: `f ((λx. x) y)` o `λa. (λb. b) a`.

### Forma normal
Una expresión está en **forma normal** cuando **no tiene ningún redex en ninguna parte**: es el resultado final.
> **Analogía:** simplificar una fracción hasta `2/3`.

- Si existe, es **única** (salvo α-equivalencia), sin importar el orden de reducción.
- **No siempre existe** (por ejemplo Ω).
- **No es decidible** saber de antemano si existe.

**Ejemplo con dos caminos al mismo resultado:**
```
λa. (λb. b) a
  →β  λa. a          (b ← a)
  →η  λb. b          (solo reenvía: λa. (M a) con M = λb. b, que no usa a)
λa. a  ≡α  λb. b     ← la misma identidad
```

### Forma normal débil (WNF, slide 27)
No hay redex **en la raíz**, aunque sí adentro: `f ((λx. x) y)` está en WNF pero no en forma normal. (Un regalo envuelto: por fuera listo, por dentro no.)

---

## 2. Estrategias de evaluación: eager contra lazy

| | **Aplicativa** (eager, by-value) | **Orden normal** (lazy, by-need) |
|---|---|---|
| Regla del profe | *left-most inner-most first* | *left-most outer-most first* |
| En palabras | primero los **argumentos** | primero la **función de afuera**; argumentos solo si se usan |
| Quién la usa | JS, Java, Python | Haskell |
| ¿Termina? | **no siempre** | si existe forma normal, **la encuentra** (normalizante) |

> **Analogía:** el cocinero eager pica todos los ingredientes antes de leer la receta; el lazy pica solo lo que la receta pide.

### Ejemplo: `(λx. λy. x) a ((λz. z) b)`
Tiene 2 redex: `(λx. λy. x) a` (afuera) y `(λz. z) b` (adentro, en el argumento).
```
EAGER (3 pasos)                          LAZY (2 pasos)
(λx. λy. x) a ((λz. z) b)                (λx. λy. x) a ((λz. z) b)
→β (λx. λy. x) a b        z ← b          →β (λy. a) ((λz. z) b)    x ← a
→β (λy. a) b              x ← a          →β a                      y ← ... (se descarta sin calcular)
→β a                      y ← b (se descarta)
```
Mismo resultado; eager calculó `(λz. z) b` y luego lo botó.

### Ω: la expresión sin forma normal
```
Ω = (λz. z z)(λz. z z)   →β   (λz. z z)(λz. z z)   →β ...   ♾️ (dos espejos frente a frente)
```
Paso: cuerpo `z z`, reemplazar **las dos** `z` por `(λz. z z)` → queda igual.

| Expresión | Eager | Lazy | ¿Forma normal? |
|---|---|---|---|
| `(λx. y) Ω` | ♾️ | `y` ✅ (Ω se descarta) | sí: `y` |
| `(λx. x x) Ω` | ♾️ | `Ω Ω` → ♾️ | **no** |

Lazy solo se salva si el argumento problemático **no se usa**.

### En JS (eager)
```javascript
const primero = a => b => a
const bucle = () => bucle()
primero("hola")(bucle())          // 💥 se cuelga: JS calcula bucle() antes (RangeError)
primero("hola")(() => bucle())    // "hola" ✅  thunk: lo hace lazy
false && bucle()                  // false: && y ?: ya son lazy
```

---

## 3. Numerales de Church

**Un número n es "la función que aplica algo n veces".**
```
n̂ = λs. λz. sⁿ z        s = la acción que se repite · z = el punto de partida
```
| Número | Cálculo λ | JS |
|---|---|---|
| 0 | `λs. λz. z` | `s => z => z` |
| 1 | `λs. λz. s z` | `s => z => s(z)` |
| 2 | `λs. λz. s (s z)` | `s => z => s(s(z))` |
| 4 | `λs. λz. s (s (s (s z)))` | `s => z => s(s(s(s(z))))` |

> **Analogía:** un número es una **orden de repetición** ("estampa 3 veces"), no un dato. Las `s` son las **capas** de una cebolla y `z` es el **centro**.

**Propiedad fundamental:** `n̂ s z = s(s(...s(z)...))` (s aplicada n veces).

**Ver el número:** darle `s` y `z` concretos.
```javascript
TRES(x => x + 1)(0)       // 3
TRES(x => x + "★")("")    // "★★★"
DOS(x => x * 10)(1)       // 100  (1 → 10 → 100: cada vuelta aplica s al resultado anterior)
```
⚠️ `DOS(x => x * 10)(1)` es **100**, no 20: el 2 no se multiplica; es **cuántas veces** se aplica la acción.

### SUCC
```
SUCC = λn. λs. λz. s (n s z)          JS: n => s => z => s( n(s)(z) )
```
"Las n repeticiones (`n s z`) **y una más** (`s` de afuera)." Tiene tres λ porque recibe un número `n` y **devuelve un número** (que espera `s` y `z`).
> **Analogía:** un sello que estampa n veces → fabricar uno que estampe n + 1: usar el viejo y estampar una vez más.

**Prueba general:** `SUCC n̂ s z = s (n̂ s z) = s (sⁿ z) = sⁿ⁺¹ z = (n+1)^ s z`.

**Prueba formal `SUCC 1̂ = 2̂`** (renombrar 1̂ con α para no confundir nombres):
```
SUCC 1̂
=   (λn. λs. λz. s (n s z)) (λa. λb. a b)
→β  λs. λz. s ((λa. λb. a b) s z)          n ← 1̂
→β  λs. λz. s ((λb. s b) z)                a ← s
→β  λs. λz. s (s z)                        b ← z
=   2̂   ∎
```
**Prueba `SUCC 0̂ = 1̂`:**
```
SUCC 0̂
=   (λn. λs. λz. s (n s z)) (λa. λb. b)
→β  λs. λz. s ((λa. λb. b) s z)            n ← 0̂
→β  λs. λz. s ((λb. b) z)                  a ← s   (no hay a en el cuerpo: s se descarta)
→β  λs. λz. s z                            b ← z
=   1̂   ∎
```

### PLUS y MULT
| Operación | Cálculo λ | JS | Idea |
|---|---|---|---|
| `SUCC` | `λn s z. s (n s z)` | `n=>s=>z=> s(n(s)(z))` | n veces y **una** más |
| `PLUS` | `λm n s z. m s (n s z)` | `m=>n=>s=>z=> m(s)(n(s)(z))` | n veces y **m** más (escalera) |
| `MULT` | `λm n s z. m (n s) z` | `m=>n=>s=>z=> m(n(s))(z)` | m veces "**s, n veces**" (cajas de huevos) |

**Diferencia clave:** en PLUS, a `n` le das `s` **y** `z` (produce un resultado). En MULT, a `n` le das **solo** `s` (produce una **acción nueva**, "una caja", que `m` repite).

**Pruebas:**
```
PLUS m̂ n̂ s z = m̂ s (n̂ s z) = m̂ s (sⁿ z) = sᵐ(sⁿ z) = sᵐ⁺ⁿ z
MULT m̂ n̂ s z = m̂ (n̂ s) z = (n̂ s)ᵐ z = (sⁿ)ᵐ z = sᵐˣⁿ z
```

**Ejemplos resueltos:**
```javascript
PLUS(UNO)(DOS)(x => x + 1)(0)
= UNO(s)( DOS(s)(0) )       // primero DOS: 0 → 2 ; luego UNO sobre eso: 2 → 3
= 3

MULT(TRES)(DOS)(x => x + 1)(0)
= m( n(s) )(z)                         // cuerpo de MULT
= n(s)( n(s)( n(s)( z ) ) )            // m = TRES: 3 capas de n(s) (DOS aplica "su acción", sea cual sea)
= s(s( s(s( s(s( z )) )) ))            // n = DOS: cada capa son 2 s
= s(s(s(s(s(s( 0 ))))))                // s = x => x+1, z = 0
= 6                                    // 3 cajas × 2 huevos
```
**Por qué `DOS(A)(z) = A(A(z))`:** con otros nombres `DOS = f => x => f(f(x))`: "dame cualquier acción y la aplico dos veces". En MULT, la acción que recibe `m` es `n(s)`.
⚠️ Errores típicos: poner la acción en el centro y `z` afuera (es al revés: `s` son las capas, `z` el centro); paréntesis desbalanceados (tantos `)` como `s(`).

---

## 4. Booleanos

```
TRUE  = λx. λy. x        "de dos opciones, elige la PRIMERA"     JS: x => y => x
FALSE = λx. λy. y        "de dos opciones, elige la SEGUNDA"     JS: x => y => y
NOT   = λp. λx. λy. p y x   (le pasa las opciones intercambiadas)
ITE   = λp. λt. λe. p t e   (if-then-else ≈ p ? t : e)
```
> **Analogía:** un booleano **es una decisión**. ITE es el mesero que le ofrece las dos opciones al cliente (el booleano), que ya sabe cuál quiere.

### Ejercicio e: ¿qué significa `TRUE = ZERO`?
**FALSE** = `λx. λy. y` y **0̂** = `λs. λz. z` son **α-equivalentes** (misma expresión, otros nombres). El `=` del profe es **α-equivalencia**. (Ojo: en la slide dice `FALSE = ZERO`; el enunciado del ejercicio escribe `TRUE = ZERO`, pero el que coincide con cero es FALSE.)

### Ejercicio f: verificar ITE
⚠️ El enunciado escribe `p => t => f => p(t)(e)`: error de tipeo (`f` en vez de `e`). La correcta es `p => t => e => p(t)(e)` (`logica.ts`).
```
ITE TRUE 'YES' 'NO'
=   (λp. λt. λe. p t e) TRUE 'YES' 'NO'
→β  (λt. λe. TRUE t e) 'YES' 'NO'      p ← TRUE    (se va λp.)
→β  (λe. TRUE 'YES' e) 'NO'            t ← 'YES'   (se va λt.)
→β  TRUE 'YES' 'NO'                    e ← 'NO'    (se va λe.)
=   (λx. λy. x) 'YES' 'NO'
→β  (λy. 'YES') 'NO'                   x ← 'YES'
→β  'YES'                              y ← 'NO' (y no aparece: se descarta)  ∎

ITE FALSE 'YES' 'NO'
→β ... →β  FALSE 'YES' 'NO'            (tres primeros pasos iguales)
=   (λx. λy. y) 'YES' 'NO'
→β  (λy. y) 'NO'                       x ← 'YES' (x no aparece en λy. y: 'YES' se descarta)
→β  'NO'                               y ← 'NO'  ∎
```
⚠️ **En cada β, la λ desaparece junto con su argumento** (una λ menos y un argumento menos por paso).
⚠️ **Regla del descarte:** si el parámetro no aparece en el cuerpo, el argumento no tiene dónde ir y se bota.

### Ejercicio g: OR (y AND)
**Truco:** escríbelo primero como un `if-else` normal y copia lo que devuelve cada rama.
```javascript
const OR  = p => q => ITE(p)( TRUE )( q )      // si p: ya ganó · si no: decide q
const AND = p => q => ITE(p)( q )( FALSE )     // si p: decide q · si no: ya perdió
```
```
OR  = λp. λq. ITE p TRUE q          AND = λp. λq. ITE p q FALSE
```
⚠️ Dentro de la función se usa el **parámetro** `q`, no `X` (eso sería una variable libre).
⚠️ `ITE(p)(TRUE)(FALSE)` o `ITE(p)(TRUE)(p)` **no** son AND: no usan `q` (fallan con p = TRUE, q = FALSE).

| p | q | OR | AND |
|---|---|---|---|
| TRUE | TRUE | TRUE | TRUE |
| TRUE | FALSE | TRUE | FALSE |
| FALSE | TRUE | TRUE | FALSE |
| FALSE | FALSE | FALSE | FALSE |

> OR y AND son **espejos**: en una rama ya está decidido; en la otra, decide `q`.

---

## 5. Combinador Y: recursión sin nombres

### El problema
En el cálculo λ las funciones **no tienen nombre**, así que no pueden llamarse a sí mismas como `fact(n - 1)`.

### Paso 1: el truco a mano (autoaplicación)
```javascript
const factSelf = self => n => n === 0 ? 1 : n * self(self)(n - 1)
factSelf(factSelf)(3)    // 6
```
```
factSelf(factSelf)(2)          ← la llamada inicial (la haces tú)
= 2 * self(self)(1)            ← adentro: se pasa a sí misma
= 2 * (1 * self(self)(0))
= 2 * (1 * 1)                  ← n === 0: ya NO se hace self(self) → se detiene
= 2
```
> **Analogía:** una hoja de instrucciones que lleva **una fotocopia de sí misma**: "si n no es 0, fotocopia esta hoja, dásela a alguien con n − 1 y multiplica".

### Paso 2: Y lo hace automático
Escribes la versión "limpia" con un parámetro `g` y Y se encarga del `self(self)`:
```javascript
const factGen = g => n => n === 0 ? 1 : n * g(n - 1)
const fact = Y(factGen)       // fact(5) = 120
```
> **Analogía:** Y es una fotocopiadora automática / una fábrica que mete dentro de la función un sobre con una copia de sí misma.

**Propiedad (punto fijo):** `Y g = g (Y g)`. Si `h = Y(g)`, `h` es la versión recursiva de `g`: cuando `g` llama a su parámetro, en realidad llama a `h`.
(Punto fijo: valor que una función no cambia; ej. 1 para `f(x) = x²`.)

### El Y del profe (`Y.ts`)
```javascript
export function Y(F){
    let U = f => f(f);                  // 🪞 espejo: f se recibe a sí misma (el self(self))
    let Z = f => x => U(f)(x);          // ✉️ sobre cerrado: lo mismo, pospuesto (lazy)
    return U( f => F( Z(f) ) );         // 🏭 arma F con un sobre que contiene a la función terminada
};
```
Desarrollo (con `W = f => F(Z(f))`):
```
Y(F) = U(W) = W(W) = F( Z(W) ) = F( x => W(W)(x) ) = F( x => Y(F)(x) )
```
→ `Y F = F (Y F)` con un "sobre" en medio.

**¿Por qué el sobre `x => ...`?** Porque **JS es eager**: sin él, `f(f)` se calcularía de inmediato una y otra vez (como Ω) → bucle infinito. Es una **η-expansión** (lo contrario de η) usada para retrasar el cálculo. En el cálculo λ puro (lazy) no haría falta: `Y = λf. (U f)(U f)` con `U = λf x. f (x x)` (slides 35–36).

**Por qué termina:** en el caso base el `if` elige el valor y **nunca abre el sobre**.

Para el examen basta saber **qué hace** y **cómo usarlo** (el ejercicio h dice "use Y aunque no sea claro cómo está implementado").

### Receta para usar Y
1. Escribe la recursión **normal** (con nombre).
2. Agrega **`g =>`** al inicio (**primero** `g`, que lo da Y; **después** el dato, que lo da el usuario).
3. Cambia **cada llamada a sí misma** por **`g(...)`**.
4. Úsala con **`Y(...)`**.

⚠️ El orden importa: `n => g => ...` está mal (Y metería la copia en el lugar de `n`).
⚠️ Usa el mismo nombre de parámetro que en el cuerpo (`g => y => ... n ...` deja `n` libre).
⚠️ `factGen` es el **molde**; `fact = Y(factGen)` es la función terminada.

### Ejercicio i: factorial
```javascript
const factGen = g => n => n === 0 ? 1 : n * g(n - 1)
const h = Y(factGen)          // h(n) = n!
```

### Ejercicio h: máximo de un arreglo no vacío (slide 40)
```javascript
// normal
const maximo = a => a.length === 1 ? a[0] : Math.max(a[0], maximo(a.slice(1)))
// para Y
const maxGen = g => a => a.length === 1 ? a[0] : Math.max(a[0], g(a.slice(1)))
const h = Y(maxGen)
h([3, 9, 2])    // 9
```
```
h([3, 9, 2]) = Math.max(3, h([9, 2]))
             = Math.max(3, Math.max(9, h([2])))
             = Math.max(3, Math.max(9, 2))      // un solo elemento: caso base
             = 9
```
> **Analogía:** un torneo: el primero se compara con el ganador del resto. (Es el `[F | R]` de Prolog: `a[0]` = F, `a.slice(1)` = R.)

| | Factorial | Máximo |
|---|---|---|
| Para Y | `g => n => n===0 ? 1 : n * g(n-1)` | `g => a => a.length===1 ? a[0] : Math.max(a[0], g(a.slice(1)))` |
| Caso base | `n === 0` → 1 | un elemento → ese elemento |

---

## 6. Resumen rápido

| Tema | Recordar |
|---|---|
| Redex | `(λx. M) N`: λ escrita entre paréntesis + argumento; pueden estar adentro |
| Forma normal | sin redex; única si existe; puede no existir (Ω); indecidible |
| Eager | primero argumentos (JS); no siempre termina |
| Lazy | primero la función; normalizante; se salva si el argumento no se usa |
| Ω | `(λz. z z)(λz. z z)` → a sí misma |
| Numeral | `n̂ = λs z. sⁿ z`: repetir s n veces; s capas, z centro |
| SUCC / PLUS / MULT | una más / m más / m veces "n veces s" |
| TRUE / FALSE | elige primero / elige segundo; FALSE ≡α 0̂ |
| ITE | `λp t e. p t e` (≈ `p ? t : e`) |
| OR / AND | `ITE p TRUE q` / `ITE p q FALSE` |
| β paso a paso | una λ menos y un argumento menos por paso; si el parámetro no aparece, se descarta |
| Y | `Y g = g (Y g)`; `g => dato => ... g(...)`; sobre `x =>` por ser JS eager |
