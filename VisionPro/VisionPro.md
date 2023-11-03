# VisionPro

## 图像转换

ICogImage可以强转CogImage8Grey(CogImage8Grey继承接口ICogImage)



## 读取本地图片

```c#
 private void radButton_Click(object sender, EventArgs e)
        {
            RadioButton rb = sender as RadioButton;
            m_imageIndex = 0;
            switch (rb.Name)
            {
                    //相机取图
                case "radCamera":
                    if (mAcq == null)
                    {
                        radCamera.Checked = false;
                    }
                    else
                    {
                        radCamera.Checked = true;
                    }
                    break;
                    //读取文件夹并且遍历
                case "radFolder":
                    {
                        FolderBrowserDialog fbd = new FolderBrowserDialog();
                        fbd.Description = "请选择文件夹";
                        if (fbd.ShowDialog() == DialogResult.OK)
                        {
                            imageInfoList.Clear();
                            m_folderPath = fbd.SelectedPath;
                            root = new DirectoryInfo(m_folderPath);
                            mfiles = root.GetFiles("*.*");
                            foreach (FileInfo file in mfiles)
                            {
                                imageInfoList.Add(file.FullName);
                            }
                            label1.Text = $"{imageInfoList.Count}张图片";
                            label2.Text = "0";
                        }

                    }
                    break;
                    //读取一张图片
                case "radImage":
                    {
                        OpenFileDialog ofd = new OpenFileDialog();
                        ofd.Multiselect = false;
                        ofd.Title = "请选择图片文件";
                        ofd.Filter = "所有文件|*.*|bmp文件|*.bmp|jpg文件|*.jpg";
                        if (ofd.ShowDialog() == DialogResult.OK)
                        {
                            m_imagePath = ofd.FileName;
                            mIFTool.Operator.Open(ofd.FileName, CogImageFileModeConstants.Read);
                            mIFTool.Run();
                            //mIFTool.Run()之后才会得到mIFTool.OutputImage
                            m_Image = (CogImage8Grey)mIFTool.OutputImage;

                            label1.Text = null;
                            label2.Text = null;
                        }
                       
                    }
                    break;
            }
        }
```



## 加载读取的图片

```c#
     private void button1_Click(object sender, EventArgs e) //运行按钮
        {
            cogDisplay1.Fit(); //CogDisplay显示控件
            //相机取图
            if (radCamera.Checked)
            {
                int trigNum = 0;
                cogDisplay1.Image = mAcq.Acquire(out trigNum);
            }
            //文件夹加载
            else if (radFolder.Checked)
            {
                
                if (imageInfoList.Count > 0)
                    mIFTool.Operator.Open(imageInfoList[m_imageIndex], CogImageFileModeConstants.Read);
                else
                    MessageBox.Show("图像不存在");
                mIFTool.Run();
                cogDisplay1.Image = mIFTool.OutputImage;
                label2.Text = $"第{m_imageIndex + 1}张图片";
                m_imageIndex++;
                if (m_imageIndex > imageInfoList.Count - 1)
                {
                    m_imageIndex = 0;
                }
            }
            //文件加载
            else if (radImage.Checked)
            {
                cogDisplay1.Image = m_Image;
            }
        }
```



## 相机取图

 

```c#
      //初始化相机 并且注册取图完成事件
      private void InitCamera()
        {
            CogFrameGrabbers cfg = new CogFrameGrabbers();
            mAcq = cfg[0].CreateAcqFifo(cfg[0].AvailableVideoFormats[0],
                CogAcqFifoPixelFormatConstants.Format8Grey, 0, true);
            var trigger = mAcq.OwnedTriggerParams;
            trigger.TriggerModel = CogAcqTriggerModelConstants.Auto;
            if (trigger.TriggerEnabled == false)
            {
                trigger.TriggerEnabled = true;
            }
            mAcq.Complete += AcqFifo_Complete;
        }
```



```c#
   
     //取图完成事件
    private void AcqFifo_Complete(object sender, CogCompleteEventArgs e)
    {
        try
        {
            if (InvokeRequired)
            {
                object[] eventArgs = { sender, e };
                Invoke(new CogCompleteEventHandler(AcqFifo_Complete), eventArgs);
                return;
            }
            int numReadyVal, numPendingVal;
            bool busyVal;
            CogAcqInfo info = new CogAcqInfo();
            mAcq.GetFifoState(out numPendingVal, out numReadyVal, out busyVal);
            int numAcqs = 0;
            if (numReadyVal > 0)
            {
                m_Image = mAcq.CompleteAcquireEx(info);
                //在这里运行作业
                numAcqs++;
            }
            if (numAcqs > 4)
            {
                GC.Collect();
                numAcqs = 0;
            }
        }
        catch (CogException ex)
        {

            return;
        }
    }
```
