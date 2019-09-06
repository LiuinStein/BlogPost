### 0x00 工具

[Atom](https://atom.io/)是GitHub出品的一款编辑器，号称是A hackable text editor for the 21st Century，其最大的亮点当属其插件系统，像Chrome浏览器一样，经由插件(package)的增强可以实现多彩的功能。当然，除了明显的性能的问题，尤其是内存占用上，我对他没有什么别的坏的意见。真是搞不懂，同样是基于electron的vscode为什么就没有这么明显的性能问题。我对比过，打开同样的项目，vscode的内存占用只有atom的五分之一。我估计这可能由于vscode基于TypeScript的原因。我看来atom对vscode的唯一的优势就是颜值高，当然是配置了插件主题以后的颜值高。

### 0x01 基于Atom搭建$\LaTeX$编辑器

我们要用到的核心的插件如下：

[latex](https://atom.io/packages/latex)：用于在Atom下调用外部`tex`编译器编译`tex`文件

> 这个插件是不带latex编译器的，需要你自己单独安装，推荐使用`tex live`，`tex live`如果完全安装的话，体积是十分庞大的，建议部分安装，以后如果有什么需要的包，可以使用其自带的`tex live manager`工具一键式安装即可。
>
> 如果你没有把`tex live`配置到系统PATH路径下的话，需要你手动在latex插件的设置界面设置`TeX Path`选项路径，例如：
>
> ```
> D:\texlive\2018\bin\win32\
> ```
>
> 注意：此路径下一定要包含`latexmk`等编译工具

[language-latex](https://atom.io/packages/language-latex)：用于在Atom中高亮latex代码

[pdf-view](https://atom.io/packages/pdf-view)：很遗憾Atom没有自带PDF阅读器，如果你不装这个的话，他会自动调用你系统默认的PDF阅读器来打开编译生成的PDF文件

> pdf-view这个插件有一个缺陷：如果你生成的PDF所在的目录路径中带有中文的话，将无法使用`SyncTeX`功能，`SyncTeX`功能用于PDF到latex代码的反向定位，这个文件在2017年11月份就有人在其GitHub提出issue，且被作者标记为bug，但至今仍未被修复，具体进度可以参考：
>
> [pdf-view can't approved Chinese path #194](https://github.com/izuzak/atom-pdf-view/issues/194)
>
> [Synctex does not work with japanese text #195](https://github.com/izuzak/atom-pdf-view/issues/195)
>
> 但是此处仍然推荐使用pdf-view，latex插件所支持的pdf阅读器，及其对比信息可参考：[Supported Viewers](https://github.com/thomasjo/atom-latex/wiki/Supported-Viewers)