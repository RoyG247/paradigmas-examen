# DCG, Combinadores y Recursión de Cola

**EIF400 Paradigmas de Programación · Repaso para el II Parcial (viernes 13/11, 10:00 am)**
Temas: DCG en Prolog, combinadores de parsers (`star`, `plus`), combinadores de `Array` (JS), `Stream` (Java) y listas (Prolog), `call`/`apply`, recursión de cola en cualquier lenguaje.

---

## 0. Aclaraciones del profesor (correos del 9/11 y 11/11)

| | Tema |
|---|---|
| ❌ **Sale del examen** | α-equivalencia, η-reducción, numerales de Church, combinador Y |
| ✅ **Entra** | β-reducción, generadores, combinadores en FP y LP |
| ⚠️ **Recomendado repasar** | combinadores de `Array` (JS), `Stream` (Java), `List` (Prolog); `apply`, `call`, `map`, `filter`, `reduce` y sus equivalentes en Prolog |
| ⚠️ **Entra** | recursión de cola en **cualquier** lenguaje: reconocerla o implementarla |
| ⚠️ **Entra** | lo visto en Java/JS y ya evaluado, implícita o explícitamente |

**Forro:** se pueden poner los combinadores, pero **sin ejemplos específicos**.

---

## 1. DCG (gramáticas en Prolog)

### La idea
Un parser es un **Pac-Man**: recibe un texto, se come un pedazo del principio y deja lo que sobra.
Todo parser tiene dos argumentos: **lo que entra** (`I`) y **lo que sobra** (`O`).

El texto es una **lista de códigos**: `` `42+1` `` es `[52, 50, 43, 49]`. El código de un carácter se escribe `0'c` (`0'7` es el código del `7`).

### La notación `-->` esconde `I` y `O`
```prolog
digit(D) --> [D], {is_digit(D)}.
% Prolog lo traduce a:
digit(D, I, O) :- I = [D | O], is_digit(D).
```

| En la DCG | Significa |
|---|---|
| `[C]` | cómete **un** código de la entrada y llámalo `C` |
| `"let"` o `[0'l, 0'e, 0't]` | cómete exactamente esos códigos |
| `{ … }` | Prolog normal: **no come nada**, solo revisa o calcula |
| `a, b` | primero `a`; lo que sobra se lo pasa a `b` |
| `[]` | no comas nada (siempre tiene éxito) |
| `!` | no te devuelvas a probar otra cláusula |
| `phrase(P, Texto)` | corre `P` exigiendo que **no sobre nada** |
| `phrase(P, Texto, Resto)` | corre `P` y devuelve lo que sobró |

### Ejemplo 1: espacios y dígitos (código de clase)
```prolog
ws --> [C], {is_ws(C)}, !, ws.      % un espacio y sigo
ws --> [].                           % o ninguno

digits([D | RD]) --> [D], {is_digit(D)}, !, digits(RD).
digits([]) --> [].

int(Num) --> ws, digits(Digits), {number_codes(Num, Digits)}.
```
```prolog
?- phrase(digits(L), `42+1`, Resto).
L = [52, 50],  Resto = [43, 49].        % se comió "42", sobró "+1"

?- phrase(int(N), `  42`).
N = 42.
```
⚠️ Este `int` acepta **cero dígitos** (porque `digits` puede dar `[]`), y `number_codes(N, [])` falla o da error. Hay que exigir al menos uno.

### Ejemplo 2: `peek` para ver qué está pasando
```prolog
peek(Msg, I, I) :- format('(~w) I=~w~n', [Msg, I]).   % no come nada: entra I, sobra I
digits([D | RD]) --> peek('R1.'), [D], {is_digit(D)}, !, digits(RD).
```
`peek` es un predicado normal con los dos argumentos `I`, `O` escritos a mano. Como no consume nada, lo que entra es igual a lo que sobra (`I, I`). Sirve para depurar.

---

## 2. Combinadores de parsers: `star` y `plus`

Un **combinador** es un predicado (o función) que **recibe otro** como argumento.
`star` y `plus` son el `*` y el `+` de las expresiones regulares:

| | Significa | Con texto `abc` y `digit` |
|---|---|---|
| `star(L, P)` | `P*`: cero o más | éxito, `L = []` |
| `plus(L, P)` | `P+`: una o más | **falla** |

```prolog
% star(LT, P, I, O): cero o más. LT recoge el resultado de cada llamada a P.
star([T | RT], P, I, O) :- call(P, T, I, O1), !, star(RT, P, O1, O).
star([], _, I, I).

% plus(LT, P, I, O): una o más = una obligatoria + cero o más.
plus([T | RT], P, I, O) :- call(P, T, I, O1), star(RT, P, O1, O).
```
- `call(P, T, I, O1)` ejecuta `P` agregándole tres argumentos: el resultado, lo que entra y lo que sobra.
- `plus` **no tiene segunda cláusula**: como exige al menos una, no existe el caso "no leí ninguna".
- `plus` delega el resto en **`star`**, no en sí mismo: si se llamara a sí mismo, cada llamada exigiría otra más y nunca podría terminar.

> **Analogía:** la entrada al cine: la primera persona **tiene que** pagar (obligatoria) y después entran las que quieran, incluso ninguna (`star`).

⚠️ El `plus` escrito en clase tenía la cláusula `plus([], _, I, I)`, así que en realidad se comportaba como `star`.

### Ejemplo 1: usarlos en una DCG
Como sus dos últimos argumentos son `I` y `O`, en una DCG se escriben sin ellos:
```prolog
digit(D)   --> [D], {is_digit(D)}.
digits(Ds) --> star(Ds, digit).                              % cero o más dígitos
int(N)     --> ws, plus(Ds, digit), {number_codes(N, Ds)}.   % al menos un dígito
```
Con `plus`, `int` ya no acepta cero dígitos: queda arreglado el problema de `number_codes`.

### Ejemplo 2: combinador sobre combinador
```prolog
numeros(Ns) --> plus(Ns, int).
?- phrase(numeros(Ns), `1 22 333`).
Ns = [1, 22, 333].
```

**Advertencias:**
- `P` debe **consumir algo** cada vez que tiene éxito. Si `P` puede tener éxito sin comer nada (como `ws`), `star(L, P)` entra en **ciclo infinito**.
- El `!` hace a `star` **glotón** (*greedy*): come todo lo que pueda y no se arrepiente.

**Comparación:** `star`/`plus` son a los parsers lo que `map`/`filter` son a las listas: orden superior. Es la misma idea de los *parser combinators* en Java/JS, con `call` en lugar de lambdas.

---

## 3. Combinadores de colecciones: JS, Java y Prolog

> **Analogía:** una fábrica con tres máquinas en fila. **`map`** transforma cada pieza (entran 5, salen 5). **`filter`** es control de calidad (salen 5 o menos, sin cambiar). **`reduce`** es la compactadora: junta todo en una sola cosa con un acumulador (bola de nieve).

### Ejemplo 1: duplicar, quedarse con los mayores que 4 y sumar
**JS (Array)**
```javascript
[1, 2, 3, 4].map(x => x * 2)                 // [2, 4, 6, 8]
            .filter(x => x > 4)              // [6, 8]
            .reduce((acc, x) => acc + x, 0)  // 14
```
**Java (Stream)**
```java
List.of(1, 2, 3, 4).stream()
    .map(x -> x * 2)
    .filter(x -> x > 4)
    .reduce(0, (acc, x) -> acc + x);         // 14
```
**Prolog (listas)**
```prolog
doble(X, Y) :- Y is X * 2.
mayor4(X)   :- X > 4.
suma(X, Acc0, Acc) :- Acc is Acc0 + X.

?- maplist(doble, [1,2,3,4], L1),    % L1 = [2,4,6,8]
   include(mayor4, L1, L2),          % L2 = [6,8]
   foldl(suma, L2, 0, R).            % R = 14
```

### Ejemplo 2: suma de las longitudes de los nombres con más de 4 letras
```javascript
["ana", "roberto", "luis", "carolina"]
    .map(x => x.length)               // [3, 7, 4, 8]
    .filter(x => x > 4)               // [7, 8]
    .reduce((acc, x) => acc + x, 0)   // 15
```
```prolog
largo(Nombre)       :- atom_length(Nombre, N), N > 4.
longitud(Nombre, N) :- atom_length(Nombre, N).
suma(X, Acc0, Acc)  :- Acc is X + Acc0.

total(Nombres, R) :-
    include(largo, Nombres, Largos),          % [roberto, carolina]
    maplist(longitud, Largos, Longitudes),    % [7, 8]
    foldl(suma, Longitudes, 0, R).            % 15
```
La salida de cada línea es la entrada de la siguiente: es el encadenamiento con puntos de JS, pero con variables intermedias.

### Tabla comparativa

| Qué hace | JS (Array) | Java (Stream) | Prolog |
|---|---|---|---|
| transformar cada uno | `map(f)` | `map(f)` | `maplist(P, L, L2)` |
| quedarse con los que cumplen | `filter(p)` | `filter(p)` | `include(P, L, L2)` |
| quitar los que cumplen | `filter(x => !p(x))` | `filter(x -> !p(x))` | `exclude(P, L, L2)` |
| juntar en un valor | `reduce(f, ini)` | `reduce(ini, f)` | `foldl(P, L, Ini, R)` |
| hacer algo con cada uno | `forEach(f)` | `forEach(f)` | `forall(member(X, L), G)` |
| ¿alguno cumple? | `some(p)` | `anyMatch(p)` | `member(X, L), p(X)` |
| ¿todos cumplen? | `every(p)` | `allMatch(p)` | `forall(member(X, L), p(X))` o `maplist(p, L)` |
| el primero que cumple | `find(p)` | `filter(p).findFirst()` | `member(X, L), p(X), !` |
| recoger resultados | (ya es arreglo) | `.toList()` / `collect(...)` | `findall(X, G, L)` |

### Cómo llama cada combinador de Prolog a tu predicado

| Combinador | Lo llama como | Tu predicado necesita |
|---|---|---|
| `include(P, L, L2)` | `P(X)` | 1 argumento: éxito si `X` se queda |
| `maplist(P, L, L2)` | `P(X, Y)` | 2 argumentos: `X` entra, `Y` sale |
| `foldl(P, L, Ini, R)` | `P(X, Acc0, Acc)` | 3 argumentos: elemento, acumulador viejo, acumulador nuevo |

### Trampas

**1. El orden de `reduce` cambia.**
- JS: `reduce((acc, x) => …, ini)`: inicial al final; acumulador primero.
- Java: `reduce(ini, (acc, x) -> …)`: inicial primero.
- Prolog: `foldl(P, L, Ini, R)` con `P(X, Acc0, Acc)`: el elemento va primero.

**2. Stream de Java es lazy; Array de JS es eager.**
```java
Stream.of(1, 2, 3)
      .map(x -> { System.out.println("map " + x); return x * 2; });
System.out.println("fin");
// Imprime SOLO: fin
```
`map` y `filter` son operaciones **intermedias**: no hacen nada hasta que hay una **terminal** (`forEach`, `reduce`, `toList`, `count`, `findFirst`). Con `.toList()` al final sí imprime `map 1, map 2, map 3, fin`. En JS, el mismo código con un Array imprime los `map` de inmediato.

> **Analogía:** armaste la línea de producción pero nadie apretó el botón de encendido. Es lo mismo que "crear no es ejecutar" de los generadores.

| | ¿Cuándo trabaja `map`? |
|---|---|
| JS `Array` | de inmediato (eager) |
| Java `Stream` | solo con operación terminal (lazy) |
| JS generador | solo cuando piden `next()` (lazy) |

**3. En Prolog un predicado no devuelve verdadero o falso: tiene éxito o falla.**
`N > 4` basta; no se escribe `(N > 4 -> TRUE ; FALSE)`. Además `TRUE` y `FALSE` en mayúscula son **variables**; los valores son `true` y `fail`.

**4. `length` en JS es propiedad, sin paréntesis:** `x.length`. Y `[x].length` es 1 (un arreglo con un elemento), no la longitud de `x`. En Java: `s.length()` para String, `arr.length` para arreglos, `lista.size()` para listas.

---

## 4. `call` y `apply`

Sirven para llamar una función o predicado que está guardado en una variable.

| | JS | Prolog |
|---|---|---|
| argumentos sueltos | `f.call(thisArg, a, b)` | `call(P, A, B)` |
| argumentos en lista | `f.apply(thisArg, [a, b])` | `apply(P, [A, B])` |

> **Regla:** **a**pply = **a**rreglo.

**Ejemplo 1 (JS):**
```javascript
const suma = (a, b) => a + b
suma.call(null, 2, 3)       // 5   (el primer argumento es el this)
suma.apply(null, [2, 3])    // 5
```
**Ejemplo 2 (Prolog):**
```prolog
?- call(atom_length, hola, N).      % igual que atom_length(hola, N)   → N = 4
?- call(plus(1), 3, Y).             % los argumentos nuevos van AL FINAL: plus(1, 3, Y) → Y = 4
?- apply(atom_length, [hola, N]).   % N = 4
```
Así funciona `star`: `call(P, T, I, O1)` le agrega tres argumentos a `P`.

---

## 5. Recursión de cola

Una función es **recursiva de cola** cuando la llamada recursiva es **lo último que hace**: al regresar, no queda **nada pendiente** con su resultado.

> **Analogía:** sumar facturas pasándolas de persona en persona. **Normal:** cada uno espera el total del de adelante y después le suma la suya. **De cola:** cada uno suma la suya al total que ya viene anotado y pasa el papel; el último ya tiene la respuesta.

### Cómo reconocerla: el truco del dedo
Tapa la llamada recursiva y mira qué queda en esa rama. Si queda solo el cuadrito, es de cola.

| Código | Con la llamada tapada | ¿Cola? |
|---|---|---|
| `1 + largo(lista.slice(1))` | `1 + ▢` | ❌ |
| `n * fact(n - 1)` | `n * ▢` | ❌ |
| `cuenta(n - 1)` | `▢` | ✅ |
| `mcd(b, a % b)` | `▢` | ✅ |

⚠️ El criterio **no** es "tiene acumulador": `cuenta` y `mcd` son de cola sin acumulador. Los argumentos (`n - 1`, `a % b`) se calculan **antes** de llamar, así que no cuentan como pendientes.

En **Prolog**: es de cola si la llamada recursiva es la **última meta** del cuerpo.

### Receta para convertir
1. Agregar un parámetro **acumulador**, con el valor que devolvía el caso base.
2. El caso base **devuelve el acumulador**.
3. La operación pendiente se mete **dentro del acumulador**, en la llamada.

### Ejemplo 1: factorial
```javascript
const fact  = n => n === 0 ? 1 : n * fact(n - 1)                          // normal
const factC = (n, acc = 1) => n === 0 ? acc : factC(n - 1, n * acc)       // de cola
```
```java
static int factC(int n, int acc) { return n == 0 ? acc : factC(n - 1, n * acc); }   // factC(5, 1)
```
```prolog
fact(N, R) :- fact(N, 1, R).                 % envoltorio: arranca el acumulador
fact(0, Acc, Acc) :- !.
fact(N, Acc, R) :- N1 is N - 1, Acc1 is N * Acc, fact(N1, Acc1, R).
```
Traza: `factC(3, 1)` → `factC(2, 3)` → `factC(1, 6)` → `factC(0, 6)` → `6`.

### Ejemplo 2: suma de una lista
```prolog
% Normal: queda un "is" pendiente después de la llamada
suma([], 0).
suma([F | R], S) :- suma(R, S1), S is F + S1.

% De cola
suma(L, S) :- suma(L, 0, S).
suma([], Acc, Acc).
suma([F | R], Acc, S) :- Acc1 is Acc + F, suma(R, Acc1, S).
```
⚠️ **No sirve solo cambiar el orden** (`S is F + S1, suma(R, S1)`): `S1` todavía está vacía cuando corre el `is` y da error. Hay que sumar al total que **ya se lleva** (`Acc`), no al que viene del futuro.

```javascript
const suma  = lista => lista.length === 0 ? 0 : lista[0] + suma(lista.slice(1))                 // normal
const sumaC = (lista, acc = 0) => lista.length === 0 ? acc : sumaC(lista.slice(1), acc + lista[0])   // de cola
```
```java
static int suma(List<Integer> lista, int acc) {
    return lista.isEmpty() ? acc : suma(lista.subList(1, lista.size()), acc + lista.get(0));
}
```

| | Normal | De cola |
|---|---|---|
| ¿Cuándo hace la cuenta? | al **regresar** | **antes** de llamar |
| Caso base | valor fijo (`0`, `1`, `[]`) | el acumulador |
| Pila | crece con cada llamada | puede reusarse (como un ciclo) |

**¿Para qué?** Como no queda nada pendiente, el lenguaje puede reusar el mismo espacio de pila. Prolog sí hace esa optimización; en Java y en la mayoría de motores de JS no está garantizada. `foldl` y `reduce` son esta misma idea ya empaquetada.

---

## 6. Resumen rápido

| Tema | Recordar |
|---|---|
| DCG | `-->` esconde `I` y `O`; `[C]` come; `{}` no come; `phrase` exige que no sobre nada |
| `star` / `plus` | cero o más / una o más; `plus` = una + `star`; sin segunda cláusula |
| `map` / `filter` / `reduce` | `maplist` / `include` / `foldl` en Prolog |
| Argumentos en Prolog | `include` → 1, `maplist` → 2, `foldl` → 3 (elemento, viejo, nuevo) |
| Stream | lazy: sin operación terminal no corre nada |
| `call` / `apply` | sueltos / en lista (**a**pply = **a**rreglo) |
| Recursión de cola | la llamada es lo último; truco del dedo; receta del acumulador |

**Errores clásicos:** `plus` que se llama a sí mismo · `star` con un parser que no consume · `[x].lenght()` en vez de `x.length` · `TRUE`/`FALSE` en mayúscula en Prolog · creer que un Stream sin terminal ejecuta el `map` · decir que no es de cola porque "no tiene acumulador" · cambiar el orden de las metas en vez de usar acumulador.
