---
title: kubernetes源码赏析——kubectl
date: 2019-07-31 10:28:02
tags: [kubectl,Observer Design Pattern]
categories: kubernetes
---
## before you begin
You need to understand the following knowledge:
- Cobra
- [Observer Design Pattern](https://design-patterns.readthedocs.io/zh_CN/latest/behavioral_patterns/observer.html)
<!--more-->
## A Tour of kubectl to API Server

我们要追踪的命令是kubectl delete -f——它会从文件删除K8s资源,如下将关键源码以逻辑顺序贴出：
### 主要流程
```golang
Run: func(cmd *cobra.Command, args []string) {
			o := deleteFlags.ToOptions(nil, streams)
			cmdutil.CheckErr(o.Complete(f, args, cmd))
			cmdutil.CheckErr(o.Validate())
			cmdutil.CheckErr(o.RunDelete())
        },
```
```golang
func (o *DeleteOptions) RunDelete() error {
	return o.DeleteResult(o.Result)
}

func (o *DeleteOptions) DeleteResult(r *resource.Result) error {
    r.Visit(func(info *resource.Info, err error) error {
        response, err := o.deleteResource(info, options)
    }
}

func (o *DeleteOptions) deleteResource(info *resource.Info, deleteOptions *metav1.DeleteOptions) (runtime.Object, error) {
    deleteResponse, err := resource.NewHelper(info.Client, info.Mapping).DeleteWithOptions(info.Namespace, info.Name, deleteOptions)
}
```

```golang
func (m *Helper) DeleteWithOptions(namespace, name string, options *metav1.DeleteOptions) (runtime.Object, error) {
	return m.RESTClient.Delete().
		NamespaceIfScoped(namespace, m.NamespaceScoped).
		Resource(m.Resource).
		Name(name).
		Body(options).
		Do().
		Get()
}
```
### 实现核心功能的关键代码——观察者模式
```golang
func (r *Result) Visit(fn VisitorFunc) error {
	err := r.visitor.Visit(fn)
}
```

```golang
func (o *DeleteOptions) Complete(f cmdutil.Factory, args []string, cmd *cobra.Command) error {
    o.Result := f.NewBuilder().
		Unstructured().
		ContinueOnError().
		NamespaceParam(cmdNamespace).DefaultNamespace().
		FilenameParam(enforceNamespace, &o.FilenameOptions).
		LabelSelectorParam(o.LabelSelector).
		FieldSelectorParam(o.FieldSelector).
		SelectAllParam(o.DeleteAll).
		AllNamespaces(o.DeleteAllNamespaces).
		ResourceTypeOrNameArgs(false, args...).RequireObject(false).
		Flatten().
		Do()
}
```
```golang
func (b *Builder) Do() *Result {
    r := b.visitorResult()
    return r
}

func (b *Builder) visitorResult() *Result {
    // visit items specified by paths
	if len(b.paths) != 0 {
		return b.visitByPaths()
	}
}

func (b *Builder) visitByPaths() *Result {
    visitors = EagerVisitorList(b.paths)
    result.visitor = visitors
}
```

```golang
type EagerVisitorList []Visitor

func (l EagerVisitorList) Visit(fn VisitorFunc) error {
	errs := []error(nil)
	for i := range l {
		if err := l[i].Visit(func(info *Info, err error) error {
			if err := fn(info, nil); err != nil {
			return nil
		})
	}
}
```

```golang
func (b *Builder) FilenameParam(...) *Builder {
    paths := filenameOptions.Filenames
    for _, s := range paths {
        b.Path(recursive, s)
    }
}

func (b *Builder) Path(recursive bool, paths ...string) *Builder {
    for _, p := range paths {
        visitors, err := ExpandPathsToFileVisitors(b.mapper, p, recursive, FileExtensions, b.schema)
        b.paths = append(b.paths, visitors...)
    }
}

func ExpandPathsToFileVisitors(...) []Visitor{
    visitor := &FileVisitor{
            Path:          path,
            StreamVisitor: NewStreamVisitor(nil, mapper, path, schema),
    }
    visitors = append(visitors, visitor)
    return visitors
}
// NewStreamVisitor is a helper function that is useful when we want to change the fields of the struct but keep calls the same.
func NewStreamVisitor(r io.Reader, mapper *mapper, source string, schema ContentValidator) *StreamVisitor {
	return &StreamVisitor{
		Reader: r,
		mapper: mapper,
		Source: source,
		Schema: schema,
	}
}


// Visit in a FileVisitor is just taking care of opening/closing files
func (v *FileVisitor) Visit(fn VisitorFunc) error {
	var f *os.File
	if v.Path == constSTDINstr {
		f = os.Stdin
	} else {
		f, err = os.Open(v.Path)
	}
	v.StreamVisitor.Reader = transform.NewReader(f, utf16bom)
	return v.StreamVisitor.Visit(fn)
}

func (v *StreamVisitor) Visit(fn VisitorFunc) error {
    d := yaml.NewYAMLOrJSONDecoder(v.Reader, 4096)
	for {
		ext := runtime.RawExtension{}
		if err := d.Decode(&ext); err != nil {
		}
		ext.Raw = bytes.TrimSpace(ext.Raw)
		info, err := v.infoForData(ext.Raw, v.Source)
		if err := fn(info, nil); err != nil {
			return err
		}
	}
}

```

## 思考

上述代码流程采用关键有效原则，省去细枝末节，以功能逻辑顺序贴出，虽缺少中文注释，但本篇教程的面向对象是具有一定代码阅读能力的同学。众所周知，golang特有的接口机制在k8s源码中大量使用，对源码理解造成了一定的困扰。相信在阅读过源码尚有疑惑的看过上述代码流程之后定会豁然开朗。但是这还不是此文的主要目的！优秀的源码是经过千锤百炼的，经受过无数次生产环境恶劣的性能测试。尤其对于k8s源码这种级别的代码，更是golang语言界的杰出代表之作！其中有很多值得我们借鉴和学习的地方，理解其源码功能为次要，主要的是学习其精髓，思考为什么这样设计？这样设计的好处在哪里？程序的扩展性如何？如果是自己设计会怎样做？如此才能不断成长，因为源码是最优秀的老师。

此文的源码分析基于kubectl delete -f，各位同学可以基于delete其它的选项进行一下kubectl源码程序的扩展性进行分析。理解观察者模式在其中的应用。
![](/image/kubectl.jpeg)
