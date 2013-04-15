title: Playing with Haskell
date: 2013-04-15 08:03
tags: haskell

Have you ever written a program to calculate the symbolic [derivative](http://en.wikipedia.org/wiki/Derivative) for a given function? I've tried to solve this problem with Haskell and the result is very simple and smart.

First at all, let's define the data structures representing the math terms:

    :::haskell
    data Term = Const Integer | Variable String |
                Add Term Term | Mul Term Term |
                Div Term Term | Pow Term Term |
                Sin Term | Cos Term | Tan Term | Ln Term
      deriving(Eq)

Assuming to have the functions `add x y`, `mul x y`, etc., we can define the derivative functions implementing the math rules:

    :::haskell
    derive (Variable u) (Variable v) = Const (if u == v then 1 else 0)
    derive (Const _) _ = zero
    derive (Add x y) v = add (derive x v) (derive y v)
    derive (Mul x y) v = add (mul (derive x v) y) (mul x (derive y v))
    derive (Pow x y) v = mul (mul y (pow x (sub y one))) (derive x v)
    derive (Div x y) v = divide (sub (mul (derive x v) y) (mul x (derive y v))) (pow y two)
    derive (Sin x)   v = mul (cos x) (derive x v)
    derive (Cos x)   v = mul (neg (sin x)) (derive x v)
    derive (Tan x)   v = derive (Div (sin x) (cos x)) v
    derive (Ln x)    v = mul (inv x) (derive x v)

<!-- more -->

Hence let's create the math term constructors based on the relative operations applying some simplification strategies:

    :::haskell
    import Prelude hiding (sin, cos, tan)

    {- Some constants -}

    zero    = Const 0
    one     = Const 1
    one_neg = Const (-1)
    two     = Const 2

    {- Simple functions -}

    neg x = mul one_neg x
    inv x = divide one x
    inc x = add x one
    dec x = sub x one

    {- Addition -}

    add (Const n) (Const m)   = Const (n + m)
    add (Const 0) y           = y       -- 0 + y = y
    add x         y@(Const m) = add y x -- x + m = m + x

    add (Add x y) z         | x == z = add y (add x x) -- (x + y) + x = y + (x + x)
                            | y == z = add x (add y y) -- (x + y) + y = x + (y + y)
    add x         (Add y z) | x == y = add z (add x x) -- x + (x + z) = z + (x + x)
                            | x == z = add y (add x x) -- x + (y + x) = y + (x + x)
                            | otherwise = add (add x y) z

    add (Mul x y) (Mul w z) | x == w = mul x (add y z) -- xy + xz = x(y + z)
                            | x == z = mul x (add y w)
                            | y == w = mul y (add x z)
                            | y == z = mul y (add x w) -- xy = wy = y(x + w)
    add x         (Mul y z) | x == y = mul x (inc z)
                            | x == z = mul x (inc y)
    add (Mul x y) z         | x == z = mul z (inc y)
                            | y == z = mul z (inc x)

    add x           (Div y z) = divide (add (mul x z) y) z
    add t@(Div x y) z         = add z t

    add (Pow (Sin x) (Const 2)) (Pow (Cos y) (Const 2)) | x == y = one -- sin(x)^2 + cos(x)^2 = 1
    add (Pow (Cos x) (Const 2)) (Pow (Sin y) (Const 2)) | x == y = one -- cos(x)^2 + sin(x)^2 = 1

    add x y | x == y    = mul two x -- x + x = 2x
            | otherwise = Add x y

    {- Subtraction -}

    sub x y = add x (neg y)

    {- Multiplication -}

    mul (Const n) (Const m)   = Const (n * m)
    mul (Const 0) _           = zero -- 0*x = 0
    mul (Const 1) y           = y    -- 1*x = x
    mul x         y@(Const m) = mul y x

    mul (Mul x y) z | x == z = mul y (mul x z)
                    | y == z = mul x (mul y z)
    mul x (Mul y z) | x == y = mul z (mul x y)
                    | x == z = mul y (mul x z)
                    | otherwise = mul (mul x y) z

    mul (Pow x y) (Pow w z) | x == w = pow x (add y z)
    mul x         (Pow y z) | x == y = pow x (inc z)
    mul (Pow x y) z         | x == z = pow x (inc y)

    mul (Div x y) z         | y == z = x
                            | otherwise = divide (mul x z) y
    mul x         (Div y z) | x == z = y
                            | otherwise = divide (mul x y) z

    mul x y | x == y    = pow x two
            | otherwise = Mul x y

    {- Division -}

    divide (Const n) (Const m) = divide (Const (div n d)) (Const (div m d)) where d = gcd n m
    divide (Const 0) _         = zero
    divide x         (Const 1) = x

    divide (Pow x y) (Pow w z) | x == w = pow x (sub y z)
    divide x         (Pow y z) | x == y = pow x (dec z)
    divide (Pow x y) z         | x == z = pow x (dec y)

    divide (Div x y) z         = divide x (mul y z)
    divide x         (Div y z) = mul x (divide z y)

    divide (Sin x) (Cos y) | x == y = tan x

    divide x y | x == y = one
               | otherwise = Div x y

    {- Exponentiation -}

    pow (Const n) (Const m) = Const (n ^ m)
    pow (Const 0) _ = zero
    pow (Const 1) _ = one
    pow _ (Const 0) = one
    pow x (Const 1) = x
    pow (Pow x y) w = pow x (mul y w) -- (x^y)^w = x^(y*w)
    pow x y = Pow x y

    {- Trascendental functions -}

    sin x = Sin x
    cos x = Cos x
    tan x = Tan x

    ln (Pow x y) = mul y (ln x)
    ln (Mul x y) = add (ln x) (ln y)
    ln (Div x y) = sub (ln x) (ln y)
    ln x = Ln x

First to test it, we implement the method to show an expression in a more concise way.

    :::haskell
    instance Show Term where
     showsPrec _ = showTerm
      where
        showTerm t = case t of
          Const n -> shows n
          Variable u -> showString u
          Add x y -> parenthesize brace t x . showString "+" . parenthesize brace t y
          Mul x y -> parenthesize brace t x . showString "*" . parenthesize brace t y
          Div x y -> parenthesize brace t x . showString "/" . parenthesize brace t y
          Pow x y -> parenthesize brace t x . showString "^" . parenthesize brace t y
          Sin x -> showFunction "sin" t x
          Cos x -> showFunction "cos" t x
          Tan x -> showFunction "tan" t x
          Ln x  -> showFunction "ln" t x
        showFunction name t x = showString name . round x

        parenthesize p t x | precedence x < precedence t = p x
                           | otherwise                   = shows x
        round t = ('(':) . shows t . (')':)
        brace t = ('{':) . shows t . ('}':)

        precedence t = case t of
          Add _ _    -> 1
          Mul _ _    -> 7
          Div _ _    -> 8
          Pow _ _    -> 10
          Const _    -> 10
          Variable _ -> 10
          otherwise  -> 11

Finally, testing the module you have:

    :::haskell
    x = Variable "x"
    y = Variable "y"

    derive (tan x) x                                --> 1/cos(x)^2
    derive (tan (ln (pow x y))) x                   --> y/{x*cos(y*ln(x))^2}
    derive (mul (pow (ln x) two) (sin (mul y x))) x --> {ln(x)^2*cos(y*x)*y*x+2*ln(x)*sin(y*x)}/x