# UpdateSourceTrigger和Mode

## UpdateSourceTrigger -- 管时机

`UpdateSourceTrigger = PropertyChanged`决定UI改动后，什么时候把值写回源对象。

例如：

```xml
<TextBox Text="{Binding Name, UpdateSourceTrigger=PropertyChanged}" />
```

意思是用户每输入一个字符，就立刻更新到name.
常见值：

* PropertyChanged:属性一变化就写回
* LostFocus:控件失去焦点时才回写
* Explicit:手动调用时才回写

---

## Mode -- 管方向

`Mode = TwoWay` 决定绑定数据流的方向

意思是：

* 源对象 -> UI
* UI -> 源对象

都会同步

例如：

```xml
<TextBox Text="{Binding Name, Mode=TwoWay}" />
```

表示：

* name 改了，TextBox.Text会更新
* 用户改了TextBox.Text,name也会更新

## 两者的核心区别

| 属性                | 作用                           |
| ------------------- | ------------------------------ |
| Mode                | 决定绑定位于哪几个方向上传数据 |
| UpdateSourceTrigger | 决定从UI回写到源对象的时机     |

## 一句话总结

* `Mode = TwoWay`:能不能回写,决定TextBox和name是否双向同步
* `UpdateSourceTrigger = PropertyChanged`:什么时候回写，决定用户每敲一个字是否立刻同步到name
