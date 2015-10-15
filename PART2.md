클로저 개발자를 위한 모나드 튜토리얼 2
=================

첫번째 장에서 기본적인 모나드인 identity 모나드와 maybe 모나드에 대해서 알아봤다. 두번째 장에서는 ``m-result``함수를 설명하기 위해 sequence 모나드에 대해 알아보려고 한다. 또 잘 알려진 유용한 몇가지 모나드 동작에 대해서도 설명하려고 한다.

sequence 모나드(하스켈에서는 list 모나드라고 부른다)는 많이 사용되는 모나드 중 하나다. sequence 모나드는 클로저에서 `for` 구문 형태로 기본적으로 제공하고 있다. 다음 예제를 보자:

```clj
(for [a (range 5)
      b (range a)]
  (* a b))
```

`for` 구문은 `let`구문과 비슷하다. `for`구문은 `let` 구문처럼 바인딩 리스트(먼저 나온 바인딩은 다음 바인딩에서 사용할 수 있다)와 바인딩 된 심볼을 사용해 최종 결과를 표현할 수 있는 구문으로 구성된다. `for` 구문이 `let` 구문과 다른 점은 `let`은 하나의 값이 하나의 심볼에 바인딩 되지만 `for`는 시퀀스의 여러개의 값이 바인딩 된다는 점이다. 그래서 바인딩 리스트는 시퀀스 값이어야 하고 결과 역시 시퀀스가 된다. `for` 구문은 ``:when``과 ``:while`` 절이 있는데 이것은 나중에 설명하려고 한다. 조합 가능한 계산을 표현하는 모나드의 관점에서 보면 시퀀스는 결정되지 않은 계산의 결과처럼 보인다. 예를 들어 하나 이상이 결과를 가지는 계산같은 것이다.

위 예제를 모나드 라이브러리를 써서 표현해 보면 다음과 같다.

```clj
(domonad sequence-m
  [a (range 5)
   b (range a)]
  (* a b))
```

앞서 domonad 매크로는 ``m-result``라고 부르는 표현의 마지막 부분에서 부터 ``m-bind``를 연속으로 부르는 형태라는 것을 알아봤다. 이제 ``m-bind``와 ``m-result``로 반복하는 구문처럼 표현하기 위해 어떻게 해야하는지 설명하려고 한다.

앞서 알아본 것처럼 ``m-bind``는 계산의 결과를 나타내는 하나의 인자와 바인딩된 값을 나타내는 함수로 호출 한다.
반복문처럼 쓰려면 이 함수를 반복해서 호출 하면된다. ``m-bind`` 함수의 첫번째 모습은 다음과 같다.

```clj
(defn m-bind-first-try [sequence function]
  (map function sequence))
```

어떻게 동작하는지 다음 예제를 보자.

```clj
(m-bind-first-try (range 5)  (fn [a]
(m-bind-first-try (range a)  (fn [b]
(* a b)))))
```

위 예제는 ``(() (0) (0 2) (0 3 6) (0 4 8 12))``와 같다. 하지만 `for`의 실행 결과는 ``(0 0 2 0 3 6 0 4 8 12)``이다.
그래서 아직 완성되지는 않았다. 결과가 하나의 시퀀스에 담겨있기를 기대했지만 ``m-bind``를 호출한 횟수의 깊이로 시퀀스가 감싸져 있다. ``m-bind``는 하나의 단계로만 감싸있어야 하기 때문에 안쪽에 감싸져있는 부분은 제거 해야한다. 이 일은 concat이 할 수 있다. 다음과 같이 해보자.

```clj
(defn m-bind-second-try [sequence function]
  (apply concat (map function sequence)))

(m-bind-second-try (range 5)  (fn [a]
(m-bind-second-try (range a)  (fn [b]
(* a b)))))
```

안타깝게도 예외가 발생한다. 예외 내용은 다음과 같다.

```clj 
java.lang.IllegalArgumentException: Don't know how to create ISeq from: Integer
```

다시 살펴보자!

Our current ``m-bind`` introduces a level of sequence nesting and also takes one away. Its result therefore has as many levels of nesting as the return value of the function that is called. The final result of our expression has as many nesting values as ``(* a b)`` – which means none at all. If we want one level of nesting in the result, no matter how many calls to ``m-bind`` we have, the only solution is to introduce one level of nesting at the end. Let’s try a quick fix:

```clj 
(m-bind-second-try (range 5)  (fn [a]
(m-bind-second-try (range a)  (fn [b]
(list (* a b))))))
```

This works! Our ``(fn [b] ...)`` always returns a one-element list. The inner ``m-bind`` thus creates a sequence of one-element lists, one for each value of ``b``, and concatenates them to make a flat list. The outermost ``m-bind`` then creates such a list for each value of ``a`` and concatenates them to make another flat list. The result of each ``m-bind`` thus is a flat list, as it should be. And that illustrates nicely why we need ``m-result`` to make a monad work. The final definition of the sequence monad is thus given by

```clj
(defn m-bind [sequence function]
  (apply concat (map function sequence)))

(defn m-result [value]
  (list value))
```

The role of ``m-result`` is to turn a bare value into the expression that, when appearing on the right-hand side in a monadic binding, binds the symbol to that value. This is one of the conditions that a pair of m-bind and ``m-result`` functions must fulfill in order to define a monad. Expressed as Clojure code, this condition reads

```clj
(= (m-bind (m-result value) function)
   (function value))
```

There are two more conditions that complete the three monad laws. One of them is

```clj
(= (m-bind monadic-expression m-result)
   monadic-expression)
```

with monadic-expression standing for any expression valid in the monad under consideration, e.g. a sequence expression for the sequence monad. This condition becomes clearer when expressed using the domonad macro:

```clj
(= (domonad
     [x monadic-expression]
      x)
   monadic-expression)
```

The final monad law postulates associativity:

```clj
(= (m-bind (m-bind monadic-expression
                   function1)
           function2)
   (m-bind monadic-expression
           (fn [x] (m-bind (function1 x)
                           function2))))
```

Again this becomes a bit clearer using domonad syntax:

```clj
(= (domonad
     [y (domonad
          [x monadic-expression]
          (function1 x))]
     (function2 y))
   (domonad
     [x monadic-expression
      y (m-result (function1 x))]
     (function2 y)))
```

It is not necessary to remember the monad laws for using monads, they are of importance only when you start to define your own monads. What you should remember about ``m-result`` is that ``(m-result x)`` represents the monadic computation whose result is ``x``. For the sequence monad, this means a sequence with the single element ``x``. For the identity monad and the maybe monad, which I have presented in the first part of the tutorial, there is no particular structure to monadic expressions, and therefore ``m-result`` is just the identity function.

Now it’s time to relax: the most difficult material has been covered. I will return to monad theory in the next part, where I will tell you more about the ``:when`` clauses in for loops. The rest of this part will be of a more pragmatic nature.

You may have wondered what the point of the identity and sequence monads is, given that Clojure already contains fully equivalent forms. The answer is that there are generic operations on computations that have an interpretation in any monad. Using the monad library, you can write functions that take a monad as an argument and compose computations in the given monad. I will come back to this later with a concrete example. The monad library also contains some useful predefined operations for use with any monad, which I will explain now. They all have names starting with the prefix m-.

Perhaps the most frequently used generic monad function is ``m-lift``. It converts a function of n standard value arguments into a function of n monadic expressions that returns a monadic expression. The new function contains implicit ``m-bind`` and ``m-result`` calls. As a simple example, take

```clj
(def nil-respecting-addition
  (with-monad maybe-m
    (m-lift 2 +)))
```

This is a function that returns the sum of two arguments, just like ``+`` does, except that it automatically returns ``nil`` when either of its arguments is ``nil``. Note that ``m-lift`` needs to know the number of arguments that the function has, as there is no way to obtain this information by inspecting the function itself.

To illustrate how ``m-lift`` works, I will show you an equivalent definition in terms of ``domonad``:

```clj
(defn nil-respecting-addition
  [x y]
  (domonad maybe-m
    [a x
     b y]
    (+ a b)))
```

This shows that ``m-lift`` implies one call to ``m-result`` and one ``m-bind`` call per argument. The same definition using the sequence monad would yield a function that returns a sequence of all possible sums of pairs from the two input sequences.

Exercice: The following function is equivalent to a well-known built-in Clojure function. Which one?

```clj
(with-monad sequence-m
  (defn mystery
    [f xs]
    ( (m-lift 1 f) xs )))
```

Another popular monad operation is ``m-seq``. It takes a sequence of monadic expressions, and returns a sequence of their result values. In terms of domonad, the expression ``(m-seq [a b c])`` becomes

```clj
(domonad
  [x a
   y b
   z c]
  '(x y z))
```

Here is an example of how you might want to use it:

```clj
(with-monad sequence-m
   (defn ntuples [n xs]
      (m-seq (replicate n xs))))
```

Try it out for yourself!

The final monad operation I want to mention is ``m-chain``. It takes a list of one-argument computations, and chains them together by calling each element of this list with the result of the preceding one. For example, ``(m-chain [a b c])`` is equivalent to

```clj
(fn [arg]
  (domonad
    [x (a arg)
     y (b x)
     z (c y)]
    z))
```

A usage example is the traversal of hierarchies. The Clojure function parents yields the parents of a given class or type in the hierarchy used for multimethod dispatch. When given a Java class, it returns its base classes. The following function builds on parents to find the n-th generation ascendants of a class:

```clj
(with-monad sequence-m
  (defn n-th-generation
    [n cls]
    ( (m-chain (replicate n parents)) cls )))

(n-th-generation 0 (class []))
(n-th-generation 1 (class []))
(n-th-generation 2 (class []))
```

You may notice that some classes can occur more than once in the result, because they are the base class of more than one class in the generation below. In fact, we ought to use sets instead of sequences for representing the ascendants at each generation. Well… that’s easy. Just replace ``sequence-m`` by ``set-m`` and run it again!


In [part 3](PART3.md), I will come back to the ``:when`` clause in for loops, and show how it is implemented and generalized in terms of monads. I will also explain another monad or two. Stay tuned!
