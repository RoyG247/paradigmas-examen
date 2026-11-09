# Cálculo Lambda — Parte 1

**EIF400 Paradigmas de Programación · Repaso para el II Parcial (viernes 13/11, 10:00 am)**
Temas del SPEC: Parte 2, secciones 2a–2c (historia, sintaxis, reducciones α/β/η) y 3c, 3h (lambdas, clausuras, currying y aplicación parcial).

---

## 1. ¿Qué es el cálculo lambda?

Es un **lenguaje de programación mínimo** creado por **Alonzo Church** en los años 30 para responder "¿qué significa calcular?". En él **solo existen funciones**: no hay números, ni `if`, ni ciclos, ni variables que cambien. Aun así, se puede calcular **cualquier cosa** que calcula una computadora (tesis de Church-Turing). Es la base teórica de la programación funcional: de aquí salen LISP, Haskell y las arrow functions de JS.

> **Analogía:** un LEGO con **una sola pieza**. Con ingenio se construye cualquier cosa.

---

## 2. Las 3 piezas

| Pieza | Cálculo λ | JS | Analogía |
|---|---|---|---|
| **Variable** | `x` | `x` | un nombre |
| **Abstracción** (función) | `λx. M` | `x => M` | 📝 escribir una receta (no la cocina) |
| **Aplicación** (llamar) | `M N` | `M(N)` | 🍳 cocinar la receta con un ingrediente |

En `λx. M`: `λ` = "aquí empieza una función", `x` = parámetro, `.` separa, `M` = cuerpo.

### Reglas de lectura

1. **La aplicación asocia a la izquierda:** `f a b` = `(f a) b` → en JS `f(a)(b)`.
2. **El cuerpo de una λ llega hasta el final:** `λx. f x` = `λx. (f x)` → `x => f(x)`.
3. **Varias λ se juntan:** `λx y. M` = `λx. λy. M` → `x => y => M`.

### Práctica (resuelta)

| Cálculo λ | JS |
|---|---|
| `λx. x` | `x => x` |
| `λx. λy. y` | `x => y => y` |
| `(λx. x) z` | `(x => x)(z)` |
| `f a b` | `f(a)(b)` ⚠️ (no `(f a)b`, que sigue siendo notación λ) |
| `λf. λx. f x` | `f => x => f(x)` |
| `λa. λb. a b` | `a => b => a(b)` |

---

## 3. Currying, aplicación parcial y clausuras

### Currying
En el cálculo λ **cada función recibe un solo argumento**. Para dos argumentos, una función devuelve otra función.

```javascript
const sumar  = (x, y) => x + y      // uncurried: sumar(2, 3)
const sumarC = x => y => x + y      // curried:   sumarC(2)(3)
```
> **Analogía:** una máquina expendedora que pide **una moneda a la vez**. Con la primera queda "esperando" la segunda.

Nombre: viene de **Haskell Curry**.

### Aplicación parcial
Dar solo los primeros argumentos y guardar la función que queda, "pre-configurada".

```javascript
const sumar10 = sumarC(10)      // devuelve y => 10 + y
sumar10(5)                      // 15
[1, 2, 3].map(sumarC(10))       // [11, 12, 13]
```
> **Analogía:** la máquina con la primera moneda ya puesta. (En Prolog: `maplist(dist(P), LP, Ds)`.)

### Clausura (closure)
> **Una función junto con las variables de afuera que recuerda** del lugar donde fue creada, aunque ese lugar ya haya terminado.

Normalmente, cuando una función termina, sus variables locales se borran (se limpia la "oficina"). Pero si devolvió una función que **usa** esas variables, la función se las **lleva consigo**.

> **Analogía: la mochila 🎒.** La función interna mete en su mochila las variables de afuera que necesita y las usa cada vez que se le llama.

```javascript
const contador = inicio => paso => inicio + paso
const desde100 = contador(100)      // mochila: inicio = 100
desde100(5)                         // 105
desde100(20)                        // 120
```
**Respuesta tipo examen:** `contador(100)` devuelve `paso => inicio + paso`, que forma una **clausura**: conserva `inicio = 100` aunque `contador` ya terminó. `inicio` es una variable **libre** de la función interna y su valor queda capturado. Por eso da 105 y 120.

> **Clausura = función + los valores de sus variables libres.**

La misma función puede tener mochilas distintas:
```javascript
const saludador = saludo => nombre => saludo + ", " + nombre
const hola   = saludador("Hola")     // mochila: "Hola"
const buenas = saludador("Buenas")   // mochila: "Buenas"
buenas("Roy")                        // "Buenas, Roy"
// En nombre => saludo + ", " + nombre:  nombre = atada, saludo = libre
```

### Ejercicio (sesión 32, slide 15): `Math.pow` curried
```javascript
Math.pow(2, 3)                                 // original (uncurried): 8

const powC = base => exponente => Math.pow(base, exponente)     // con Math.pow

const potenciaC = base => exponente =>                          // recursiva
    exponente === 0 ? 1 : base * potenciaC(base)(exponente - 1)

const potenciaDe2 = potenciaC(2)      // aplicación parcial + clausura (base = 2)
potenciaDe2(3)                        // 8
potenciaDe2(10)                       // 1024
```
Rastreo: `potenciaC(2)(3) = 2 * (2 * (2 * 1)) = 8` (recursión de pila).
⚠️ En el ternario no olvides las 3 partes: `condición ? siVerdad : siFalso`.
⚠️ En cálculo λ **puro** una función no puede llamarse por su nombre (son anónimas): para eso existe el **combinador Y** (parte 2).

---

## 4. Variables libres y atadas

- **Atada:** hay una **λ con ese nombre arriba** de ella (es un parámetro).
- **Libre:** ninguna λ la atrapa (viene de afuera, como una global).

### Método del árbol
Dibuja el AST (`λx` por cada función, `app` por cada aplicación). Desde la variable, **sube** hacia la raíz: la **primera** `λ` con su nombre es la dueña. Si llegas a la raíz sin encontrarla, es libre.

> **Analogía:** buscar la familia de un niño subiendo pisos en un edificio; la primera familia con su apellido es la suya.

### Método de las cajas 📦 (rápido en el examen)
Cada `nombre =>` abre una **caja** hasta el `)` que cierra su grupo. Una variable pertenece a **la caja más pequeña con su nombre** que la contiene.

### Ejemplos
- `λx. f x`: `x` atada, `f` libre.
- `λy. [x, λx. x]` (slide 16): la primera `x` es **libre** y la segunda **atada**. ¡La misma letra puede ser libre y atada!
- `λx. (λx. x)`: la `x` es de la λ **más cercana** (la de adentro "tapa" a la de afuera: *shadowing*).

### Ejercicio b (sesión 32): `x => f( (x => g(x => x)) x )`
```
        λx  ①
         │
        app
       /    \
      f      app
            /    \
         λx ②     x   (A)
          │
         app
        /    \
       g     λx ③
              │
              x       (B)
```
| Variable | Dueña | Resultado |
|---|---|---|
| (A) `x` del final | ① (la de afuera) | atada |
| (B) `x` de adentro | ③ | atada |
| `f`, `g` | ninguna | **libres** |

⚠️ (A) está al lado de ② pero **no** es de ②: ② es su "hermano" en el árbol, no está en su camino hacia la raíz.

---

## 5. Las tres reglas de reducción

| Regla | Qué hace | JS |
|---|---|---|
| **α** | renombrar un parámetro | `x => x+1` ≡ `y => y+1` |
| **β** | aplicar: sustituir el parámetro por el argumento | `(x => x+1)(5)` → `5+1` |
| **η** | quitar un envoltorio que solo reenvía | `x => f(x)` ≡ `f` |

### α — renombrar (α-equivalencia)
Puedes cambiar el nombre de un parámetro **junto con todas las variables que atrapa**, y la función sigue siendo la misma.

Condiciones:
1. Renombrar **todas** las que esa λ atrapa, **y solo esas**: `λx. (x, λx. x)` →α `λz. (z, λx. x)`.
2. El nombre nuevo **no puede atrapar una variable libre**: `λx. x + y` →α `λy. y + y` ❌ (**captura**).

> **Analogía:** cambiarle la etiqueta a un ingrediente en una receta: da igual, siempre que se cambie en todos lados y no se confunda con otro.

**Truco:** dos expresiones son α-equivalentes si sus árboles tienen la misma forma y cada variable tiene el dueño en la misma posición (o es libre con el mismo nombre).

**Ejercicio c (sesión 32):** ¿`x => f( (x => g(x => x)) x )` y `x => f( (w => g(z => z)) w )` son α-equivalentes?
> **No.** En la primera, la `x` del argumento final está **atada** por la λ exterior. En la segunda, la `w` final está **fuera** del alcance de `w =>` (esa caja se cierra antes), así que es **libre**. Renombrar no puede convertir una variable atada en libre.

Versiones que **sí** serían equivalentes: `x => f( (w => g(z => z)) x )` o `y => f( (w => g(z => z)) y )`.

### η — quitar el intermediario
```
λv. (M v)   →η   M      si v NO aparece libre en M
```
> **Analogía:** un reenvío de WhatsApp: si solo reenvías el mensaje sin cambiarlo, sobras.

```javascript
[1, -2, 3].map(x => Math.abs(x))   ≡   [1, -2, 3].map(Math.abs)
```
**Pregunta clave:** ¿el cuerpo es exactamente `ALGUIEN(v)`, con `v` **sola** como último argumento y ALGUIEN **sin usar** `v`? Si sí, queda `ALGUIEN`. Se quitan **las dos cosas juntas**: el `v =>` de adelante y la `(v)` de atrás.

| Función | ¿η? |
|---|---|
| `x => f(x)` | ✅ → `f` |
| `x => g(x)` | ✅ → `g` |
| `y => h(z)(y)` | ✅ → `h(z)` |
| `λx. (λy. y y) x` (slide 24) | ✅ → `λy. y y` |
| `x => f(x + 1)` | ❌ hace algo propio |
| `x => f(g(x))` | ❌ el argumento es `g(x)`, no `x` |
| `x => g(x => x)` | ❌ el argumento es una función, no `x` |
| `y => h(y)(z)` | ❌ el último argumento es `z` |
| `x => f(x)(x)` | ❌ ALGUIEN = `f(x)` usa `x` |
| `x => x` | ❌ no llama a nadie (es la identidad: un "espejo", no se reduce a `x`) |

**Ejercicio d (sesión 32):** ¿hay η-reducción en `x => f( (x => g(x => x)) x )`?
> **No.** Ninguna abstracción tiene la forma `λv. (M v)`: en la exterior, el argumento de `f` es `(λx. g(λx. x)) x` (no `x` sola); en la del medio, el argumento de `g` es `λx. x`; la interior es la identidad. Extra: sí hay una **β**: `(x => g(x => x)) x` → `g(x => x)`, y queda `x => f(g(x => x))`, que tampoco tiene η.

### β — aplicar (la que "calcula")
```
(λx. M) N   →β   M con cada x LIBRE reemplazada por N
```
A `(λx. M) N` se le llama **β-redex**.

> **Analogía: llenar una plantilla.** "Estimado [NOMBRE], gracias [NOMBRE]": reemplazas **cada** [NOMBRE] y botas la etiqueta de la plantilla.

**Receta:** (1) copia el cuerpo, (2) reemplaza **todas** las `x` libres por `N`, (3) borra `λx.` y `N`.

Ejemplos:
```
(λx. x) y              →β  y
(λx. f x x) a          →β  f a a
(λx. λy. x) a b        →β  (λy. a) b   →β  a           ("devuelve el primero")
(λz. λx. λy. z z)(λa. b)
                       →β  λx. λy. (λa. b)(λa. b)      z ← λa. b
                       →β  λx. λy. b                   (slide 21)
(λx. x (λx. x)) a      →β  a (λx. x)                   solo las libres
```

### Práctica (resuelta)
| Expresión | Pasos | Resultado |
|---|---|---|
| `(λx. x x) a` | `x ← a` | `a a` |
| `(λf. f (f z)) g` | `f ← g` (**las dos** f) | `g (g z)` |
| `(λx. λy. y x) a b` | `x ← a` → `(λy. y a) b`; `y ← b` | `b a` (¡no `a b`!) |
| `(λx. λy. x) (λz. z)` | `x ← λz. z` | `λy. λz. z` |

⚠️ **Errores típicos:** reemplazar solo la primera ocurrencia; saltarse pasos con varios argumentos (haz **un β a la vez** y escribe qué reemplazas); olvidar el punto después del parámetro.

### Forma normal
Una expresión está en **forma normal (β)** cuando ya no tiene ningún β-redex: es "el resultado".
> **Analogía:** simplificar una fracción hasta que no se pueda más: `12/18 → 6/9 → 2/3`.

- Si existe, es **única** (salvo nombres de variables atadas, o sea, α-equivalencia), sin importar el orden en que se redujo (slide 28).
- **No siempre existe**: hay expresiones que se reducen para siempre.
- **No es decidible** saber de antemano si una expresión tiene forma normal.

---

## 6. Resumen rápido

| Tema | Recordar |
|---|---|
| 3 piezas | `x` · `λx. M` (`x => M`) · `M N` (`M(N)`) |
| Lectura | aplicación a la izquierda; cuerpo hasta el final; `λx y.` = `λx. λy.` |
| Currying | un argumento a la vez: `x => y => ...` |
| Aplicación parcial | `sumarC(10)` devuelve una función pre-configurada |
| Clausura | función + valores de sus variables libres (mochila 🎒) |
| Libre / atada | subir por el árbol o usar cajas 📦; la λ más cercana con ese nombre gana |
| α | renombrar con todo lo que atrapa, sin capturar libres |
| η | `λx. (M x)` → `M` si solo reenvía y `M` no usa `x` |
| β | `(λx. M) N` → `M[x := N]`, todas las libres, un paso a la vez |
| Forma normal | sin redex; única si existe; puede no existir |

**Siguiente (parte 2):** estrategias de evaluación (eager contra lazy), numerales de Church, booleanos (`TRUE = λx y. x`, ¡el "devuelve el primero"!), ITE, OR y el combinador Y.
