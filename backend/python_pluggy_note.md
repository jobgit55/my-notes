#　はじめに

複数のエンジニアが同じプログラムの変更に参加する場合、頻繁に関数をオーバーライドするとコードが非常に複雑になり、プロジェクト全体のメンテナンスコストも高くなります。
では、コードのスケーラビリティと安定性を両立させる同時にエレガントな方法はありますか？
`pytest（Pythonユニットテストフレームワーク）`の作者はこの問題に気づきました、`pytest`のソースコードには`@pytest.hookimpl`デコレータが付けられた関数（プラグインの実現）が多く見つけられます。その役割は、プラグイン呼び出しの形式で関数のオーバーライドを置き換えることです。

`pytest`ソースコードの一部：
```python
@pytest.hookimpl(hookwrapper=True)
def pytest_load_initial_conftests(early_config: Config):
	ns = early_config.known_args_namespace
	if ns.capture == "fd":
		_py36_windowsconsoleio_workaround(sys.stdout)
	_colorama_workaround()
	_readline_workaround()
	pluginmanager = early_config.pluginmanager
	capman = CaptureManager(ns.capture)
	pluginmanager.register(capman, "capturemanager")

	# make sure that capturemanager is properly reset at final shutdown
	early_config.add_cleanup(capman.stop_global_capturing)

	# finally trigger conftest loading but while capturing (issue93)
	capman.start_global_capturing()
	outcome = yield
	capman.suspend_global_capture()
	if outcome.excinfo is not None:
		out, err = capman.read_global_capture()
		sys.stdout.write(out)
		sys.stderr.write(err)

```

以前、これはただ`pytest`のプラグインツールライブラリでしたが、このライブラリの開発と発展に伴い、作者はこのライブラリを`pytest`から分離し、`pluggy`という名前を付けました。

# pluggyとは

`pluggy`は`pytest`のプラグイン管理とフック呼び出しのコアコンポーネントです。`pluggy`は500以上のプラグインを`pytest`のデフォルト動作を自由に拡張およびカスタマイズしました。`pytest`自体は、適切なプロトコルに従って順次に呼び出されるプラグインのコレクションであるとも言えます。

メインプログラムにプラグインをインストールするとユーザーはメインプログラムの動作を拡張またはカスタマイズできます。プラグインコードは通常のプログラム実行の一部として実行され、プログラムの特定の機能を変更または強化します。

# pluggyの使い道

多くのエンジニアが同じプログラムをカスタマイズして拡張したい場合、関数のオーバーライドを使用するとプログラムコードが非常に混乱となります。`pluggy`の使用により、メインプログラムとプラグインを疎結合できます。
また、`pluggy`を使用するとメインプログラムの設計者は、フック関数の実装に必要なオブジェクトを慎重に検討させ、プラグイン作者にも明確なフレームワークを提供させます。

# pluggyの仕組み
Pluggyには3つの主要な概念があります:
1. PluginManager：プラグインの仕様とプラグイン自体を管理するために使用されます。
2. HookspecMarker：プラグイン呼び出し仕様を定義します。各仕様は1〜Nプラグインに対応でき、各プラグインは仕様を満たします。そうでない場合、外部から正常に呼び出すことができません。
3. HookimplMarker：プラグインを定義します。プラグインロジックの具体的な実現はこのクラスのデコレータにあります。

簡単に`pluggy`を試してみます
```python
import pluggy
# プラグイン仕様のクラスデコレータを作成する
hookspac = pluggy.HookspecMarker('example')
# プラグインのクラスデコレータを作成する
hookimpl = pluggy.HookimplMarker('example')

class MySpec(object):
    # プラグイン仕様を作成する
    @hookspac
    def myhook(self, a, b):
        pass

class Plugin_1(object):
    # プラグインを定義する
    @hookimpl
    def myhook(self, a, b):
        return a + b

class Plugin_2(object):
    @hookimpl
    def myhook(self, a, b):
        return a - b

# managerとhookの仕様を作成する
pm = pluggy.PluginManager('example')
pm.add_hookspecs(MySpec)

# プラグインを登録する
pm.register(Plugin_1())
pm.register(Plugin_2())

# プラグインのmyhookメソッドを呼び出す
results = pm.hook.myhook(a=10, b=20)
print(results)
```
このコードを簡単に言えば、クラスのメソッドに相応しいクラスデコレータを作成することです。これにより、プラグイン仕様とプラグイン自体が構築されます。
まず、`PluginManager`クラスをインスタンス化し、同時に唯一のプロジェクト名を渡します。`HookspecMarker`クラスと`HookimplMarker`クラスのインスタンス化も同じプロジェクト名を使用する必要があります。
プラグインマネージャを作成した後、`add_hookspecs`メソッドでプラグイン仕様を追加し、`register`メソッドをでプラグイン自体を追加します。
そうして、プラグイン呼び出し仕様とプラグイン自体を追加した後、プラグインマネージャの`hook`プロパティを介してプラグインを直接呼び出すことができます。

以上のコードによってまとめると`pluggy`を使用するプロセスは3つのステップに分けることができます:
1. `HookspecMarker`クラスデコレーターでプラグイン呼び出し仕様とプラグインロジックを定義する。
2. `PluginManager`を作成し、プラグイン呼び出し仕様をプラグイン自体にバインドする。
3. プラグインを呼び出す。

外部システムでプラグインを使用する場合は、`pm.hook.any_hook_function`メソッドを呼び出すだけで、登録済みのプラグインを簡単に呼び出すことができます。

# hookspacとhookimplデコレーターの役割
コードでは、`hookspac`クラスデコレータを使用してプラグイン呼び出し仕様を定義し、`hookimpl`クラスデコレータを使用してプラグイン自体を定義しています。この2つの機能実際には「デコレートされたメソッドに新しいプロパティを追加する」ことです。2つのデコレータロジックは類似しているため、ここではhookspacデコレーターコードのみを説明します。
```python
class HookspecMarker(object):

    def __init__(self, project_name):
        self.project_name = project_name
    def __call__(
        self, function=None, firstresult=False, historic=False, warn_on_impl=None
    ):
        def setattr_hookspec_opts(func):
            if historic and firstresult:
                raise ValueError("cannot have a historic firstresult hook")
            # デコレートされたメソッドに新しいプロパティを追加する
            setattr(
                func,
                self.project_name + "_spec",
                dict(
                    firstresult=firstresult,
                    historic=historic,
                    warn_on_impl=warn_on_impl,
                ),
            )
            return func

        if function is not None:
            return setattr_hookspec_opts(function)
        else:
            return setattr_hookspec_opts
```

クラスデコレータは、主にクラスの`__call__`メソッドをオーバーライドします。`__call__`メソッドのコアロジックは、setattrメソッドを使用して、デコレートされた`func`メソッドに新しいプロパティを追加することです、プロパティは現在のプロジェクト名に_specサフィックスを加えたものであり、プロパティ値は辞書オブジェクトです。

`pluginmanager.add_hookspecs`メソッドを呼び出す時に`hookspac`クラスデコレータによって追加された情報を使います。

`HookimplMarker`クラスは似ています、ただ追加されたプロパティが異なります。コアコードは下記の通りです:
```python
setattr(
    func,
    self.project_name + "_impl",
    dict(
        hookwrapper=hookwrapper,
        optionalhook=optionalhook,
        tryfirst=tryfirst,
        trylast=trylast,
        specname=specname,
    ),
)
```

# プラグイン仕様と登録済みプラグインの追加
`pluggy.PluginManager`クラスをインスタンス化した後、`add_hookspecs`メソッド通じてプラグインの登録とプラグイン仕様追加できます。

```python
def __init__(self, project_name, implprefix=None):
    """If ``implprefix`` is given implementation functions
    will be recognized if their name matches the ``implprefix``. """
    self.project_name = project_name
    # ---略---
    # 重要
    self.hook = _HookRelay()
    self._inner_hookexec = lambda hook, methods, kwargs: hook.multicall(
        methods,
        kwargs,
        firstresult=hook.spec.opts.get("firstresult") if hook.spec else False,
    )
```

上記のコードでは重要なのは、`self.hook`プロパティと`self._inner_hookexec`プロパティを定義することです。`self._inner_hookexec`は`hook`、`methods`、`kwargs`の3つのパラメーターを受け取り、これらのパラメーターを`hook.multicall`メソッドに渡す無名関数です。

次にadd_hookspecsメソッドを呼び出してプラグイン仕様を追加します。コードは次のとおりです:
```python
class PluginManager(object):

    # デコレートされたメソッドの対応するプロパティの情報（HookspecMarkerデコレータ追加した情報）を取得する。
    def parse_hookspec_opts(self, module_or_class, name):
        method = getattr(module_or_class, name)
        return getattr(method, self.project_name + "_spec", None)

    def add_hookspecs(self, module_or_class):
        names = []
        for name in dir(module_or_class):
            # プラグイン仕様情報を取得する
            spec_opts = self.parse_hookspec_opts(module_or_class, name)
            if spec_opts is not None:
                hc = getattr(self.hook, name, None)
                if hc is None:

                    hc = _HookCaller(name, self._hookexec, module_or_class, spec_opts)
                    setattr(self.hook, name, hc)
                # ---略---
```

上記のコードでは、メソッドの対応するプロパティのパラメーターは`parse_hookspec_opts`メソッドを介して取得されます。パラメーターが`None`ではない場合、`_HookRelay`クラスでデコレートされたメソッドの情報が取得します（このメソッドは`MySpec`クラスの`myhook`です）。ソースコードを見ると、`_HookRelay`クラスは実際には新しいプロパティを受け取るだけのクラスです。

`_HookRelay`クラスに`myhook`プロパティ情報がない場合は、`_HookCaller`クラスをインスタンス化し、それをself.hookのプロパティとして使用します。`_HookCaller`クラスコードの一部は次の通りです。
```python
class _HookCaller(object):
    def __init__(self, name, hook_execute, specmodule_or_class=None, spec_opts=None):
        self.name = name
        # ---略---
        self._hookexec = hook_execute
        self.spec = None
        if specmodule_or_class is not None:
            assert spec_opts is not None
            self.set_specification(specmodule_or_class, spec_opts)

    def has_spec(self):
        return self.spec is not None

    def set_specification(self, specmodule_or_class, spec_opts):
        assert not self.has_spec()
        self.spec = HookSpec(specmodule_or_class, self.name, spec_opts)
        if spec_opts.get("historic"):
            self._call_history = []

```
重要なのは`set_specification`メソッドです。このメソッドは`HookSpec`クラスをインスタンス化し、それを`self.spec`にコピーします。
これで、プラグイン仕様の追加が完了し、`register`メソッドを介してプラグイン自体を登録します。
```python
def register(self, plugin, name=None):
    # ---略---
    for name in dir(plugin):
        hookimpl_opts = self.parse_hookimpl_opts(plugin, name)
        if hookimpl_opts is not None:
            normalize_hookimpl_opts(hookimpl_opts)
            method = getattr(plugin, name)
            # プラグインをインスタンス化
            hookimpl = HookImpl(plugin, plugin_name, method, hookimpl_opts)
            name = hookimpl_opts.get("specname") or name
            hook = getattr(self.hook, name, None) # hookspecプラグイン仕様を取得する
            if hook is None:
                hook = _HookCaller(name, self._hookexec)
                setattr(self.hook, name, hook)
            elif hook.has_spec():
                # プラグインメソッドのメソッド名とパラメーターがプラグイン仕様と同じかどうかを確認する
                self._verify_hook(hook, hookimpl)
                hook._maybe_apply_history(hookimpl)
            # プラグイン仕様に追加してプラグインとプラグイン仕様のバインドする
            hook._add_hookimpl(hookimpl)
            hookcallers.append(hook)
```
上記のコードは`self.parse_hookimpl_opts`メソッドを介して`hookimpl`デコレーターによって追加された情報を取得し、次に`getattr`（plugin、name）メソッドを介してメソッド名を取得し、最後に`HookImpl`クラスを初期化します。そして、`_add_hookimpl`メソッドを通じて相応しいプラグイン仕様にバインドします。

`_add_hookimpl`メソッドは、hookimplインスタンスのプロパティに従って挿入位置を判断します。位置によって呼び出し順序が異なります。コードは次の通りです。
```python
def _add_hookimpl(self, hookimpl):
    """Add an implementation to the callback chain.
    """
    # デコレータの有り無し (プラグインロジックでyieldキーワード使用する)
    if hookimpl.hookwrapper:
        methods = self._wrappers
    else:
        methods = self._nonwrappers

    if hookimpl.trylast:
        methods.insert(0, hookimpl)
    elif hookimpl.tryfirst:
        methods.append(hookimpl)
    else:
        # find last non-tryfirst method
        i = len(methods) - 1
        while i >= 0 and methods[i].tryfirst:
            i -= 1
        methods.insert(i + 1, hookimpl)      
```
プラグイン仕様とプラグイン自体はデコレータに特別なインフォメーションがつけられています、これらの特別なインフォメーションによってプラグイン仕様とプラグインを検出し、それらのプロパティを利用して`HookCallerクラス`（プラグイン仕様）と`HookImplクラス`（プラグイン自体）を初期化します。最後に`_add_hookimpl`メソッドを介してバインドします。

# プラグインロジックの具体的な呼び出し方
最初の`pluggy`を試したサンプルコードから見ると、`myhook`プラグインメソッドは`pm.hook.myhook（a = 10、b = 20）`メソッドを介してよびだされることがわかります。具体的に言えば、`PluginManager.hook`実際には`_HookRelay`クラスであり、`_HookRelay`クラスは中身の何もないクラスです。`add_hookspecs`メソッドと`register`メソッドの操作により、`_HookRelay`クラスは`myhook`という名前のプロパティが追加され、このプロパティは`_HookCaller`クラスのインスタンスに対応しています。`pm.hook.myhook（a = 10、b = 20）`も実際に`_HookCaller .__ call__`を呼び出しています。
```python
def __call__(self, *args, **kwargs):
    # ---略---
    if self.spec and self.spec.argnames:
        # プラグイン仕様で受け取れるパラメーターとプラグイン自体が受け取れるパラメーターと同じかどうかを計算する。
        notincall = (
            set(self.spec.argnames) - set(["__multicall__"]) - set(kwargs.keys())
        )
        if notincall:
            # ---略---
    # 呼び出す
    return self._hookexec(self, self.get_hookimpls(), kwargs)
```
`__call__`メソッドは主にプラグイン仕様とプラグイン自体が一致するかどうかを判別し、そして`self._hookexec`メソッドをよびだして実際に実行することです。
分析を通じて、完全な呼び出しチェーンは
> `_HookCaller._hookexec` -> `PluginManager._inner_hookexec` -> `_HookCaller.multicall` -> `callers`ファイルの`_multicallメソッド`　

となります。

`_multicall`メソッドコードのコア部分
```python
def _multicall(hook_impls, caller_kwargs, firstresult=False):
    # ---略---
            for hook_impl in reversed(hook_impls):
                try:
                    # myhookメソッドを呼び出す
                    args = [caller_kwargs[argname] for argname in hook_impl.argnames]
                # ---略---
                
                # プラグインでyeildが使用されている場合は、このように呼び出す
                if hook_impl.hookwrapper:
                   try:
                       gen = hook_impl.function(*args)
                       next(gen)  # first yield
                       teardowns.append(gen)
                   except StopIteration:
                       _raise_wrapfail(gen, "did not yield")
```

これで、`pluggy`のコアロジックは終わりました。

# 終わり
最後まで読んでいただき、ありがとうございます。この記事はただ`pluggy`の最もシンプルな使用方法とそのロジックを述べましたが、`pluggy`にはまだいくつかの重要な部分があります。興味がありましたらはソースコードで詳細をご覧ください。

> 参考: [pluggy GitHub](https://github.com/pytest-dev/pluggy)
