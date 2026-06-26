# Gramática formal de VRAL

Notación EBNF: `*` = cero o más, `+` = uno o más, `?` = opcional, `|` = alternativa,
`{ }` = agrupación, `"texto"` = terminal literal, `NAME_MAYUS` = terminal por categoría.

Esta especificación es normativa. El compilador (`vralc`) implementa exactamente esta
gramática — cualquier divergencia es un bug del compilador, no de la spec.

---

## Programa

```ebnf
program  ::= domain+

domain   ::= "domain" NAME
                 verdicts_sec
                 payload_sec
                 model_sec
                 state_sec?
                 neighborhood_sec?
                 transition_sec
                 coherence_sec
             "end"
```

Las secciones `verdicts`, `payload`, `model`, `transition` y `coherence` son **obligatorias**.
`state` y `neighborhood` son opcionales. El orden de las secciones dentro del dominio
no está forzado por el parser, pero la convención canónica es el orden arriba.

---

## Secciones

```ebnf
verdicts_sec     ::= "verdicts" NAME+

payload_sec      ::= "payload" type

model_sec        ::= "model" NAME
                         field_decl*
                     "end"

state_sec        ::= "state" NAME
                         scalar_field_decl+
                     "end"

neighborhood_sec ::= "neighborhood" ("spatial" | "temporal") ("max" INTEGER)?

transition_sec   ::= "transition" IDENT IDENT?
                         trans_stmt*
                         trans_rule+
                     "end"

coherence_sec    ::= "coherence" IDENT IDENT
                         expr
                     "end"
```

**Restricciones sobre `state_sec`:** todos los campos deben ser escalares (`TypeScalar`).
Arrays y matrices en `state` están rechazados por el verificador (ventana temporal requiere
diseño de estado explícito fuera del MVP).

**Restricciones sobre `neighborhood_sec`:** `max` debe ser un entero en `[1, 256]`.
Default si se omite: 8.

---

## Declaraciones de campo

```ebnf
field_decl        ::= IDENT type

scalar_field_decl ::= IDENT scalar_type
```

---

## Sistema de tipos

```ebnf
type        ::= scalar_type
              | scalar_type "[" INTEGER "]"
              | scalar_type "[" INTEGER "," INTEGER "]"

scalar_type ::= "Float32" | "Float64" | "Int32" | "Bool"
```

`T[n]` es un array de `n` elementos de tipo `T` (`n > 0`).
`T[n, m]` es una matriz de `n` filas y `m` columnas (`n > 0, m > 0`).

---

## Cuerpo de `transition`

```ebnf
trans_stmt   ::= let_stmt | state_update_stmt

let_stmt          ::= IDENT "=" expr

state_update_stmt ::= "state" "." IDENT ":=" expr

trans_rule   ::= expr "->" NAME
               | "else" "->" NAME
```

**Restricciones:**
- La rama `else` es obligatoria y debe ser la **última** regla.
- Todo `NAME` en `->`  debe estar declarado en `verdicts`.
- Todo `state.<campo>` en `:=` y en expresiones debe existir en `state_sec`.
- Si no hay `state_sec`, las instrucciones `state.<campo> :=` son error.
- Las primitivas `nb_*` solo son válidas si `transition` declara el segundo
  parámetro (`IDENT?` en `transition_sec`).

---

## Expresiones

```ebnf
expr         ::= expr_add { cmp_op expr_add }?

expr_add     ::= expr_mul { ("+" | "-") expr_mul }*

expr_mul     ::= expr_primary { ("*" | "/") expr_primary }*

expr_primary ::= NUMBER
               | "true"
               | "false"
               | "-" expr_primary
               | "(" expr ")"
               | access_or_call

access_or_call ::= ident_like
                   ( "(" { expr { "," expr }* }? ")" )?
                   ( "." IDENT )?
                   ( "[" expr "]" )?

ident_like   ::= IDENT | "payload" | "model" | "transition"
               | "coherence" | "verdicts" | "true" | "false" | "state"

cmp_op       ::= "<" | ">" | "<=" | ">=" | "==" | "!="
```

Las comparaciones no se encadenan: `a < b < c` es un error de parse.
El acceso a campo (`.`) y el índice (`[i]`) solo pueden aparecer una vez cada uno
y en ese orden: primero `.field`, luego `[i]`.

---

## Intrínsecos certificados

Todas las llamadas a funciones en expresiones deben ser uno de los siguientes intrínsecos.
Cualquier otro nombre es rechazado por el verificador (Axioma A_Sys).

| Intrínseco      | Aridad | Tipo de retorno | Descripción |
|-----------------|--------|-----------------|-------------|
| `distancia`     | 2      | Float           | Distancia euclidiana entre dos vectores |
| `dot`           | 2      | Float           | Producto punto |
| `norm`          | 1      | Float           | Norma euclideana de un vector |
| `mahalanobis`   | 3      | Float           | `sqrt((a-b)^T M (a-b))` — M debe ser matriz NxN del model |
| `sqrt`          | 1      | Float           | Raíz cuadrada (`llvm.sqrt.f32`) |
| `abs`           | 1      | Float           | Valor absoluto (`llvm.fabs.f32`) |
| `max`           | 2      | Float           | Máximo (`llvm.maxnum.f32`) |
| `min`           | 2      | Float           | Mínimo (`llvm.minnum.f32`) |
| `nonfinite`     | 1      | Bool (i1)       | True si el argumento es NaN o Inf |
| `anynonfinite`  | 1      | Bool (i1)       | True si algún elemento del vector es no-finito |
| `nb_count`      | 0      | Float           | Número de vecinos válidos (solo en `transition c neighbors`) |
| `nb_vote`       | 1      | Float           | Vecinos con score >= umbral (solo en `transition c neighbors`) |
| `nb_max`        | 0      | Float           | Máximo de scores de vecinos |
| `nb_min`        | 0      | Float           | Mínimo de scores de vecinos |
| `nb_mean`       | 0      | Float           | Media de scores de vecinos |

**Restricción de `mahalanobis`:** el tercer argumento debe ser un campo de tipo
matriz del bloque `model`. No se acepta una expresión arbitraria.

---

## Terminales

```ebnf
NAME     ::= [A-Z][A-Za-z0-9_]*     (* empieza con mayúscula *)
IDENT    ::= [a-z_][A-Za-z0-9_]*    (* empieza con minúscula o _ *)
INTEGER  ::= [0-9]+
NUMBER   ::= [0-9]+ | [0-9]+ "." [0-9]+
```

Los comentarios de línea `// ...` y el espacio en blanco son ignorados por el lexer.

---

## ABI de la librería generada

Para un dominio `MiDominio` con payload `T[n]`, el compilador genera:

```c
/* Ejecuta un tick del monitor.
   payload   : T[n]          — observación actual
   mean      : T[n]          — centroide del modelo
   inv_cov   : T[n*n]        — inversa de covarianza, row-major (si el modelo tiene matriz NxN)
   <params>  : T             — parámetros escalares del modelo, en orden de declaración
   state_ptr : T[|state|]    — buffer de estado propiedad del llamador; leído y escrito
   retorna   : int8_t        — índice del veredicto (1 = primero, 2 = segundo, ...)
*/
int8_t midominio_transition(T* payload, T* mean, T* inv_cov, T param1, ..., T* state_ptr);

/* Chequeo de coherencia entre dos payloads consecutivos.
   prev, next : T[n]
   retorna    : int32_t (0 = fallo, 1 = ok)
*/
int32_t midominio_coherence(T* prev, T* next);
```

El nombre de la función es el nombre del dominio en minúsculas + `_transition` / `_coherence`.

---

## Invariantes del código generado (Axiomas)

| Axioma | Garantía verificable en el IR |
|--------|-------------------------------|
| **A1** | Sin `malloc`/`calloc`/`new`. Solo `alloca` de tamaño fijo en tiempo de compilación (staging de arrays). Sin `free`. |
| **A_Sys** | Solo llamadas a intrínsecos LLVM puros (`llvm.sqrt`, `llvm.maxnum`, `llvm.minnum`, `llvm.fabs`). Sin `call` a libc ni syscalls. |
| **A3** | Existe una implementación de referencia Python/NumPy congelada. El harness diff-test verifica acuerdo con tolerancia relativa ≤ 10⁻⁴ sobre ≥ 400 entradas aleatorias. |
| **A4** | Sin instrucciones `br` dependientes de datos. Todas las bifurcaciones se compilan a `fcmp + select`. Cuenta de instrucciones fija para un dominio dado. |
