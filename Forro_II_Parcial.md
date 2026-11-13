# Forro · II Parcial EIF400 (13/11, 10:00 am)

Solo reglas, tablas y plantillas genéricas. **Sin ejercicios ni ejemplos resueltos.**
⭐ = énfasis del profesor: si el espacio no alcanza, copiar primero estas secciones.

---

## ⭐ 1. JS: ejecución a mano (pila y colas)

**Un solo hilo.** Orden del event loop:
1. Todo el código **síncrono** (hasta vaciar la pila).
2. **Todas** las microtareas: `.then`, `.catch`, lo que sigue a un `await`.
3. **Una** tarea: callback de `setTimeout` / `setInterval`. Volver al paso 2.

| | |
|---|---|
| `setTimeout(f, ms)` | `f` una vez, después de `ms`. No pausa el programa. Con `0` igual va después de lo síncrono y de las promesas |
| `setInterval(f, ms)` | `f` cada `ms`; devuelve `id` |
| `clearTimeout(id)` / `clearInterval(id)` | cancelan |
| Promesa | pending → fulfilled / rejected (cambia una sola vez) |
| `new Promise((resolve, reject) => …)` | el cuerpo corre **síncrono**, de inmediato |
| `.then(f)` | nunca corre de inmediato; devuelve otra promesa |
| rechazo / `throw` | se salta los `then` hasta el `catch`; después del `catch` la cadena **sigue** |
| `async function` | **siempre** devuelve una promesa |
| `await p` | pausa **esa función**, no el programa; lo que sigue es microtarea |
| antes del primer `await` | corre síncrono |

```javascript
const esperar = ms => new Promise(resolve => setTimeout(resolve, ms))
async function f() { try { const v = await p } catch (e) { } }
```

## ⭐ 2. Lazy vs eager, lambdas, clausuras

| | |
|---|---|
| **Eager** | evalúa los argumentos **antes** de llamar (JS, Java) |
| **Lazy** | evalúa solo **cuando se necesita** (generadores, Stream, `&&`, `\|\|`, `? :`) |
| **Lambda** | función anónima: `x => …` (JS), `x -> …` (Java), `[X]>>(…)` (Prolog) |
| **Clausura** | función + las variables de donde nació (la "mochila"); las recuerda aunque la función externa ya terminó |
| **Currying** | `f(a, b)` → `a => b => …`, se llama `f(a)(b)` |
| **Aplicación parcial** | dar solo algunos argumentos y recibir una función que espera el resto |
| **Variable ligada / libre** | ligada: tiene un `λ` que la declara; libre: no |
| **β-reducción** | `(λx. M) N → M[x := N]`: quitar el `λx` y reemplazar **todas** las `x` ligadas a ese `λ` por `N` |
| **Redex / forma normal** | redex: aplicación reducible; forma normal: ya no quedan redex |

## ⭐ 3. Generadores (JS)

```javascript
function* g(params) { while (cond) { yield valor } }   // crear NO ejecuta
const it = g()
it.next()                        // { value, done }   ← usar .value
for (const x of g()) { }         // llama a next() solo
async function* ag() { yield await p }
for await (const x of ag()) { }
```
`yield` = entrega y **pausa**; sigue desde ahí en el próximo `next()`. Infinito: `while (true)` + `yield`.
Azar entre `0` y `L-1`: `Math.floor(Math.random() * L)`.

## ⭐ 4. Combinadores

| Qué hace | JS (Array) | Java (Stream) | Prolog (listas) |
|---|---|---|---|
| transformar cada uno | `map(f)` | `map(f)` | `maplist(P, L, L2)` |
| quedarse con los que cumplen | `filter(p)` | `filter(p)` | `include(P, L, L2)` |
| quitar los que cumplen | `filter(x => !p(x))` | `filter(x -> !p(x))` | `exclude(P, L, L2)` |
| juntar en un valor | `reduce(f, ini)` | `reduce(ini, f)` | `foldl(P, L, Ini, R)` |
| hacer algo con cada uno | `forEach(f)` | `forEach(f)` | `forall(member(X, L), G)` |
| ¿alguno cumple? | `some(p)` | `anyMatch(p)` | `member(X, L), p(X)` |
| ¿todos cumplen? | `every(p)` | `allMatch(p)` | `forall(member(X, L), p(X))` |
| el primero que cumple | `find(p)` | `filter(p).findFirst()` | `member(X, L), p(X), !` |
| aplanar | `flatMap(f)` | `flatMap(f)` | `append(ListaDeListas, L)` |
| recoger resultados | (ya es arreglo) | `.toList()` | `findall(X, G, L)` |

| | |
|---|---|
| `reduce` JS | `reduce((acc, x) => …, ini)` |
| `reduce` Java | `reduce(ini, (acc, x) -> …)` |
| `foldl` Prolog | llama `P(X, Acc0, Acc)`: elemento, viejo, nuevo |
| Argumentos del predicado | `include` → 1 · `maplist` → 2 (entra, sale) · `foldl` → 3 |
| `call` | JS `f.call(this, a, b)` · Prolog `call(P, A, B)` (agrega los argumentos **al final**) |
| `apply` (**a**rreglo) | JS `f.apply(this, [a, b])` · Prolog `apply(P, [A, B])` |
| Array JS | **eager**: cada `map` recorre todo ya |
| Stream Java | **lazy**: intermedias (`map`, `filter`, `limit`) no corren sin una **terminal** (`forEach`, `reduce`, `toList`, `count`, `findFirst`) |
| Longitud | JS `x.length` · Java `s.length()`, `arr.length`, `lista.size()` · Prolog `length(L, N)`, `atom_length(A, N)` |

## ⭐ 5. Recursión de cola

**De cola:** la llamada recursiva es **lo último**; al regresar no queda nada pendiente. En Prolog: es la **última meta**.
**Truco del dedo:** tapar la llamada; si queda solo `▢`, es de cola (`1 + ▢` no lo es). No depende de tener acumulador.

**Receta pila → cola:**
1. Agregar un **acumulador** que arranca con lo que devolvía el caso base.
2. El caso base **devuelve el acumulador**.
3. La operación pendiente se mete **dentro del acumulador**, en la llamada.

```javascript
// pila:  f = n => base(n) ? INI : OP(n, f(sig(n)))
const f = (n, acc = INI) => base(n) ? acc : f(sig(n), OP(n, acc))
```
```java
static T f(N n, T acc) { return base(n) ? acc : f(sig(n), OP(n, acc)); }
```
```prolog
f(X, R) :- f(X, INI, R).                         % envoltorio
f(Base, Acc, Acc).
f(X, Acc, R) :- Acc1 is OP(X, Acc), siguiente(X, X1), f(X1, Acc1, R).
```
Pila: la cuenta se hace **al regresar**. Cola: **antes de llamar**. Ventaja: reusa la pila, como un ciclo.

## ⭐ 6. Mónada

**Mónada = caja + unit + bind.** Modela **estado sin mutación**: el estado viaja en la caja y el bind lo propaga.

| | |
|---|---|
| **unit** | mete un valor en la caja: `Promise.resolve(x)`, `Optional.of(x)`, `Stream.of(x)`, `[x]` |
| **bind** | aplica una función **que devuelve caja** y **aplana**: `.then(f)`, `.flatMap(f)` |
| `map` vs `flatMap` | `map` envuelve (puede dar caja en caja); `flatMap` aplana |

| Mónada | Estado que modela | El bind… |
|---|---|---|
| `Promise` | pending / fulfilled / rejected | espera; si hay rechazo, salta al `catch` |
| `Optional` | hay valor / vacío | si está vacío, se salta lo que sigue |
| `Stream`, `Array` | cero, uno o muchos | aplica a cada uno y junta |

**Promise como mónada:** caja = la promesa · unit = `Promise.resolve` · bind = `then` (nunca hay promesa de promesa) · `await` = `then` escondido.

## ⭐ 7. Generadores vs Stream vs backtracking

Los tres dan resultados **de a uno, bajo demanda**: separan **producir** de **consumir**.

| | Generador (JS) | Stream (Java) | Backtracking (Prolog) |
|---|---|---|---|
| Se define con | `function*` + `yield` | `Stream.of`, `iterate`, `.stream()` | varias cláusulas / hechos |
| Pide el siguiente | `next()`, `for...of` | la operación terminal | `;`, `fail`, `findall` |
| Infinito | `while (true)` | `iterate` + `limit` | `between(1, inf, X)` |
| Recoger todo | `[...g]` | `.toList()` | `findall` |
| Cortar | `break`, `return` | `limit`, `findFirst` | `!`, `once` |

## ⭐ 8. DCG

| En la DCG | Significa |
|---|---|
| `a --> b, c.` | `a(I, O) :- b(I, M), c(M, O).` (dos argumentos escondidos: entra, sobra) |
| `[C]` | consume **un** código |
| `"txt"` | consume esos códigos |
| `{ … }` | Prolog normal; **no consume** |
| `[]` | no consume nada |
| `!` | no probar otra cláusula |
| `phrase(P, Texto)` | no debe sobrar nada |
| `phrase(P, Texto, Resto)` | devuelve lo que sobró |

Texto: `` `abc` `` = lista de códigos · `0'a` = código de `a` · `number_codes(N, Cs)` · `atom_codes(A, Cs)` · `code_type(C, digit)` / `space` / `alpha`.

```prolog
% cero o más                               % una o más
xs([X | R]) --> x(X), !, xs(R).            xs1([X | R]) --> x(X), xs(R).
xs([]) --> [].

% un símbolo que cumple una condición
x(C) --> [C], { condicion(C) }.

% operadores asociando a la izquierda (acumulador)
expr(E) --> term(T), resto(T, E).
resto(Acc, E) --> "op", !, term(T), resto(nodo(Acc, T), E).
resto(E, E) --> [].
```
Precedencia: lo que amarra más fuerte va **más abajo** (`expr` → `term` → `factor`).
Combinadores `star(L, P)` = cero o más, `plus(L, P)` = una o más (una + `star`); usan `call(P, T, I, O1)`. `P` debe **consumir** algo o hay ciclo infinito. El `!` los hace glotones.

## 9. Prolog: lo esencial

| | |
|---|---|
| Variables / átomos | `Mayúscula` o `_` / `minúscula` |
| Lista | `[F \| R]` cabeza y resto; `[]` vacía |
| `=` | unifica (no calcula) |
| `\=` | no unifican |
| `==` | idénticos, sin unificar |
| `is` | calcula el lado derecho: `X is 2 + 3` |
| `=:=`, `=\=`, `<`, `>`, `=<`, `>=` | calculan ambos lados y comparan |
| `=..` | `T =.. [Functor \| Args]` |
| `,` / `;` / `->` | y / o / si-entonces: `(C -> A ; B)` |
| `!` | no retroceder: descarta las otras cláusulas y las opciones anteriores de esa regla |
| `\+ G` | éxito si `G` falla (negación por falla); no amarra variables |
| Éxito / falla | no hay `return true/false`: `true`, `fail` |
| `findall(T, G, L)` | todas las soluciones (da `[]` si no hay) |
| `forall(C, A)` | para toda solución de `C`, se cumple `A` |
| `between(A, B, X)`, `member(X, L)`, `append(A, B, C)`, `length(L, N)`, `nth0`/`nth1`, `reverse`, `msort` | utilitarios |
| `assertz(H)`, `asserta(H)`, `retract(H)`, `:- dynamic p/1.` | modificar la base de hechos |
| Lambda (yall) | `[X, Y]>>(Y is X * 2)` |
| Dict | `_{k: v}`, `get_dict(K, D, V)`, `put_dict(K, D, V, D2)` |
| `catch(G, E, Manejo)` / `throw(E)` | excepciones |

**Resolución:** cláusulas de **arriba hacia abajo**, metas de **izquierda a derecha**, búsqueda en **profundidad (DFS)**; al fallar retrocede al último punto de elección.
**Recursión vs backtracking:** recursión = un predicado se llama a sí mismo (baja en la estructura); backtracking = volver a probar otra alternativa.

## 10. Plantillas

**Autómata finito** (estado = lo que ya leí; cada flecha consume un símbolo; acepta si termina en un final; **no puede contar**: `aⁿbⁿ` no)
```prolog
inicial(Q0).   final(QF).   delta(Estado, Simbolo, NuevoEstado).
acepta(T) :- inicial(Q), recorre(Q, T).
recorre(Q, [])      :- final(Q).
recorre(Q, [C | R]) :- delta(Q, C, Q1), recorre(Q1, R).
```

**Visitor de AST** (una cláusula por tipo de nodo)
```prolog
visita(hoja(X), Resp).                                             % caso base
visita(nodo(A, B), R) :- visita(A, RA), visita(B, RB), combinar(RA, RB, R).
```
| | Devuelve | Combina con |
|---|---|---|
| Evaluador | un valor | `is` |
| Analizador | información (conteo, lista) sin ejecutar | suma, `append` |
| Generador | otro programa (instrucciones) | código izq. + código der. + instrucción del operador |

**Generar y probar**
```prolog
solucion(X, Y) :- generar(X), generar(Y), probar(X, Y).
?- findall(X-Y, solucion(X, Y), L).
```
Probar **apenas** las variables tengan valor; **nunca** antes de generar.

## 11. Conceptos

| | |
|---|---|
| **Declarativo** | se dice **qué** se quiere, no **cómo** (Prolog, FP, SQL, Cypher) |
| **Homoicónico** | código y datos con la **misma forma**; un programa manipula programas (Prolog: términos, `=..`, `call`, `assert`). JS y Java no |
| **Tipado estático** | los tipos se revisan **antes** de ejecutar (Java, TypeScript) |
| **Tipado dinámico** | los tipos se revisan **al ejecutar** (JS, Prolog) |
| **Compilación** | traducir a un lenguaje de más bajo nivel (Java → bytecode) |
| **Transpilación** | traducir a otro lenguaje de nivel similar (TypeScript → JS) |
| **Máquina virtual** | programa que ejecuta código intermedio (JVM; la VM de SWI-Prolog) |
| **Motor (engine)** | programa que ejecuta el lenguaje (V8 para JS) |
| **Función pura** | mismo resultado para los mismos argumentos, sin efectos secundarios |
| **Orden superior** | función que recibe o devuelve funciones |
| **Compilar una consulta a Prolog** | parser (DCG) → AST → visitor generador → meta de Prolog → `call` |
| **MQI** | *Machine Query Interface* de SWI-Prolog: otro lenguaje (p. ej. Python) lanza a Prolog como proceso, le manda consultas por un socket y recibe las respuestas en JSON |

## 12. Errores clásicos (revisar antes de entregar)

- JS: `n + 2` sin asignar · `yield i` en vez de `valores[i]` · olvidar `.value` · comparar una promesa con un valor · creer que `setTimeout(f, 0)` corre ya · `.length()` con paréntesis.
- Java: Stream sin operación terminal no ejecuta nada.
- β: no quitar el `λ` · reemplazar solo la primera aparición.
- Prolog: átomos en mayúscula · `=` en vez de `is` · unir casos alternativos con coma · argumentos al revés · probar antes de generar · cambiar el orden de las metas en vez de usar acumulador.
- DCG: `plus` que se llama a sí mismo · repetir un parser que no consume.
- Autómata: flecha vacía para lo opcional · olvidar que terminar en estado no final rechaza.
- **Toda respuesta se justifica.**
