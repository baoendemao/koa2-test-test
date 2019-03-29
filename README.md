### co源码学习 
* co源码： https://github.com/tj/co 
* 简介：
  * （1) 传入generator，将内部yield同步执行
  * （2）co只允许yield a function, promise, generator, array, object，且array或者object里值也必须是这些类型
* 原理：
  * （1）generator + promise
  * （2）将所有的yield返回，转成promise，并执行promise.then。 
  * （3）递归的next()
* co源代码注释如下：
```

/**
 * slice() reference.
 * 
 * slice引用，方便后面截取数组
 */
var slice = Array.prototype.slice;

/**
 * Expose `co`.
 */
module.exports = co['default'] = co.co = co;

/**
 * Wrap the given generator `fn` into a
 * function that returns a promise.
 * This is a separate function so that
 * every `co()` call doesn't create a new,
 * unnecessary closure.
 *
 * @param {GeneratorFunction} fn
 * @return {Function}
 * @api public
 */

co.wrap = function (fn) {
  createPromise.__generatorFunction__ = fn;
  return createPromise;
  function createPromise() {
    return co.call(this, fn.apply(this, arguments));
  }
};

/**
 * Execute the generator function or a generator
 * and return a promise.
 *
 * @param {Function} fn
 * @return {Promise}
 * @api public
 */
// 入参: generator函数或者generator对象
// 返回: promise对象
function co(gen) {
  // 获取当前执行上下文, 比如调用方式：co.call(this, obj)
  var ctx = this;

  // args数组保存传递的参数
  var args = slice.call(arguments, 1);

  // we wrap everything in a promise to avoid promise chaining,
  // which leads to memory leak errors.
  // see https://github.com/tj/co/issues/180
  
  // co函数返回promise对象
  return new Promise(function(resolve, reject) {

    // 如果传入的是generator函数， 则apply调用，gen重新赋值为generator对象
    if (typeof gen === 'function') gen = gen.apply(ctx, args);
    
    // 如果gen不存在 或者 gen不是generator函数，则直接返回 
    if (!gen || typeof gen.next !== 'function') return resolve(gen);

    // 调用next
    onFulfilled();

    /**
     * @param {Mixed} res
     * @return {Promise}
     * @api private
     */
    // 递归调用next()
    function onFulfilled(res) {
      var ret;
      try {
        // generator对象的next()方法，每次调用后停留在下一个yield
        ret = gen.next(res);
      } catch (e) {
        return reject(e);
      }
      // 处理next()的结果
      next(ret);
      return null;
    }

    /**
     * @param {Error} err
     * @return {Promise}
     * @api private
     */

    function onRejected(err) {
      var ret;
      try {
        ret = gen.throw(err);
      } catch (e) {
        return reject(e);
      }
      next(ret);
    }

    /**
     * Get the next value in the generator,
     * return a promise.
     *
     * @param {Object} ret
     * @return {Promise}
     * @api private
     */
    // 递归, 处理next()的value
    function next(ret) {

      // generator对象的next()是否已经执行完, 即递归的出口
      if (ret.done) return resolve(ret.value);

      // 将ret.value转换成promise对象
      var value = toPromise.call(ctx, ret.value);

      // 执行promise的then方法
      // 如果正确, 则执行onFulfilled, 且将上次next执行的结果传入onFullfilled, 继续next()递归
      // 如果错误，则执行onRejected，调用generator对象的throw将错误抛出
      if (value && isPromise(value)) return value.then(onFulfilled, onRejected);

      return onRejected(new TypeError('You may only yield a function, promise, generator, array, or object, '
        + 'but the following object was passed: "' + String(ret.value) + '"'));
    }
  });
}

/**
 * Convert a `yield`ed value into a promise.
 *
 * @param {Mixed} obj
 * @return {Promise}
 * @api private
 * 
 * 将next()返回的value，转成promise
 */
function toPromise(obj) {
  if (!obj) return obj;

  // 如果是promise，则直接返回
  if (isPromise(obj)) return obj;

  // 如果是generator函数 或者 generator对象， 再次调用co，生成promise
  if (isGeneratorFunction(obj) || isGenerator(obj)) return co.call(this, obj);

  // 函数，则调用thunkToPromise转成promise
  if ('function' == typeof obj) return thunkToPromise.call(this, obj);

  // 数组，则调用arrayToPromise转成promise
  if (Array.isArray(obj)) return arrayToPromise.call(this, obj);

  // 纯对象, 则调用objectToPromise转成promise
  if (isObject(obj)) return objectToPromise.call(this, obj);

  return obj;
}

/**
 * Convert a thunk to a promise.
 *
 * @param {Function}
 * @return {Promise}
 * @api private
 * 
 * 函数转为promise
 * 
 */
function thunkToPromise(fn) {
  var ctx = this;
  return new Promise(function (resolve, reject) {
    // call调用该函数
    fn.call(ctx, function (err, res) {
      if (err) return reject(err);
      if (arguments.length > 2) res = slice.call(arguments, 1);
      resolve(res);
    });
  });
}

/**
 * Convert an array of "yieldables" to a promise.
 * Uses `Promise.all()` internally.
 *
 * @param {Array} obj
 * @return {Promise}
 * @api private
 * 
 * 数组转成promise
 */
function arrayToPromise(obj) {
  return Promise.all(obj.map(toPromise, this));
}

/**
 * Convert an object of "yieldables" to a promise.
 * Uses `Promise.all()` internally.
 *
 * @param {Object} obj
 * @return {Promise}
 * @api private
 * 
 * obj对象转成promise, 
 * 且遍历obj对象的所有的key，都调用toPromise函数，转成promise
 * 
 */
function objectToPromise(obj){
  var results = new obj.constructor();

  // 获取obj对象的所有的key
  var keys = Object.keys(obj);
  var promises = [];

  for (var i = 0; i < keys.length; i++) {
    var key = keys[i];
    var promise = toPromise.call(this, obj[key]);
    if (promise && isPromise(promise)) defer(promise, key);
    else results[key] = obj[key];
  }

  return Promise.all(promises).then(function () {
    return results;
  });

  function defer(promise, key) {
    // predefine the key in the result
    results[key] = undefined;
    promises.push(promise.then(function (res) {
      results[key] = res;
    }));
  }
}

/**
 * Check if `obj` is a promise.
 *
 * @param {Object} obj
 * @return {Boolean}
 * @api private
 * 
 * 判断obj是否是promise
 * 判断依据：obj.then函数
 * 
 */
function isPromise(obj) {
  return 'function' == typeof obj.then;
}

/**
 * Check if `obj` is a generator.
 *
 * @param {Mixed} obj
 * @return {Boolean}
 * @api private
 * 
 * 判断obj是否是generator对象
 * 判断依据：obj.next函数 和 obj.throw函数
 * 
 */
function isGenerator(obj) {
  return 'function' == typeof obj.next && 'function' == typeof obj.throw;
}

/**
 * Check if `obj` is a generator function.
 *
 * @param {Mixed} obj
 * @return {Boolean}
 * @api private
 * 
 * 判断obj是否是generator函数
 * 判断依据：obj.constructor.name, obj.constructor.displayName 或者 obj.constructor.prototype
 * 
 */
function isGeneratorFunction(obj) {
  var constructor = obj.constructor;
  if (!constructor) return false;
  if ('GeneratorFunction' === constructor.name || 'GeneratorFunction' === constructor.displayName) return true;
  return isGenerator(constructor.prototype);
}

/**
 * Check for plain object.
 *
 * @param {Mixed} val
 * @return {Boolean}
 * @api private
 * 
 * 判断val是否是纯对象
 * 判断依据：val.constructor
 * 
 */
function isObject(val) {
  return Object == val.constructor;
}

```