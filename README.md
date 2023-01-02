# 2022計組期末專題 - CPU pipeline

## 環境
* Ubuntu 22.04.1 LTS
* C/C++ standard: C11
* C/C++ compiler: g++
> Windows系統可以編譯，但因為cmd指令不同，clean、clean-all無法使用


## 如何使用

> 使用makefile的方式避免編譯環境上的問題，你只需要確保你有開發環境中的編譯器及make的工具 \
> 如果沒有make的工具，在Linux系統的cmd中輸入 
> ```cmd 
> sudo apt install make
> ```


首先，開啟你的bash界面，並輸入以下指令，將此專案下載到本地端

```cmd
cd Desktop
git clone https://github.com/Sunny1928/CO_mid_pipline.git
```
進入 CO_mid_pipline 中，並進行編譯
```cmd
cd CO_mid_pipline
make
```
開始執行
```cmd
#使用預設
./main 

# 放入檔案路徑
./main <file_path>
```
清除所有.o檔，不包含執行檔
```cmd
make clean
```
清除所有.o檔，包含執行檔
```cmd
make clean-all
```

## 專案架構
```
CO-mid_pipline
├── main.cpp
├── Makefile
├── Flow_Chart.png
├── README.md
├── lib
│   ├── include
│   │   ├── CPU_pipeline.h
│   │   ├── EX.h
│   │   ├── ID.h
│   │   ├── IF.h
│   │   ├── MEM.h
│   │   └── WB.h
│   └── src
│       ├── CPU_pipeline.cpp
│       ├── EX.cpp
│       ├── ID.cpp
│       ├── IF.cpp
│       ├── MEM.cpp
│       └── WB.cpp
└── test_data
    ├── q1.txt
    ├── q2.txt
    ├── q3.txt
    ├── q4.txt
    ├── q5.txt
    ├── q6.txt
    ├── q7.txt
    └── q8.txt

```
## 程式架構
<img src="https://github.com/Sunny1928/CO_mid_pipline/blob/main/Flow_Chart.png" width = "70%">

&ensp;&ensp;&ensp;&ensp;程式主要分成三個部份，main、CPU_pipeline和五個階段IF、ID、EX、MEM、WB。<br>
<br>
&ensp;&ensp;&ensp;&ensp;我們將這次的專題實做成一個Class CPU_pipeline，使用者只需要從main將MIPS指令傳入class中，就可以得到對應的執行結果。<br>
<br>
&ensp;&ensp;&ensp;&ensp;實做的主體在於cycle階段執行的內容，我們將PC的更動和是否需要執行Stall的判斷，此安排是用來解決無法同步執行的問題，接著正式進入pipeline中，循環的順序為intoWB()->intoMEM()->inoEX()->intoID()->intIF()，這樣的安排是為了確保每次該階段執行時所拿的值，都是還沒被更新過的值，如果相反過來，就會出現像WB要拿MEM/WB register的值出來，但此暫存器的值已經被先執行的MEM更新，導致WB獲取的會是下一個cycle要用的值。<br>
<br>
&ensp;&ensp;&ensp;&ensp;在循環的過程中，我們將本來要在ID中判斷的beq改放到ID和EX之間，這樣設計的原因與前面新舊值影響的問題相同，且因為我們是將PC的更動放在cycle中，因此beq的判斷也放在同一層會比較好處理，可以透過改變index移動目標指令。<br>
<br>
&ensp;&ensp;&ensp;&ensp;不斷循環直到傳入的所有指令都執行完cycle function就會結束，並輸出結果。<br>

## 程式說明

#### 撰寫基礎
* 在ID所使用的值為IF/ID register中的值，在EX所用的值為ID/EX register中的值,以此類推。
* 為了方便處理每個regiter都一定會傳入rs,rt,rd和offset，沒有用到的就會填0。
* stall或是目前stage沒有指令，均會填入null做區別。

#### 功能說明

- ### main
    - 程式進入點。
    - 讀取MIPS指令，並做字串處理將每行指令分開。
- ### CPU_pipeline
    - 用index模擬PC的變化。
    - 將記憶體、暫存器與cycle數初始化。
    - 讀取的指令進行循環，並計算最後一個指令結束時所花費的cycle數。
    - 在一輪cycle結束後，判斷EX hazard和MEM hazard情況，如果有hazard的話，會將EX的傳入值改為null，代表不將ID的值往下傳，ID只執行更新WB回傳的值，IF不變，模擬stall。
    - 如果沒有hazard的話，正常運行。若在執行ID前opcode為beq，就會檢查beq條件是否成立。若成立就會改變下個指令的位置，並將ID傳入null，代表原本的IF沒有往下傳，而在寫入新位置到IF。
    
- ### IF
    - 將讀入的指令字串切割，例如：讀入add $1,$2,$3,處理後為[add,$1,$2,$3,]。
- ### ID 
    - Decode將讀入指令，根據對應的操作轉換成signal，例如：讀到指令lw，轉換成0101011、讀到指令add或是sub，轉換成1000010，而指令rs,rt,rd或是offset，則直接用字串處理取得使用，過程不會轉換成machine code。
    - 利用rs, rt取出要使用的暫存器存到reg1, reg2。
- ### EX
    - 判斷opcode是add,sub,lw,sw，決定要執行哪種操作，存到ALUresult。
- ### WB
    - 如果MemtoReg為1，則從記憶體取出的值，且RegWrite為1，更新到暫存器rt中。
    - 如果MemtoReg為0，則把從ALU中計算出的值，且RegWrite為1，更新到暫存器rd中。


## 程式重點

> 更詳細的內容請詳閱程式碼註解！ 

* ### Data Hazard判別
![image](https://user-images.githubusercontent.com/88101776/210197860-c670a2f4-91b8-43f5-9dc1-63d9aadee6e7.png)
* ### Beq 判別，改變PC
![image](https://user-images.githubusercontent.com/88101776/210197894-642fc4f8-d4f9-458a-9c54-51113ec8c48d.png)



## 遇到問題
1. 指令讀取處理
   * prob: 轉成machine code後，指令在ID階段讀取不方便。
   * sol: 放棄轉成machine code，到ID階段再使用字串切割分割出十進制的值。
2. Data Hazard處理
   * prob 1: 沒有單純做stall的電路架構
   * sol: 參考forwarding的電路架構，使用一樣的EX hazard、MEM hazard的判別方式，但不做提前把值做更新的操作。
   * prob 2:軟體無法模擬硬體的同步執行，導致hazard的判斷無法放在ID階段。
   * sol: 將hazard的判斷留到一個cycle執行完再判斷。
3. Beq的判斷
   * prob: Beq原本是在ID中做判斷的，但會因為執行先後順序的影響，破壞到指令運行的步驟
   * sol: 因此，我們把它移到循環的地方做判斷，因為那邊可以直接處理指令運行的順序。
4. lw & sw Offset
   * 在寫的時候，指令lw, sw的offset我們是直接讀取存入，但取得register或memory值之前，需要先除以四，位置才會對，因為單位是word。網路上查了很多資料，有些沒有除，有些有除，所以當時我們花了許多時間討論。


## 分工

在討論完整體架構後，我們將工作分為前半部，架構撰寫包含pipelined的五個階段，後半部為beq和data hazard的判斷及彙整程式碼。會這樣分配的原因是從頭建立架構比較麻煩，所以將較麻煩的stall處理拆出來，順便進行程式碼的檢查並彙整。
其餘的報告、Readme、makefile均為兩人合力討論寫出來的內容。


|Name|工作內容|
| :-----|:-----|
|莊郁誼 |程式架構設計(前半部）、撰寫、寫報告、Readme、makefile|
|廖怡誠 |程式架構設計(後半部）、撰寫、寫報告、Readme、makefile| 

## Contributors
|Name|Github Link|
| :-----|:-----|
|Yu-Yi Chuang | https://github.com/Sunny1928|
|Yi-Cheng Liao |https://github.com/yeeecheng| 
