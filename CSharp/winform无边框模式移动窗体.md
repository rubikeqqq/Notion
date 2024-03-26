# winform无边框模式移动窗体



```c#
 private bool isDrag = false;
 private int enterX = 0; //记录鼠标按下时的X坐标
 private int enterY = 0; //记录鼠标按下时的Y坐标
 

 private void Form1_MouseDown( object sender, MouseEventArgs e )
 {
     enterX = e.X;
     enterY = e.Y;
     isDrag = true;
 }

 private void Form1_MouseUp( object sender, MouseEventArgs e )
 {
     enterX = 0;
     enterY = 0;
     isDrag = false;
 }

 private void Form1_MouseMove( object sender, MouseEventArgs e )
 {
     if(isDrag )
     {
         //鼠标现在的坐标 - 鼠标按下时的坐标 = 鼠标移动的坐标
         //将窗口的左边加上 鼠标移动的距离
         this.Left += e.X - enterX;
         this.Top += e.Y - enterY;  
     }
 }
```

