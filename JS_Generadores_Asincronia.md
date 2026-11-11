# JS: Generadores y Asincronía

**EIF400 Paradigmas de Programación · Repaso para el II Parcial (viernes 13/11, 10:00 am)**
Temas del SPEC: iteradores y generadores, objetos infinitos (lazy), event loop, timers (`setTimeout`, `setInterval`), `Promise`, `async/await`. Ejercicios A2 y A3 de la sesión 32.

---

## 1. Iteradores y generadores

### El problema
No se puede guardar una lista **infinita** en un arreglo. Solución: **dar los valores uno por uno, solo cuando se piden** (evaluación **lazy**).

> **Analogía:** el dispensador de turnos del banco: imprime el siguiente tiquete solo cuando aprietas el botón.

### Iterador
Objeto con un método `next()` que devuelve `{ value, done }`:
```javascript
{ value: 5, done: false }           // aquí está el 5, y hay más
{ value: undefined, done: true }    // ya no hay más
```

### Generador: `function*` + `yield`
```javascript
function* tresNumeros() {
    yield 1
    yield 2
    yield 3
}
const gen = tresNumeros()   // ⚠️ NO ejecuta nada: solo crea el iterador
gen.next()                  // { value: 1, done: false }
gen.next()                  // { value: 2, done: false }
gen.next()                  // { value: 3, done: false }
gen.next()                  // { value: undefined, done: true }

for (const x of tresNumeros()) console.log(x)    // 1, 2, 3  (for...of llama a next() por ti)
```
`yield` = "**entrega este valor y PAUSA aquí**". La función se congela y sigue desde ahí en el próximo `next()`.

> **Analogía:** un libro con **marcapáginas**.

### Crear no es ejecutar (pregunta típica de orden)
```javascript
function* letras() {
    console.log("empiezo")
    yield "a"
    console.log("sigo")
    yield "b"
}
const g = letras()
console.log("creado")
console.log(g.next().value)
console.log(g.next().value)
// Salida: creado, empiezo, a, sigo, b
```
"creado" sale antes que "empiezo": el cuerpo no corre hasta el primer `next()`, y cada `next()` corre solo hasta el siguiente `yield`.

### Generadores infinitos (objetos infinitos)
```javascript
function* naturales() {
    let n = 0
    while (true) {      // no se cuelga: el yield pausa en cada vuelta
        yield n
        n++
    }
}
function* pares() {
    let n = 0
    while (true) {
        yield n
        n += 2          // ⚠️ "n + 2" solo calcula y bota el resultado: daría 0, 0, 0…
    }
}
```
⚠️ `for (const x of naturales())` sin `break` sí es infinito.

### Con límite
```javascript
function* naturales(max = Infinity) {       // valor por defecto
    let n = 0, entregados = 0
    while (entregados < max) {
        yield n
        n++
        entregados++
    }
}
```

### Ejercicio A3 (parte 1): `random_select`
```javascript
function* random_select(values, max = Infinity) {
    let entregados = 0
    while (entregados < max) {
        const i = Math.floor(Math.random() * values.length)   // posición al azar: 0 … length-1
        yield values[i]                                       // el VALOR, no la posición
        entregados++
    }
}
```
**La ruleta:** `Math.random()` da de 0 a 0.999… → `* L` da de 0 a L−0.001 → `Math.floor` da 0, 1, …, L−1.
⚠️ Errores típicos: `yield i` (entrega la posición); `* (values.length - 1)` (el último nunca sale).

| Cálculo λ | Generadores |
|---|---|
| lazy: calcular solo lo necesario | cada valor se calcula al pedirlo con `next()` |
| Ω se cuelga con eager | `while (true)` normal se cuelga; con `yield`, no |

---

## 2. Event loop y timers

**JS tiene un solo hilo:** hace una cosa a la vez.

| Pieza | Qué es | Analogía |
|---|---|---|
| **Call stack** (pila) | lo que se ejecuta ahora | el único mesero |
| **Timers / Web APIs** | esperas fuera de JS | la cocina con cronómetros |
| **Cola de microtareas** | callbacks de promesas (`.then`, lo que sigue a un `await`) | 🥇 fila VIP de la barra |
| **Cola de tareas** | callbacks de `setTimeout` / `setInterval` | 🥈 fila normal de la barra |

**Regla del event loop:**
1. Ejecuta **todo el código síncrono** (hasta vaciar la pila).
2. Atiende **todas las microtareas** (promesas).
3. Atiende **una** tarea (timer) y vuelve al paso 2.

### Timers
```javascript
setTimeout(f, ms)                  // ejecuta f UNA vez, después de ms. NO pausa el programa
const id = setInterval(f, ms)      // ejecuta f cada ms
clearInterval(id)                  // lo detiene (clearTimeout cancela un setTimeout)
```
> `setTimeout` = alarma de una vez; `setInterval` = alarma que se repite.

### Orden de ejecución (preguntas típicas)
```javascript
console.log("1")
setTimeout(() => console.log("2"), 1000)
setTimeout(() => console.log("3"), 0)
console.log("4")
// 1, 4, 3, 2
```
`1` y `4` son síncronos (van primero). Los `setTimeout` solo registran callbacks; el de 0 ms entra antes a la cola que el de 1000 ms. El event loop los pasa a la pila solo cuando está vacía.
⚠️ **`setTimeout(f, 0)` igual se ejecuta después de todo el código síncrono.**

```javascript
console.log("A")
setTimeout(() => console.log("B"), 0)
Promise.resolve().then(() => console.log("C"))
console.log("D")
// A, D, C, B      ← la promesa (microtarea) va antes que el timer
```

---

## 3. Promesas

> **Promesa:** objeto que representa un valor que todavía no está listo, pero lo estará (o fallará).

> **Analogía:** el **beeper** del restaurante: no es la hamburguesa, es la promesa de una.

| Estado | Significado |
|---|---|
| **pending** | todavía no hay resultado |
| **fulfilled** | terminó bien y tiene un valor |
| **rejected** | falló y tiene un error |

Cambia de estado **una sola vez**.

```javascript
const p = new Promise((resolve, reject) => {
    // trabajo que tarda…   resolve(valor)  ó  reject(error)
})

const esperar = ms => new Promise(resolve => setTimeout(resolve, ms))   // "espera ms milisegundos"

p.then(valor => …)      // si se cumplió
 .catch(error => …)     // si se rechazó
 .finally(() => …)      // siempre

Promise.resolve(5).then(x => x * 2).then(x => x + 1).then(console.log)   // 11 (cada then devuelve otra promesa)
```
⚠️ Un `.then` **nunca** corre de inmediato: siempre después del código síncrono.

---

## 4. `async` / `await`

| | Qué hace |
|---|---|
| `async function` | **siempre devuelve una promesa** (aunque haga `return 5`) |
| `await promesa` | pausa **esa función** (no el programa) hasta que la promesa termine, y da su valor |
| lo que sigue al `await` | corre después del código síncrono (microtarea) |
| `try { await … } catch (e) { … }` | atrapa promesas rechazadas |

`await` solo se usa dentro de una función `async` (o en el nivel superior de un módulo).

```javascript
async function f() {
    console.log(1)       // síncrono
    await null           // pausa: lo que sigue va a la fila VIP
    console.log(2)
}
f()
console.log(3)
// 1, 3, 2
```
> `await` = sentarte a esperar tu beeper; el restaurante sigue funcionando.

### Ejercicio A2: el `succesor` de Carlitos
```javascript
async function succesor(n){
    return await ++n
}
console.log("C1)=", foo(0) == 1)    // Expected true. Returned false
```
**Respuesta:** `succesor` es `async`, así que **devuelve una promesa**, no un número. `succesor(0) == 1` compara una promesa con `1` → siempre `false` (comparó el beeper con la hamburguesa). Aunque se quite el `await` de adentro, sigue fallando: la causa es `async`. Además, el enunciado define `succesor` pero llama a `foo`.

Soluciones:
```javascript
console.log("C1)=", await succesor(0) == 1)                 // 1) esperar afuera
succesor(0).then(r => console.log("C1)=", r == 1))          // 2) con then
function succesor(n) { return ++n }                         // 3) la mejor: quitar async (sumar 1 no es asincrónico)
```
El `await ++n` de adentro es inútil: `++n` no es una promesa.

---

## 5. Generadores asincrónicos

Unión de generadores (valores de a uno) y promesas (valores que tardan).
> **Analogía:** streaming: los capítulos llegan uno por uno y cada uno tarda en cargar.

| | Síncrono | Asincrónico |
|---|---|---|
| Un valor | `function` → valor | `async function` → promesa |
| Muchos valores | `function*` → `for...of` | `async function*` → `for await...of` |

### `generators_async.js` del profe
```javascript
async function fakeAPIFetch(page){                       // simula un servidor que tarda
    const delay = 500;
    const fakePage = [ { id: `id-${page}_1`, name: `User A${page}` },
                       { id: `id-${page}_2`, name: `User B${page}` } ]
    await new Promise(resolve => setTimeout(resolve, delay))     // espera delay ms
    return fakePage
}

async function* pagesGenerator(totalPages) {             // generador asincrónico
  for (let page = 1; page <= totalPages; page++) {
    yield await fakeAPIFetch(page)                       // espera la página y la entrega
  }
}

for await (const page of pagesGenerator(3)) {            // una por una, apenas llega
    page.forEach(user => console.log(user.name))
}

const names = await Array.fromAsync(pagesGenerator(3),   // todas juntas
                                    page => page.map(u => u.name))
names.flat()                                             // [[A1,B1],[A2,B2]] → [A1,B1,A2,B2]
```
| | `for await` | `Array.fromAsync` |
|---|---|---|
| Procesa | apenas llega cada página | cuando llegaron todas |
| En Prolog | `forall` | `findall` |

### Ejercicio A3 (parte 2): delay al azar
```javascript
const delays = random_select([500, 1000, 1500, 2000])     // generador infinito, creado UNA vez

async function fakeAPIFetch(page){
    const delay = delays.next().value                     // ⚠️ .value: next() devuelve { value, done }
    const fakePage = [ { id: `id-${page}_1`, name: `User A${page}` },
                       { id: `id-${page}_2`, name: `User B${page}` } ]
    await new Promise(resolve => setTimeout(resolve, delay))
    return fakePage
}
```
⚠️ Sin `.value`, `delay` sería el objeto `{ value: 1000, done: false }` y `setTimeout` no esperaría el tiempo correcto.

---

## 6. Resumen rápido

| Tema | Recordar |
|---|---|
| Iterador | `next()` → `{ value, done }` |
| Generador | `function*` + `yield` (marcapáginas); crear no es ejecutar |
| Infinito / lazy | `while (true)` + `yield`; se calcula al pedir |
| Un solo hilo | síncrono → microtareas (promesas) → tareas (timers) |
| `setTimeout(f, 0)` | después de todo lo síncrono y de las promesas |
| Promesa | pending → fulfilled / rejected; `.then` / `.catch` |
| `async` | siempre devuelve promesa |
| `await` | pausa la función, no el programa; saca el valor |
| Async generator | `async function*`, `yield await`, `for await` |

**Errores clásicos:** `n + 2` sin asignar · `yield i` en vez de `values[i]` · `length - 1` en la ruleta · comparar una promesa con un valor (`succesor(0) == 1`) · olvidar `.value` después de `next()` · creer que `setTimeout(f, 0)` corre de inmediato.
