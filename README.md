providejs
=========

the first implementation:

```
!function (root) {

    var REQUIRE_PATH = './requirejs.js',
        DOT_RE = /\/\.\//g,
        DOUBLE_DOT_RE = /\/[^/]+\/\.\.\//,
        DOUBLE_SLASH_RE = /([^:/])\/\//g;
    var _head = document.getElementsByTagName('head')[0],
        _base = './',
        _currDefines = [],
        _defines = [],
        _requires = [],
        _loadNum = 0,
        _requireReady = false,
        _requireFactory;

    function _normalize(base, id) {
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

    /* Class */
    /**
     * Cache
     * @class
     * @static
     */
    var Cache = function () {
        var map = {};
        return {
            /**
             * get
             * @param {String} key
             * @returns value
             */
            get: function (key) {
                return map[key];
            },
            /**
             * set
             * @param {String} key
             * @param {Object} value
             * @returns isSuccess
             */
            set: function (key, value) {
                if (key in map) return false;
                map[key] = value;
                return true;
            }
        };
    }();

    function Loader(url, name) {
        _isUnnormalId(url) || (name = url) && (url = requirejs.paths[url] + '.js');
        this.load(url, name);
        this.path = url;
        this.succList = [];
        this.failList = [];
    }
    Loader.prototype = {
        constructor: Loader,
        /**
         * load
         * @param {String} url
         */
        load: function (url, name) {
            var 
                node = document.createElement('script'),
                self = this;
            node.addEventListener('load', _onload, false);
            node.addEventListener('error', _onerror, false);
            node.type = 'text/javascript';
            node.async = 'async';
            node.src = url;
            _head.appendChild(node);
            function _onload() {
                _onend();
                _currDefines.forEach(function (args) {
                    if (typeof args[0] !== 'string' || args.length === 1) {
                        args.unshift(name);
                    } else {
                        // ERROR ?
                        args[0] = _resolvePath(name, args[0]).replace(/\.js$/, '');
                    }
                    _defines.push(args);
                });
                _currDefines.length = 0;
                return self.done();
            }
            function _onerror() {
                _onend();
                _head.removeChild(node);
                if (_base && !~url.indexOf(_localBase)) {
                    return self.load(url.replace(_base, _localBase));
                } else {
                    return self.down();
                }
            }
            function _onend() {
                node.removeEventListener('load', _onload, false);
                node.removeEventListener('error', _onerror, false);
            }
        },
        /**
         * _push
         * @private
         * @param {Array} list
         * @param {Function} cb
         */
        _unshift: function (list, cb) {
            if (!list.some(function (item) { return item === cb }))
                return list.unshift(cb);
        },
        /**
         * succ
         * @param {Function} cb
         */
        succ: function (cb) {
            return this._unshift(this.succList, cb);
        },
        /**
         * fail
         * @param {Function} cb
         */
        fail: function (cb) {
            return this._unshift(this.failList, cb);
        },
        /**
         * done
         */
        done: function () {
            this.loaded = true;
            this.succList.forEach(function (cb) {
                return cb();
            });
        },
        /**
         * down
         */
        down: function () {
            this.failList.forEach(function (cb) {
                return cb();
            });
        }
    }

    function require(deps, succ, fail) {
        var fired = false, 
            _deps = deps.slice(0);
        _requires.push(arguments);
        _loadNum++;
        function _checkDeps() {
            var res = [];
            _deps.forEach(function (dep, i) {
                var 
                    path = _normalize(_base, dep),
                    loader = Cache.get(path);
                if (!loader) {
                    loader = new Loader(path, dep);
                    Cache.set(path, loader);
                }
                if (loader && loader.loaded) {
                    return;
                } else {
                    res.push(dep);
                }
                loader.succ(_checkDeps);
                fail && loader.fail(fail);
            });
            _deps = res;
            // make sure success callback will not trigger multiple times
            if (!_deps.length && !fired) {
                fired = true;
                _loadNum--;
                ready();
            }
        }
        _checkDeps();
    }

    function define() {
        _currDefines.push([].slice.apply(arguments));
    }

    function ready() {
        if (!_requireReady || !_requires.length || _loadNum) return;
        root.require = root.define = undefined;
        _requireFactory(root);
        _defines.forEach(function (args) {
            root.define.apply(this, args);
        });
        _requires.forEach(function (args) {
            root.require.apply(this, args);
        });
    }

    function preload(factory) {
        _requireFactory = factory;
    }

    // init require.js
    (new Loader(REQUIRE_PATH))
        .succ(function () {
            _requireReady = true;
            ready();
        });

    root.preload = preload;
    root.require = require;
    return root.define = define;
}(this);
```
