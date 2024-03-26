当await一个async的异步方法时，不会卡住,但是还是会等待此方法运行结束

```c#
async Task<int> Calculate(int n)
{
	return await Task.Run(new Func<int>(()=>
	{
		int num = 0;
		for(int i = 0;i < n; i++)
		{
			num += i;
			Task.Delay(10).Wait();
		}
		return num;
	}
}

async void button_Click(object sender,EventArgs e)
{
	var t = await Calculate(100);

	textbox1.Text = t.ToString();
}
```

