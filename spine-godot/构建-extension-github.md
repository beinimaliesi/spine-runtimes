
# spine-godot GDExtension 构建

本文说明 Spine 4.3 runtime 面向 Godot 4.7.2 的 GDExtension 构建流程。

## 利用github的Action做统一构建构建

- .github/workflows/spine-godot-extension-v4.yml, 和 .github/workflows/spine-godot-extension-v4-all.yml 文件包含了 GitHub 工作流(workflow), 它们能通过GitHub Actions构建 spine-godot GDExtension. 相比本地编译过程要更加简单

- 手动触发 spine-godot-extension-v4-all 工作流即可


## 版本

- Godot：`4.7.2-stable`
- godot-cpp：`master`
- godot-cpp API：`4.7`


## 参考文档

- https://zh.esotericsoftware.com/spine-godot#%E7%BC%96%E8%AF%91-GDExtension