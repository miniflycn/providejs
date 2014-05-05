providejs
=========

> parallel load js before requirejs onload.

the first implementation:

```
!function (root) {
    var REQUIRE_PATH = './requirejs.js',
        DOT_RE = /\/\.\//g,
        DOUBLE_DOT_RE = /\/[^/]+\/\.\.\//,
        DOUBLE_SLASH_RE = /([^:/])\/\//g,
        BASE = _resolvePath(location.href, './'),
        doc = document,
        head = doc.getElementsByTagName('head')[0],
        defines = [],
        currDefines = [],
        _factory;

    function _normalize(base, id) {
        if (requirejs && requirejs.paths && requirejs.paths[id]) return requirejs.paths[id];
        if (_isUnnormalId(id)) return id;
        if (_isRelativePath(id)) return _resolvePath(base, id) + '.js';
        return id;
    }

    function _isUnnormalId(id) {
        return (/^https?:|^file:|^\/|\.js$/).test(id);
    }

    function _isRelativePath(path) {
        return (path + '').indexOf('.') === 0;
    }

    // reference from seajs
    function _resolvePath(base, path) {
        path = base.substring(0, base.lastIndexOf('/') + 1) + path;
        path = path.replace(DOT_RE, '/');
        while (path.match(DOUBLE_DOT_RE)) {
            path = path.replace(DOUBLE_DOT_RE, '/');
        }
        return path = path.replace(DOUBLE_SLASH_RE, '$1/');
    }

    function load(url, succ, fail) {
        url = _normalize(BASE, url);
        var node = doc.createElement('script');
        node.addEventListener('load', succ, false);
        node.addEventListener('error', fail, false);
        node.type = 'text/javascript';
        node.async = 'async';
        node.src = url;
        head.appendChild(node);
    }

    function require(deps, succ, fail) {
        // load require too
        deps.push(REQUIRE_PATH);
        var l = deps.length;
        for (var i = 0; i < l; i++) {
            load(deps[i], function () {
                _pushDefines(this.src);
                if (--l === 0) return _clear(arguments);
            }, function () {
                l = -1;
                return fail();
            })；
        }
    }

    function define() {
        currDefines.push([].slice.call(arguments, 0));
    }

    function _pushDefines(src) {
        for (var i = 0, l = currDefines, args; i < l; i++) {
            args = currDefines[i];
            if (typeof args[0] === 'string' && args.length > 1) {
                args[0] = _resolvePath(src, args[0]).replace(BASE, './').replace(/\.js$/, '');
            } else {
                args.shift(src.replace(BASE, './').replace(/\.js$/, ''));
            }
            defines.push(args);
        }
        currDefines.length = 0
    }

    function _clear(args) {
        root.define = undefined;
        root.require = undefined;
        _factory(root);
        root.require.apply(root, args);
    }

    function preload(factory) {
        _factory = factory;
    }

    root.define = define;
    root.require = require;
    root.preload = preload;

}(this);
```


加载器的困境
------------

类AMD模块加载器都会遇到下面两个主要问题：

1. 由于假定一个模块对应一个文件，但出于减少HTTP请求考虑，前端需要将多个模块合并成一个文件的时候如何解决？
2. 如何让加载器不阻塞其他模块的加载。

多模块单文件问题
----------------

通常多模块单文件的解决方法是，在每个模块前加载该模块名，如对于最后合并的单文件all.js包含了模块a和模块b，则：
``` javascript
define('./a', function () {});
define('./b', function () {});
```
再比如a模块和b模块原本和all不在同一个目录里，而在all所在目录的a和b目录，则all.js为：
``` javascript
define('./a/a', function () {});
define('./b/b', function () {});
```
所以all.js中的模块命名和是相对于all.js路径的，所以要获取a.js和b.js的全名，必须通过all.js的路径获得。但很可惜，javascript中没有获取all.js的通用方案，只能通过requirejs接口加载all.js脚本时候，将该脚本的路径作为上下文，带到define函数中。所以从requirejs实现技术来讲，如果使用requirejs的话，只能通过data-mian或者require方法来加载脚本，这样才能得到模块正确的全名，来保证模块定义正确性。

可是我不希望加载器阻塞模块加载
------------------------------

所以我们会在html中将模块通过script标签加载，则这些模块就需要写相对于base url的路径名，例如base url为[http://ke.qq.com/js](http://ke.qq.com/js)，而all.js文件路径是[http://ke.qq.com/js/common/all.js](http://ke.qq.com/js/common/all.js)，则all.js最后写法为：
``` javascript
define('./tool/a/a', function () {});
define('./tool/b/b', function () {});
```
但这种方法有下面几个问题：
* 很丑，不好看，居然要把打包后的文件路径也写上
* 当想用require(['./all'], function () {})来引用all.js的时候，a和b模块全名居然就变成了./tool/tool/a/a和./tool/tool/b/b
* 如果对于另一个并行加载的c模块，其依赖a模块，则很有可能a模块会加载两次，甚至会报错。因为c加载完毕的时候a模块可能还没定义，所以模块加载器只好去再加载a模块一次，如果服务器没有对应的a.js则会报错。

还有一只种方案是把加载器打包到html里面，则加载器本身随html一同加载了，但加载器比较大，这似乎又是一个性能问题。

如实第三种方案 —— providejs，诞生了。前置模块加载器就是打包到html里面，但只是将define延迟，等到真正的模块加载器加载完毕后，再去define对应的模块。
