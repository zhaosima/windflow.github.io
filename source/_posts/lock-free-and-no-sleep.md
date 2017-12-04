---
title: free lock and no sleep
date: 2017-12-01 14:22:33
tags:
---

## 失踪的毕加索和斯内普

### 多线程顺序读取不加锁


        int myindex;
        readonly List<int> mydata = new List<int>() { 5, 7, 9 };

        int getdata()
        {
            var oldindex = myindex;

            while(Interlocked.CompareExchange(ref myindex, (oldindex+1)%mydata.Count, oldindex) != oldindex)
            {
                oldindex = myindex;
            }

            return mydata[myindex];
        }


### 等待读取不睡眠

        ConcurrentQueue<object> logQueue = new ConcurrentQueue<object>();

        void ConsumeLogContinuously()
        {
            while (true)
            {
                if (this.logQueue.Count > 0)
                {
                    this.ConsumeLog();
                }
                else
                {
                    SpinWait.SpinUntil(() => this.logQueue.Count > 0);
                }
            }
        }

