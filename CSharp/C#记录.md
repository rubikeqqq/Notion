# C#记录

## 数据类型

+ 0d代表为0的double类型数值

### 字符串

``````c#
string str = "aaatbbbscctdd";
string[] strArray = str.split(new char[]{'t',2});  //只切割成2份
str[0] = 'aaa';
str[1] = 'bbbscctdd';

//string.Format
string str = string.Format("{0}:{1:F3}:{2:F2}:{3:F1}","OK",3.1211,1.13333,2.43132)
//1:F2是保留2位小数
``````



## OpenFileDialog

**注意点**

1. 初始目录

   `ofd.InitialDirectory = @“C:\\” 注意后面还要加一个\`

2. 筛选器

   `ofd.Filter = "文本文件|*.txt*|C#文件|*.cs*|所有文件|*.*";`

3. 是否保存最后选择的目录

   `ofd.RestoreDirectory = true;`

4. 右下角文件格式第一次显示

   `ofd.FilterIndex = 1;`

5. 最后显示多种图片

   `ofd.Filter = "所有文件|*.*|bmp文件|*.bmp|jpg文件|*.jpg"`







## 线程

```c#
//将方法排入队列以便执行。此方法在有线程池变得可用时执行。
ThreadPool.QueueUserWorkItem(CheckCommunicationConnect)
```



## 事件

```c#
//1先声明一个public类型的委托
//2在类中声明一个以此委托类型为返回值的事件event
//3在调用的方法以On开头,比如OnChanged()
//4在此方法中调用事件Invoke() 或者直接调用
//5在调用之前判断一下，当此事件不为null时调用
//6在其他类对此类的事件进行注册以后，在类型的事件就绑定了当前类的方法。
```



## 异常

```c#
//我们自己可以在方法中调用这个，当进入这个的时候，会抛出异常，到时候我们就可以用try语句抓到这个异常，便于我们自己进行判断
throw new Exception("值的范围不合法");
```





## DateTime

``````c#
TimeSpan ts = DateTime.Now - _initializationTime;
//这里的timespan 表示一个时间间隔
//我们可以在程序开始的时候给_initializationTime赋一个初始值，然后2个相减，就可以获得程序运行了多少时间了
``````





## 程序运行

``````c#
//1如果在属性中实例化对象,则此属性的对象会在这个类的构造函数执行之前先实例化
//2 获取该应用程序关联的产品版本 
Application.ProductVersion
//3 获取启动应用程序的可执行文件的路径，包含可执行文件的名称
Application.ExecutablePath 
``````



## 文件、路径

//将2个字符串组合成一个路径

Path.Combine(string path1,string path2)



## 语法糖

//这句表示如果socketClient不为null时执行Close的方法

socketClient?.Close();





## 关键字

###params

params 是C#开发语言中关键字， params主要的用处是在给函数传参数的时候用，就是当函数的参数不固定的时候。 在方法声明中的 params 关键字之后不允许任何其他参数，并且在方法声明中只允许一个 params 关键字。 关于参数数组，需掌握以下几点。

（1）若形参表中含一个参数数组，则该参数数组必须位于形参列表的最后；

（2）参数数组必须是一维数组；

（3）不允许将params修饰符与ref和out修饰符组合起来使用；

（4）与参数数组对应的实参可以是同一类型的数组名，也可以是任意多个与该数组的元素属于同一类型的变量；

（5）若实参是数组则按引用传递，若实参是变量或表达式则按值传递。

（6）用法：可变的方法参数，也称数组型参数，适合于方法的参数个数不知的情况，用于传递大量的数组集合参数；当使用数组参数时，可通过使用params关键字在形参表中指定多种方法参数，并在方法的参数表中指定一个数组，

形式为：方法修饰符　返回类型　方法名（params　类型［］　变量名） 如带有参数的SQL 语句，不同的表的字段数量也不同， 当你更新修改的时候就可以用。例如：



# 窗体

##关闭窗体时提示 

``````c#
 private void Form1_FormClosing(object sender, FormClosingEventArgs e)
        {
            if (DialogResult.Yes == MessageBox.Show("确定退出应用软件吗？", "退出确认", MessageBoxButtons.YesNo, MessageBoxIcon.Question, MessageBoxDefaultButton.Button1, MessageBoxOptions.DefaultDesktopOnly))
            {
               
                this.FormClosing -= new FormClosingEventHandler(this.Form1_FormClosing);
             
                //Process.GetCurrentProcess().Kill();
            }
            else
            {
                e.Cancel = true;
            }
        }

``````

## 跨线程显示

``````c#
delegate void DelegateLableDisplay(int index,Color c,string str);
private void ButtonDIsplay(int index,Color c,string str)
{
    if(this.InvokeRequired){
        this.Invoke(new DelegateLableDisplay(ButtonDisplay),new object[]{index,c,str});
        return;
    }
    if(index>=0&&index<Global.btnShowLength){
        Global.btnShow[index].BackColor = c;
        Global.btnShow[index].Text = str;
    }
}
``````





## DataGridViewComboBoxCell

原因：

 传递给DataGridView#DataGridViewComboBoxCell的值类型与DataGridViewComboBoxCell要求的数据类型不符，传递的是Int32但实际要求的是System.String. 

解决：

修改传递给DataGridView数据格式，其实也可以在DataGridViewComboBoxCell端进行转换

