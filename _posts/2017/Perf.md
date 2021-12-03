---
layout: post
title: Perf Experiment
date: '2021-12-03 11:00'
excerpt: >-
  Experiment on Perf
comments: true
---
###### tags: `os`
# Perf 
是一個系統效能分析工具
Data source:http://wiki.csie.ncku.edu.tw/embedded/perf-tutorial
## 環境
ubuntu 18.04
## 安裝Perf
### 查看目前的 Kernel config 有沒有啟用 Perf
```
cat "/boot/config-`uname -r`" | grep "PERF_EVENT"
```

![](https://i.imgur.com/9iyClnI.png)

可以看到目前perf已被啟用

### 測試perf是否被安裝
```
 perf top 
```

![](https://i.imgur.com/ZHPjzFD.png)


發現尚未安裝跟著指示執行安裝(會有些套件要裝)即可

安裝後執行
```
 perf top 
```
會出現以下error畫面
![](https://i.imgur.com/VE9qkq7.png)

kernel.perf_event_paranoid 是用來決定你在沒有 root 權限下 (Normal User) 使用 perf 時，你可以取得哪些 event data。預設值是 1 ，你可以輸入
```
cat /proc/sys/kernel/perf_event_paranoid
```

![](https://i.imgur.com/VHpx8yy.png)
能看到我預設是4,代表disallow kernel profiling

### 取消 kernel pointer 的禁用
要檢測 cache miss event ，需要先取消 kernel pointer 的禁用
```
sudo sh -c " echo 0 > /proc/sys/kernel/kptr_restrict"
```

## 開始實驗perf
### 實驗程式
perf_top_example.c
```
#include <stdio.h>
#include <unistd.h>

double compute_pi_baseline(size_t N) {
    double pi = 0.0;
    double dt = 1.0 / N;
    for (size_t i = 0; i < N; i++) {
        double x = (double) i / N;
        pi += dt / (1.0 + x * x);
    }
    return pi * 4.0;
}
int main() {
    printf("pid: %d\n", getpid());
    sleep(10);
    compute_pi_baseline(50000000);
    return 0;
}


```
### 執行perf

使用GCC編譯perf_top_example.c並執行
```
g++ -c perf_top_example.c //Compile
g++ perf_top_example.o -o example  //Convert from object code to executable
./example  //Run
```

執行後可得到一個pid值
![](https://i.imgur.com/r3OwRqc.png)
可看到pid=3464
在跑得同時用perf去看效能
```
perf top -p $pid
```

但我出現以下的error

![](https://i.imgur.com/fZ95ZxT.png)

發現原因為我的kernel.perf_event_paranoid 被預設為4,所以權限不足
有兩種方式修改

1. 

```
sudo sh -c "echo -1 > /proc/sys/kernel/perf_event_paranoid"

```
或
```
sudo sysctl -w kernel.perf_event_paranoid=-1
```


2. 
```
sudo sh -c 'echo "kernel.perf_event_paranoid=-1" >> /etc/sysctl.conf'
sudo sysctl -p
```


解決後再次執行
```
perf top -p $pid
```

![](https://i.imgur.com/TbUPa45.png)

預設的 performance event 是 「cycles」，所以這條指令可以分析出消耗 CPU 週期最多的部份以上結果可看出在函式 compute_pi_baseline() 佔了近 96.12％

## PMU
PMU (performance monitor unit)。PMU 允許硬體針對某種事件設置 counter，此後處理器便開始統計該事件的發生次數，當發生的次數超過 counter 內設定的數值後，便產生中斷。比如 cache miss 達到某個值後，PMU 便能產生相應的中斷。一旦捕獲這些中斷，便可分析程式對這些硬體特性的使用率了。

## Tracepoint
Tracepoint 是散落在核心原始程式碼的一些 hook，一旦使能，在指定的程式碼被運行時，tracepoint 就會被觸發，這樣的特性可被各種 trace/debug 工具所使用，perf 就是這樣的案例。若你想知道在應用程式執行時期，核心記憶體管理模組的行為，即可透過潛伏在 slab 分配器中的 tracepoint。當核心運行到這些 tracepoint 時，便會通知 perf。

Perf 將 tracepoint 產生的事件記錄下來，生成報告，通過分析這些報告，效能分析調校的工程人員便可瞭解程式執行時期的核心種種細節，也能做出針對效能更準確的診斷。

## perf top
能夠「即時」的分析各個函式在某個 event 上的熱點，找出拖慢系統的兇手，就如同上面那個範例一樣。甚至，即使沒有特定的程序要觀察，你也可以直接下達 $ perf top 指令來觀察是什麼程序吃掉系統效能，導致系統異常變慢譬如我執行一個無窮迴圈：
```
int main() {
    long int i = 0;
    while(1) {
        i++;
        add(i);
        div(i);
    }
    return 0;
}
```
會出現以下畫面

![](https://i.imgur.com/sIzOxDH.png)

左邊第一行是該符號引發的 event 在整個「監視域」中佔的比例，我們稱作該符號的熱度，監視域指的是 perf 監控的所有符號，特別的是若符號旁顯示[.]表示其位於 User mode，[k]則為 kernel mode。

若你想要觀察其他 event ( 預設 cycles ) 和指定取樣頻率 ( 預設每秒4000次 ) :

```
perf top -e cache-misses -c 5000
```

## perf stat
使用 perf stat 往往是你已經有個要優化的目標，對這個目標進行特定或一系列的 event 檢查，進而了解該程序的效能概況。（event 沒有指定的話，預設會有十種常用 event。） 我們來對以下程式使用 perf stat 工具 分析 cache miss 情形

測試code
```
static char array[10000][10000];
int main (void){
  int i, j;
  for (i = 0; i < 10000; i++)
    for (j = 0; j < 10000; j++)
       array[j][i]++;
  return 0;
}
```
編譯執行後輸入：
```
perf stat --repeat 5 -e cache-misses,cache-references,instructions,cycles ./perf_stat_cache_miss
```
會出現如下畫面

![](https://i.imgur.com/cmXhUme.png)

--repeat <n>或是-r <n> 可以重複執行 n 次該程序，並顯示每個 event 的變化區間。 cache-misses,cache-references和 instructions,cycles類似這種成對的 event，若同時出現 perf 會很貼心幫你計算比例。

根據這次 perf stat 結果可以明顯發現程序有很高的 cache miss，連帶影響 IPC 只有0.65。

如果我們善用一下存取的局部性，將 i，j對調改成array[i][j]++

可發現cache miss少很多

![](https://i.imgur.com/vkh5CLo.png)

cache-references 從 408,030,984下降到 2,525,828，差了一百五十幾倍，執行時間也縮短為原來的三分之一！
不是很理解爲何將 col major 改爲 row major 造成cache refs & cache miss 都下降，但 cache refs 下降得更多，始得 cache miss rate看起來是升高的，而非原本心中預期的 cache refs 不變且 cache miss count 下降。

Cache-references: cache 命中的次數

Cache-misses: cache 失效的次數
IPC：是 Instructions/Cycles 的比值，該值越大越好，說明程式充分利用了處理器的特性


## perf record & perf report
有別於 stat，record 可以針對函式級別進行 event 統計，方便我們對程序「熱點」作更精細的分析和優化。 我們來對以下程式，使用 perf record 進行 branch 情況分析
```
#define N 5000000
static int array[N] = { 0 };
void normal_loop(int a) {
    int i;
    for (i = 0; i < N; i++)
        array[i] = array[i]+a;
}
void unroll_loop(int a) {
    int i;
    for (i = 0; i < N; i+=5){
        array[i] = array[i]+1;
        array[i+1] = array[i+1]+a;
        array[i+2] = array[i+2]+a;
        array[i+3] = array[i+3]+a;
        array[i+4] = array[i+4]+a;
    }
}
int main() {
    normal_loop(1);
    unroll_loop(1);
    return 0;
}
```

編譯執行後
```
$ perf record -e branch-misses:u,branch-instructions:u ./perf_record_example
 perf report
```
:u是讓 perf 只統計發生在 user space 的 event。最後可以觀察到迴圈展開前後 branch-instructions 的差距。
另外，使用 record 有可能會碰到的問題是取樣頻率太低，有些函式的訊息沒有沒顯示出來（沒取樣到），這時可以使用 -F <frequcncy>來調高取樣頻率，可以輸入以下查看最大值，要更改也沒問題，但能調到多大可能還要查一下。
```
cat /proc/sys/kernel/perf_event_max_sample_rate
```
結果如下：

![](https://i.imgur.com/lBaL0Gr.png)

