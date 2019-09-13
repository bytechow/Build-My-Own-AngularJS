## 创建 NPM 所需的 package.json（Create package.json for NPM）

要用 NPM，我们就需要创建一个名为`package.json`的文件。NPM 能够从这个文件获知项目的一些基本信息，同时还有更重要的，告知项目依赖了哪些 NPM 包。

我们在项目根目录下创建一个基本的`package.json`文件，然后在里面加入一些基本的元信息——项目名称和版本：

_package.json_

```json
{
  "name": "my-own-angularjs",
  "version": "0.1.0"
}
```

现在我们已经拥有一个小型 JavaScript 项目所需要的目录和文件了。