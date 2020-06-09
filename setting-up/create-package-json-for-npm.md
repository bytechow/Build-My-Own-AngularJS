### 创建 NPM 所需的 package.json（Create package.json for NPM）

要用 NPM，我们就需要创建 `package.json` 文件。NPM 能够从这个文件获取项目的一些基本信息，同时更重要的是，它说明了项目依赖哪些 NPM 包。

我们需要在项目根目录下创建 `package.json`文件，然后在文件里加入一些基本的元信息——项目名称和版本：

_package.json_

```json
{
  "name": "my-own-angularjs",
  "version": "0.1.0"
}
```

这样，我们就已经创建好一个小型 JavaScript 项目所需要的目录和文件了。