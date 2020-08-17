# はじめに

継承（英：inheritance）は、オブジェクト指向ソフトウェアテクノロジの概念です。[Wikipedia](https://ja.wikipedia.org/wiki/%E7%B6%99%E6%89%BF_(%E3%83%97%E3%83%AD%E3%82%B0%E3%83%A9%E3%83%9F%E3%83%B3%E3%82%B0))によりますと：
> 継承とはオブジェクト指向を構成する概念の一つである。あるオブジェクトが他のオブジェクトの特性を引き継ぐ場合、両者の間に「継承関係」があると言われる。

Pythonのオブジェクト指向プログラミングには、継承、カプセル化、ポリモーフィズムの3つの主要な特性があります。Python2および以前のバージョンでは、組み込み型から派生したクラス（組み込み型がクラスツリーの特定の位置にある限り）はすべて新スタイルクラスです。逆に、組み込み型から派生していないクラスは旧スタイルクラスと呼ばれます。


# Python新スタイルクラスとC3線形化アルゴリズム

Python3以降、そのような区別はなくなり、すべてのクラス組み込み型`object`から派生しています。`object`から明示に継承するかどうかに関係なく、すべてのクラスは新スタイルクラスです。旧スタイルクラスの場合、多重継承のMROは深さ優先探索です、つまり下から上への検索です。新スタイルクラスのMROは`C3線形化`というアルゴリズムを使用ています（さまざまな状況で、幅優先探索または深さ優先探索で表現できます）。

`C3線形化`が深さ優先探索で表現される例：
```python
# C3-深さ優先探索（D -> B -> A -> C）
class A:
    var = 'A var'

class B(A):
    pass

class C:
    var = 'C var'

class D(B, C):
    pass

if __name__ == '__main__':
    print(D.mro())  # [<class '__main__.D'>, <class '__main__.B'>, <class '__main__.A'>, <class '__main__.C'>, <class 'object'>]

    print(D.var)    # A var
```

`C3線形化`が幅優先探索で表現される例：
```python
# C3-幅優先探索（D -> B -> C -> A）
class A:
    var = 'A var'

class B(A):
    pass

class C(A):
    var = 'C var'

class D(B, C):
    pass

if __name__ == '__main__':
    print(D.mro())  # [<class '__main__.D'>, <class '__main__.B'>, <class '__main__.C'>, <class '__main__.A'>, <class 'object'>]
    print(D.var)    # C var
```

# Pythonの単一継承

- Pythonの継承では、親クラスのコンストラクタは自動的には呼び出されません。派生クラスのコンストラクタで具体的に呼び出す必要があります。
- 親クラスのメソッドを呼び出すときは、親クラスのクラス名をプレフィックスとして追加し、`self`パラメータを含む必要があります。
- Pythonは常に対応するクラスのメソッドを最初に検索します。派生クラスで対応するメソッドが見つからない場合は、親クラスで1つずつ検索します。

Python単一継承の例：
```python
class Animal:
   def __init__(self, name, age):
       self.name = name
       self.age = age

   def call(self):
       print(self.name, 'Meow')


class Cat(Animal):
   def __init__(self, name, age, sex):
       super(Cat, self).__init__(name, age)  # # Animalクラスから初期化する
       self.sex = sex


if __name__ == '__main__':  
   c = Cat('Kitten', 1, 'Male')  
   c.call()  

# アウトプット: Kitten Meow ，Catは親クラスAnimalのメソッドを継承しました
```

# Pythonの多重継承
多重継承の場合、派生クラスは複数の親クラスのメソッドを使用できます。Python多重継承の例：
```python
class Plant:
    def __init__(self, color):
        print("init Plant start")
        self.color = color
        print("init Plant end")

    def show(self):
        print("The Plant color is:", self.color)


class Fruit:
    def __init__(self, color):
        print("init Fruit start")
        self.color = color
        print("init Fruit end")

    def show(self):
        print("The Fruit color is:", self.color)


class Melon(Fruit):
    def __init__(self, color):
        print("init Melon start")
        super(Melon, self).__init__(color)
        self.color = color
        print("init Melon end")

    def show(self):
        print("The Melon color is:", self.color)


class Mango(Fruit, Plant):
    def __init__(self, color):
        print("init Mango start")
        Plant.__init__(self, color)
        Fruit.__init__(self, color)
        self.color = color
        print("init Mango end")

    def show_color(self):
        print("The Mango color is:", self.color)


class Watermelon(Melon, Plant):
    def __init__(self, color):
        print("init Watermelon start")
        Melon.__init__(self, color)
        Plant.__init__(self, color)
        self.color = color
        print("init Watermelon end")

    def show_color(self):
        print("The Watermelon color is:", self.color)


if __name__ == "__main__":
    mango = Mango("yellow")
    mango.show()

    watermelon = Watermelon("red")
    watermelon.show()
```
サンプルコードでは、まず、`Plant`クラスと`Fruit`クラスを定義し、`Melon`クラスを`Fruit`クラスから継承します。そして、`Fruit`クラスと`Plant`クラスから継承する`Mango`クラスを定義し、`Melon`クラスと`Plant`クラスから継承する`Watermelon`クラスを定義します。実行結果は以下となります。
```
init Mango start
init Plant start
init Plant end
init Fruit start
init Fruit end
init Mango end
The Fruit color is: yellow

init Watermelon start
init Melon start
init Fruit start
init Fruit end
init Melon end
init Plant start
init Plant end
init Watermelon end
The Melon color is: red
```
### 多重継承のメソッド実行順序

実行結果から分析しますと：

1. 派生クラスを定義するときは、括弧で囲まれた親クラスを継承する順序に注意する必要があります。親クラスに同じ名前のメソッドがある場合、派生クラスでその同名メソッドを使用するときに指定しないと、Pythonは左から右に検索します。つまり、メソッドが派生クラスに見つからない場合、左から右に、親クラスにメソッドが含まれているかどうかを確認します。
2. 継承された親クラスで、親クラスが他の親クラスも継承する場合、同じ名前のメソッドを呼び出すと最初にアクセスした同名メソッドを呼び出します。
3. Pythonはマルチレベルの親クラスの継承をサポートします。マルチレベルの親クラスの継承をサポートします。派生クラスは、そのうえにあるマルチレベル親クラスのすべてのプロパティとメソッドを継承します。

### 多重継承時superメソッドによる初期化

単一継承では、同じクラスが複数回インスタンス化されるのを防ぐため、`super`メソッドを使用して親クラスのコンストラクタを初期化しますが、これにより多重継承で問題が発生することがよくあります。上記のサンプルコードを次の通りに変更してみますと：
```python
class Plant(object):
    def __init__(self, color):
        print("init Plant start")
        self.color = color
        print("init Plant end")

    def show(self):
        print("The Plant color is:", self.color)


class Fruit(object):
    def __init__(self, color):
        print("init Fruit start")
        self.color = color
        print("init Fruit end")

    def show(self):
        print("The Fruit color is:", self.color)


class Melon(Fruit):
    def __init__(self, color):
        print("init Melon start")
        super(Melon, self).__init__(color)
        self.color = color
        print("init Melon end")

    def show(self):
        print("The Melon color is:", self.color)


class Mango(Fruit, Plant):
    def __init__(self, color):
        print("init Mango start")
        super(Mango, self).__init__(color)
        self.color = color
        print("init Mango end")

    def show_color(self):
        print("The Mango color is:", self.color)


class Watermelon(Melon, Plant):
    def __init__(self, color):
        print("init Watermelon start")
        super(Watermelon, self).__init__(color)
        self.color = color
        print("init Watermelon end")

    def show_color(self):
        print("The Watermelon color is:", self.color)


if __name__ == "__main__":
    mango = Mango("yellow")
    mango.show()

    watermelon = Watermelon("red")
    watermelon.show()
```

上記の`Mango`クラスと`Watermelon`クラスの親クラスの初期化方法をsuperに変更するだけで実行結果は以下の通りに変わりました：
```
init Mango start
init Fruit start
init Fruit end
init Mango end
The Fruit color is: yellow
init Watermelon start
init Melon start
init Fruit start
init Fruit end
init Melon end
init Watermelon end
The Melon color is: red
```

`mango`と`watermelon`2つのインスタンスでは、`Plant`クラスはインスタンス化されません。多重継承で`super`メソッドの使用には問題があり、インスタンス化は一度しか行われないため、`super`メソッドは多重継承の状況での使用には適していません。この問題を解決するには、`super`メソッドを`Fruit`クラスに追加し、`Fruit`クラスを以下の通りに変更します。

```python
class Fruit(object):
    def __init__(self, color):
        super().__init__(color)
        print("init Fruit start")
        self.color = color
        print("init Fruit end")

    def show(self):
        print("The Fruit color is:", self.color)
```

# まとめ
- python2.3以降バージョンのメソッド解決順序(Method Resolution Order)は、C3線形化アルゴリズムに従って実装されています。
- 単一継承の場合は、`super().init`は`クラス名.init`と同じです。
- 多重継承の場合は、継承された各親クラスが`super`メソッドを実装しない限り、`super`メソッドは最初の親クラスしか初期化できません。さらに、派生クラスが複数の親クラスから派生し、派生クラスに独自のコンストラクタがない場合：

  1. 順番によって継承します、例えば、一番上にある(最初の)親クラスが独自のコンストラクタを持つとそのコンストラクタを継承します。

  2. 最初の親クラスにコンストラクタがない場合、2番目のコンストラクタが継承されます、2番目もコンストラクタがない場合は、それ以降を探します、こういう流れで類推します。


# 参考
> [C3 linearization - Wikipedia](https://en.wikipedia.org/wiki/C3_linearization)
> [The C3 algorithm, under control of a total order](https://doc.sagemath.org/html/en/reference/misc/sage/misc/c3_controlled.html)
> [The Python 2.3 Method Resolution Order](https://www.python.org/download/releases/2.3/mro/)
