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
