在进入最后的“指令”主题之前，我们还需要一个重要的服务——$http。顾名思义，这个服务主要负责 AngularJS 应用中的 HTTP 通信。

大部分 Angular 应用都会使用像 ngResource 之类的 API 工具，这些工具都直接或间接使用了 $http。之后等我们讲到指令的时候，还会发现 AngularJS 使用 $http 服务来加载 HTML 模板。

本章，我们会看看 $http 服务做了些什么，它是如何利用 $rootScope 和 $q 服务来实功能的。

