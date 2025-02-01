# FSxForLustre_GPUDirectStorage

ここでは、**AWS FSx For Lustre**で用いられるGPUDirect Storageについて、ハードウェア視点から詳細に解説します。より高い性能を追求するためには、計算機科学の視点からAWSの実装を抑えておくことはとても重要です。

今回は、DMAやRDMAの基本的なテクノロジーを抑えた上で、GPUDirect Storage、GPUDirect RDMAについて解説して行きたいと思います。

# 1. What is RDMA ?
RDMA はリモート ダイレクト メモリ アクセスです。 
しかし、RDMA 自体が非常に難解であるため、計算機科学の視点からきちんと理解できるように段階的に書いてみました。
RDMA について話す前に、DMA の仕組みとその問題点を勉強する必要がありますので、一旦DMAの原理から解説します。

---
RDMA is Remote Direct Memory Access. 
But not all of engineers can understand it well because RDMA itself is very difficult to understand. I wrote it down step by step to understand it. 
Before talking about RDMA, we should study how DMA works and its problems. 
See also the URL below:
(https://github.com/developer-onizuka/what_is_DMA/blob/main/README.md)

# 2. Problems of traditional DMA
**DMA はホスト メモリとデバイスのメモリ間のコピーです。** 以下に示すように、デバイスの BAR 空間からホスト マシンの物理メモリ アドレスへのコピーです。
ホストとデバイス間の DMA の後、デバイスの DMA エンジンは CPU に割り込み、CPU がこの DMA 処理されたデータ (まだカーネル空間にある) を CPU 負荷
によってユーザー空間にコピーし始めることができます。これは DMA ではありません。次に、カーネル内のスペースが次の DMA 操作のために解放されます。
これは DMA のフロー制御として知られています。
カーネル空間からユーザー空間へのコピーは、CPU 負荷のある一部のプロセスによって実行されたことに注意してください。これは、コピーが「仮想アドレス」
を通じて実行されたことを意味します。 もちろん、コピーには「物理アドレス」が必要です。

ここで、「アドレスを仮想アドレスから物理アドレスに変換するのは誰ですか？」という問いをもし投げかけられたとすると、その答えはカーネルになります。

つまり、DMA 後のコピーでは、カーネルだけが仮想アドレスと物理アドレス間のマッピングを知っているため、一部のプロセス (TCP/IP および NIC ドライバー) は
カーネル API を呼び出して仮想アドレスから物理アドレスに変換します。この変換後、ハードウェアによって物理コピーが実行されます。

非常に高スループットの NIC を使用している場合、どれくらいの割り込みが発生すると思いますか?現在、Mellanox や Intel からは非常に多くの高性能 NIC 
が提供されています。ここで問題になるのは、追加の CPU ワークロードを引き起こすこれらの大量の割り込みそのものです。 CPUはNICなどのI/O専用ではなく、
ユーザープログラムの処理も含めた用途に使用されます。さらには、CPUが占有されるだけでなく、**貴重なTLBなどのリソース**も使われてしまいます。

---
**DMA is a copy between host memory and device's memory.** As you can see below, the copy from device's BAR space to a physical memory address in host machine. After DMA between host and device, Device's DMA Engine interrupts to CPU so that CPU can start copying this DMAed data (it's still in kernel space) to the user space by CPU load, which is not DMA. Next, the space in kernel is released for the next DMA operation, which we know it as flow control of DMA.

Please note the copy from the kernel space to the user space was done by some processes with CPU load. It means the copy was performed thru "Virtual Address". 
Of cource, copy needs "Physical Address".** Who translates the address from Virtual address to Physical address ??? Yes, Kernel does**.

For copying after DMA, some processes (TCP/IP and NIC driver) call kernel API for translation from Virtual address to Physical address because Only the kernel knows the mapping between Virtual Address and Physical Address. After this translation, physical copies will be perform by hardware.

Guess how many interrupts happens if you are using so high throughput NIC? Today, there are so many High performance NICs from Mellanox and Intel... 
The problem is these heavy interrupts itself which introduce additional CPU workloads. CPU is not dedicated to I/O such as NIC but for the purpuses including user program's processing. In addtion, its resources such as **TLB are very very precious**. 
   
```
          Physical Memory
          +----------+
          |          |
          |          |
          +----------+
          |XXXXXXXXXX|
   +----- |XXXXXXXXXX|
   |      +----------+ 0xf0000000 (NIC BAR#1)
   |      |          |                                          Kernel Space (Virtual Address)
 Copy     |          |                                          +----------+
 (DMA)    |          |                                          |          |
   |      +----------+                                          |          | 
   +----> |XXXXXXXXXX|                                          |XXXXXXXXXX|
          |XXXXXXXXXX| <================Mapping===============> |XXXXXXXXXX| -----+
          +----------+ Host Memory for DMA operation            +----------+      |
          |          | (Physical Address)                                        Copy
          |          |                                          +----------+     (CPU)
          +----------+                                          |          |      |
          |XXXXXXXXXX|                                          |XXXXXXXXXX|      |
          |XXXXXXXXXX| <================Mapping===============> |XXXXXXXXXX| <----+
          +----------+ User Space (Physical Address)            +----------+
          |          |                                          User Space (Virtual Address)
          |          |
          +----------+ 0x00000000
```

# 3. Zero Copy (Kernel Bypass)
DMAの問題を克服することを考えてみましょう。アイデアの 1 つは、コンテキストの切り替えが起こらないようにカーネルをバイパスすることです。ユーザーは物理アドレスを参照できないため、ユーザーアプリケーションは仮想アドレスを介してプログラムされます。ユーザプログラム内で「verbs API」を使用することで、ユーザプログラムはInfiniBand HCAやMellanox NICなどのRDMAデバイスにユーザ空間の物理アドレスを知らせることができます。ただし、RDMA は DMA の 1 つであり、ハードウェア ロジックによってアドレス自体を変換できる点が異なります。

---
How can we come over the problem of traditional DMA? One of ideas is bypassing kernel so that any context switch never happens. The user application is programmed through virtual address because users can not refer the physical address. The user program can let the RDMA device such as InfiniBand HCA or Mellanox NIC know user space's physical address by using "verbs API" in the user program. But RDMA is one of DMA and the difference is what it can translate the address itself by its hardware logic.

Step 1. 
---
ユーザープログラムは、malloc()を通じて仮想アドレスとして独自の空間を作成します。

---
User program creates its own space as virtual address thru malloc().
```
          Physical Memory
          +----------+            
          |          |             
          |          | 
          +----------+ 
          |          |
          |          | 
          +----------+ 0xf0000000 (NIC BAR) 
          |          |                       
          |          |
          |          | 
          |          | 
          |          | 
          |          |
          |          |
          |          |                                   
          |          |                                          +----------+
          |          |                                          |          |
          |          |                                          |          |
          |          |                                          |          |
          |          |                                          +----------+ 0xc0004000
          |          |                                          User Space (Virtual Address)
          |          |
          +----------+ 0x00000000
```
Step 2. 
---
ユーザープログラムで扱うメモリが、カーネルによってスワップアウトされないように、事前にVerbs API を介して登録されたスペースを要求します。これを**PIN**と呼びます。 PIN に加えて、ユーザープログラムはいくつかの制御リソースを作成します。その一つとして PTE があり、ユーザープログラム空間の物理アドレスと仮想アドレス間の変換テーブルとして機能するものです。この操作を**メモリレジストレーション**と呼び、 PTE はメモリレジストレーションのバックグラウンドで作成されるものとなります。ただし、メモリーレジストレーションは非常に負荷が大きいので、このAPIを呼ぶタイミングを意識したチューニングが必要となる場合が多いです。というわけで、カーネルに頼らず、通信性能を向上させるわけですから、ユーザープログラムが直接ハードウェアを操作することを行う必要があります。

---
An user program asks the space registered thru verbs API so that the kernel could not swap it out to disk. We call it "**PIN**". In addition to the PIN, the user Program creates several control resources. One of resources is the PTE which is for translation table between physical address and virtual address of user program space. We call this operation "**Memory Registration**". The PTE is gonna be created in the background of the operation of Memory Registration. But please understand the Memory Registration is very heavy operation, so users should tune performances. These are almost everything which the user program should do for perspectives of kernel bypass before a packet arriving.
```
          Physical Memory
          +----------+            
          |          |             
          |          |                                          RDMA Control resources
          +----------+                                          +----------+
          |          |                                          |          |
          |          |                                          |          |
          +----------+ 0xf0000000 (NIC BAR)                     | PTE#1    |
          |          |                                          +----------+                       
          |          |
          |          | 
          |          | 
          |          | 
          |          |
          |          |
          |          |                                          PINNED
          |          |                                          +----------+
          +----------+                                          |          |
          |          |                                          |          |
          |          | <================Mapping===============> |          |
          +----------+ User Space (Physical Address)            +----------+ 0xc0004000
          |          |                                          User Space (Virtual Address)
          |          |
          +----------+ 0x00000000
```

Step 3. 
---
パケットはNICに到着すると、データは NIC 上の BAR スペースに置かれます。しかし、この時点では、NICのDMAエンジンは、どこに DMA を実行すべきかを実は知りません。

---
Data comes into the NIC logic and put it on BAR space on NIC. But **the DMA engine doesn't know where it should do DMA to!**
```
          Physical Memory
          +----------+            +---------- Data From Outside
          |          |            |
          |          |            |                             RDMA Control resources
          +----------+            |                             +----------+
          |          |            |                             |          |
   +----- |XXXXXXXXXX| <----------+                             |          |
   |      +----------+ 0xf0000000 (NIC BAR#1)                   | PTE#1    |
   |      |          |                                          +----------+                       
   V      |          |
  ???     |          | 
          |          | 
          |          | 
          |          |
          |          |
          |          |                                          PINNED
          |          |                                          +----------+
          +----------+                                          |          |
          |          |                                          |          |
          |          | <================Mapping===============> |          |
          +----------+ User Space (Physical Address)            +----------+ 0xc0004000
          |          |                                          User Space (Virtual Address)
          |          |
          +----------+ 0x00000000
```
Step 4. 
---
DMAエンジンは、どこにDMAするかを知る必要があります。このため、ホストメモリの特定のスペースから PTE#1 をフェッチします。このPTE#1には、ユーザー空間に対するコピー先がプログラムされています。これにより、DMAエンジンはNICとユーザー空間の間でコピーできるようになります。しかし、**ユーザープロセスはカーネル介入なしでコピーの完了をどのように理解するのか？**　という疑問は残ります。 これは、**割り込み**と呼ばれるカーネル機能を使用していないためなのですが、RDMAを扱う上での対策しないといけない課題です。それはStep5で説明します。

---
DMA Engine starts fetching the PTE#1 from certain space from host memory so that it can know where it does DMA. Then, DMA Engine can copy between NIC and user space without additional copies. But **how does the user process understand the completion of copy without kernel interventions???** This is a new problem while we don't use kernel features which we call "interrupts".
```
          Physical Memory
          +----------+            +---------- Data From Outside
          |          |            |
          |          |            |                             RDMA Control resources
          +----------+            |                             +----------+
          |          |            |                             |          |
   +----- |XXXXXXXXXX| <----------+                             |          |
   |      +----------+ 0xf0000000 (NIC BAR#1)                   | PTE#1    |
   |      |          |                                          +--+-------+                       
   V      |          |                                             |
  ??? <------------------------------------------------------------+ 
   |      |          | 
   |      |          | 
   |      |          |
   |      |          |
   |      |          |                                          PINNED
   |      |          |                                          +----------+
   |      +----------+                                          |          |
   |      |          |                                          |          |
   +----> |XXXXXXXXXX| <================Mapping===============> |XXXXXXXXXX|
          +----------+ User Space (Physical Address)            +----------+ 0xc0004000
          |          |                                          User Space (Virtual Address)
          |          |
          +----------+ 0x00000000
```
Step 5. 
---
**DMA エンジンは、カーネルに割り込みするのではなく、Completion Queue (CQE#1) というものを作成します。** そして、ユーザープログラムは、CQE が作成されるまでポーリングします。これで RDMA ステップは終了です。完了後、BAR スペースのデータは (ポインターをインクリメントすることによって) 削除でき、NIC の受信機は次のパケットを受信する準備が整います。 PTE や CQE などのホスト メモリ内の各リソースは、後続のプロセスがこれらのリソースを使用しない場合、破棄され固定解除されます。また先述の通り、メモリレジストレーションは非常にコストの高い操作なので、再利用することをお勧めします。

---
**DMA Engine creates the Completion Queue(CQE#1) instead of interruptting to the kernel.** The user program polls until a CQE is created. This is the end of RDMA step. After completion, the data of BAR space can be removed (by incrementing the pointer) and NIC's receiver is ready for receiving the next packet. Each resource in host memory such as PTE or CQE will be destroyed and unpinned if subsequent process does not use these resources. The space already registered may be used again so that we can prevent from heavy process of memory registration.
```
          Physical Memory
          +----------+            
          |          |             
          |          |                                          RDMA Control resources
          +----------+                                          +----------+
          |          | <--- next pointer to receive             |          |
          |XXXXXXXXXX|                                          | CQE#1    | <---Polling by
          +----------+ 0xf0000000 (NIC BAR#1)                   | PTE#1    |     user program
          |          |                                          +----------+                       
          |          |                                              
          |          |
          |          | 
          |          | 
          |          |
          |          |                                          
          |          |                                          Unpinned (Kernel can swap out)
          |          |                                          +----------+
          +----------+                                          |          |
          |          |                                          |          |
          |XXXXXXXXXX| <================Mapping===============> |XXXXXXXXXX|
          +----------+ User Space (Physical Address)            +----------+ 0xc0004000
          |          |                                          User Space (Virtual Address)
          |          |
          +----------+ 0x00000000
```

# 4. GPUDirect Storage
さて、前置きが長くなりましたが、GDS とは、ストレージ (NVMe または NIC) 近くの DMA エンジンがデータを GPU メモリに直接プッシュ (またはプル) するための特別なDMAテクノロジーです。
GDSのDMAオペレーションでは、ホストメモリの代わりに、PCI デバイス上の BAR スペースをターゲットとして使用することになります。GDS では、GPU がその BAR を提供し、NVMe の DMA エンジンは、カーネルが GPU の BAR を介してマッピングした物理アドレスにアクセスします。 GDSに関するDMA は、NVMe の DMA エンジンによって、GPU BAR スペースと NVMe BAR スペース (わずか 16KB、正規の NVMe デバイスに必要) 間のコピーを通じて実行されます。 (GPU の DMA エンジンによるものではありません) 

---
We can use BAR space on a PCI device as a target instead of host memory for DMA operation. According the P2P DMA, One device needs to present a memory BAR, and the other one accesses it. 
In GDS, GPU provides its BAR and the NVMe's DMA Engine accesses the physical address which the kernel mapped into thru the GPU's BAR. The P2P DMA is done thru the copy between GPU BAR space and NVMe Bar space (only 16KB, any legitimate NVMe device must have) by NVMe's DMA Engine. (Not by GPU's DMA Engine) 

```
         Physical Memory
          +----------+                                          The file in NVMe Storage
          |          |                                          (mount -t ext4 -o data=ordered /dev/nvme0n1 /mnt)
          +----------+ 0xfd603fff                Fetching       +----------+
   +----- |XXXXXXXXXX| 16KB  <--------------------------------- |XXXXXXXXXX| /mnt/test.txt
   |      +----------+ 0xfd600000 (NVMe's BAR)                  |          |
  Copy*   |          |                                          +----------+
(P2P DMA) |          |                                           
   |      +----------+ 0xdfffffff                               GPU Memory associated with CudaMalloc()
   |      |          |     Copy from BAR space to GPU Memory    +----------+
   +----> |XXXXXXXXXX| ---------------------------------------> |XXXXXXXXXX| ---> Cunsumed by GPU core
          +----------+ 0xd0000000 (GPU's BAR)                   |          |
          |          |                                          +----------+
          |          |
          |          |                                          Kernel Space (Virtual Address)
          |          |                                          +----------+
          |          |                                          |          |
          |          |                                          |          |
          |          |                                          +----------+
          |          |                                                      
          |          |                                          +----------+
          |          |                                          |          |
          |          |                                          |          |
          |          |                                          |          |
          |          |                                          +----------+
          |          |                                          User Space (Virtual Address)
          |          |
          +----------+ 0x00000000

   * : I believe the NVMe's DMA Engine do this copy. Not by GPU's DMA Engine.

```

# 5. GPUDirect RDMA
前述のGPUDirect Storageは、GPUメモリとNVMeとの間でのDMAでした。GPUDirect RDMAは、別ノードにおけるGPUメモリとGPUメモリの間でのDMAとして扱われるものです。イメージとしてはこんな感じです。

![gpudirect-rdma.png](https://d29g4g2dyqv443.cloudfront.net/sites/default/files/akamai/GPUDirect/gpudirect-rdma.png)
