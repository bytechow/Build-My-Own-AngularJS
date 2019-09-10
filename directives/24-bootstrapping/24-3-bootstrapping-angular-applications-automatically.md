### 自动启动 Angular 应用（Bootstrapping Angular Applications Automatically）

对于应用开发者来说，手动启动 Angular 应用并不困难：我们只需要调用一个函数就可以了。但这种启动方式并不是 Angular 应用最常用的启动方式。还有一种更简便的启动方法，我们只需要在 HTML 元素中加入一个`ng-app`属性就可以了。当页面加载时，Angular 会自动查找带有该属性的元素，并在这个元素的基础上启动应用。这就是所谓的_自动启动_。

在开发这个功能的时候，我们不打算使用之前的“测试先行”的开发模式，不写单元测试而直接写代码。这是因为自动启动是与页面的加载事件绑定在一起的，使用单元测试来对齐进行测试有一定的难度。我们需要设计一些特殊的测试架构来确保正确地完成这件事。

我们会把测试放到下一节，下一节我们会为我们的框架构建一个成熟的测试应用。我们会用自动启动的方式运行这个应用，这样我们就能确保已有的功能都正常运行。

自动启动会从一个普通的、历史悠久的 jQuery [DOM ready](https://api.jquery.com/ready/)回调函数开始。我们会把它放在`bootstrap.js`中：

_src/bootstrap.js_

```js
// 'use strict';

// var $ = require('jquery');
// var publishExternalAPI = require('./angular_public');
// var createInjector = require('./injector');

// publishExternalAPI();

// window.angular.bootstrap = function(element, modules, config) {
//   var $element = $(element);
//   modules = modules || [];
//   config = config || {};
//   modules.unshift(['$provide', function($provide) {
//     $provide.value('$rootElement', $element);
//   }]);
//   modules.unshift('ng');
//   var injector = createInjector(modules, config.strictDi);
//   $element.data('$injector', injector);
//   injector.invoke(['$compile', '$rootScope', function($compile, $rootScope) {
//     $rootScope.$apply(function() {
//       $compile($element)($rootScope);
// }); }]);
//   return injector;
// };

$(document).ready(function() {

});
```

我们在这里要做的是，如果文档中有一个元素含有`ng-app`属性，程序会找到它。实际上，除了最常用的`ng-app`以外，我们还可以使用其他替代语法：

- ng-app
- data-ng-app
- ng:app
- x-ng-app

我们会把这几种语法的前缀都放到一个数组中，然后在 document.ready 处理器中对他们进行遍历：

_src/bootstrap.js_

```js
var ngAttrPrefixes = ['ng-', 'data-ng-', 'ng:', 'x-ng-'];
$(document).ready(function() {
  _.forEach(ngAttrPrefixes, function(prefix) {

  });
});
```

此时，我们需要引入 LoDash 依赖：

```js
'use strict';

var $ = require('jquery');
var _ = require('lodash');
var publishExternalAPI = require('./angular_public');
var createInjector = require('./injector');

// ...
```

对每一种前缀，我们都会构建一个 [CSS 属性选择器](https://developer.mozilla.org/en-US/docs/Web/CSS/Attribute_selectors)，这个选择器可以帮助我们定位到对应属性所在的元素：

```js
var ngAttrPrefixes = ['ng-', 'data-ng-', 'ng:', 'x-ng-'];
$(document).ready(function() {
  _.forEach(ngAttrPrefixes, function(prefix) {
    var attrName = prefix + 'app';
    var selector = '[' + attrName.replace(':', '\\:') + ']';
  });
});
```

注意，要让`ng:`前缀对应的选择器正常运作，我们需要对`:`字母进行转移，这样它才不会被解析成是一个 CSS 伪类选择器。

现在，我们可以试着去查找符合这些选择器的元素了。我们会使用 DOM 标准的 [querySelector](https://developer.mozilla.org/en-US/docs/Web/API/Document/querySelector) API，它会返回与这个选择器匹配的第一个元素。我们只需要一个元素，所以如果我们已经找到了带有其中一种前缀属性的元素，那就直接跳过后面的前缀：

```js
var ngAttrPrefixes = ['ng-', 'data-ng-', 'ng:', 'x-ng-'];
$(document).ready(function() {
  var foundAppElement;
  _.forEach(ngAttrPrefixes, function(prefix) {
    var attrName = prefix + 'app';
    var selector = '[' + attrName.replace(':', '\\:') + ']';
    var element;
    if (!foundAppElement &&
        (element = document.querySelector(selector))) {
      foundAppElement = element;
    }
  }); 
});
```

当我们在 DOM 中使用`ng-app`属性，也可能会把这个属性的值设置为一个_模块名称_。这个模块会控制用户自定义的模块加载到哪里。我们需要把这个模块名称也抓取下来：

```js
$(document).ready(function() {
  var foundAppElement, foundModule;
  // _.forEach(ngAttrPrefixes, function(prefix) {
  //   var attrName = prefix + 'app';
  //   var selector = '[' + attrName.replace(':', '\\:') + ']';
  //   var element;
    if (!foundAppElement && (element = document.querySelector(selector))) {
      // foundAppElement = element;
      foundModule = element.getAttribute(attrName);
    } 
  // });
});
```

值得注意的是，不同于手动启动，在自动启动模式下，我们只能定一个模块名称，其他任何模块都必须定义为主模块的依赖模块。

如果我们确实找到了含有自动启动属性的元素，我们需要在这个元素上启动应用。我们只需要调用手动启动时开发的`bootstrap`函数，并提供查找到的元素和可能存在的模块名称作为函数调用时的参数就可以了：

```js
var ngAttrPrefixes = ['ng-', 'data-ng-', 'ng:', 'x-ng-'];
$(document).ready(function() {
  // var foundAppElement;
  // _.forEach(ngAttrPrefixes, function(prefix) {
  //   var attrName = prefix + 'app';
  //   var selector = '[' + attrName.replace(':', '\\:') + ']';
  //   var element;
  //   if (!foundAppElement &&
  //       (element = document.querySelector(selector))) {
  //     foundAppElement = element;
  //   }
  // });
  if (foundAppElement) {
    window.angular.bootstrap(
      foundAppElement,
      foundModule ? [foundModule] : []
    );
  }
});
```

最后，我们也可以通过在自动启动时启用一个额外属性来设定严格的依赖注入模式。首先，我们需要支持对手动启动应用的函数传递第三个参数`config`：

```js
var ngAttrPrefixes = ['ng-', 'data-ng-', 'ng:', 'x-ng-'];
$(document).ready(function() {
  var foundAppElement, foundModule, config = {};
  // _.forEach(ngAttrPrefixes, function(prefix) {
  //   var attrName = prefix + 'app';
  //   var selector = '[' + attrName.replace(':', '\\:') + ']';
  //   var element;
  //   if (!foundAppElement &&
  //     (element = document.querySelector(selector))) {
  //     foundAppElement = element;
  //     foundModule = element.getAttribute(attrName);
  //   }
  // });
  if (foundAppElement) {
    window.angular.bootstrap(
      foundAppElement,
      foundModule ? [foundModule] : [],
      config
    );
  }
});
```

当我们找到元素时，如果元素上有一个`ng-strict-di`属性，我们就为 config 对象添加一个 strictDi 属性。这里，我们也支持使用其他几种属性前缀：

- ng-strict-di
- data-ng-strict-di
- ng:strict-di
- x-ng-strict-di

```js
// var ngAttrPrefixes = ['ng-', 'data-ng-', 'ng:', 'x-ng-'];
$(document).ready(function() {
  // var foundAppElement, foundModule, config = {};
  // _.forEach(ngAttrPrefixes, function(prefix) {
  //   var attrName = prefix + 'app';
  //   var selector = '[' + attrName.replace(':', '\\:') + ']';
  //   var element;
  //   if (!foundAppElement &&
  //     (element = document.querySelector(selector))) {
  //     foundAppElement = element;
  //     foundModule = element.getAttribute(attrName);
  //   }
  // });
  // if (foundAppElement) {
    config.strictDi = _.some(ngAttrPrefixes, function(prefix) {
      var attrName = prefix + 'strict-di';
      return foundAppElement.hasAttribute(attrName);
    });
  //   window.angular.bootstrap(
  //     foundAppElement,
  //     foundModule ? [foundModule] : [],
  //     config
  //   );
  // }
});
```

这样我们就实现了自动启动功能！它主要是关注如何查找正确的 DOM 元素并对该元素进行检查，剩下的都是重用之前实现的手动启动功能的代码。