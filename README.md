# SmartBeta_and_ETF

> **使用須知**  
> 由於github有檔案大小限制，故無法將股票歷史資料傳上來。  
> 因此在`clone`此專案後，先至下方連結下載整個Data資料夾並放入專案中，便可順利運行程式碼。  
> [Data資料夾連結](https://drive.google.com/drive/folders/1vPrZ9P7Bpk1GxwLc0B62CNO1v8fQufnK?usp=sharing)

## 步驟

* ### 特選高股息低波動臺灣中型30指數-一、篩選成分股

    **1. 資料前處理**

    **2. 前置篩選**

    **3. 檢測 SmartBeta 因子**

    **4. 計算 SmartBeta 因子**

---

* ### 特選高股息低波動臺灣中型30指數-二、事件整理

    **5. 不定期事件調整與處理**

---

* ### 特選高股息低波動臺灣中型30指數-三、計算成分股權重、指數

    **6. 定期調整**

    **7. 不定期調整**

    **8. 計算指數**

    **9. 績效測試**

---

* ### 特選高股息低波動臺灣中型30指數-四、績效比較

    **10. SmartBeta指數績效**
    
    **11. 完全模擬ETF**

<br>

---
## 資料前處理

* **讀檔**
    * 讀取準備好的csv檔案
    * TEJ 未調整日價格 上市 普通股
    ```python=
    df_D=pd.read_csv('.\Data\Daily_Data.txt',sep='\t',engine='python')
    ```
* **計算所需欄位**
    * 市值： 當日總市值 `當日收盤價 * 流通在外股數`
    * 20V： 20日均成交量 `過去20日成交量平均`
    * Return_252： 過去一年股價變動 `當日收盤價 / 252日前收盤價`
    * 每日報酬率： `（當日 - 前日） / 前日`
    * 20日年化波動度： `20日標準差 * 252^0.5`

* **訂定審核日及生效日**
    * Review_Date： 審核日 6月及 11 月的第7個交易日
    * Active_Date： 生效日 6月及 11 月的第13個交易日

<br>

## 前置篩選

* **市值篩選**
    * **目標**： 篩選出前 51 到 150 大市值的
    
    * 為原 Dataframe 添加一欄位為 `['市值篩選']`，並依據該證券之「當日市值」賦與值
    * 若該證券之市值為當日之 「51 ~ 150」，則給予值 "1" ，若非，則給予值 "0"
        * 該欄位值為 "1" 表示符合條件，與此同時，透過賦值 "1"，可在接下來使用`sum()`來確認是否有正確取到 100 家公司。

* **流通性篩選**
    * **目標**： 篩選出成交量大於1000萬的股票

    * 為原 Dataframe 添加一欄位為 `['市值篩選']`，並依據該證券之「20日平均成交值」賦與值
    * 若該證券之20日均成交值大於 1000 萬，則給予值 "1" ，若非，則給予值 "0"
        * 同上，該欄位值為 "1" 表示符合條件，並可透過`sum()`來確認共有多少支標的符合條件

* **整合兩條件**
    * **目標**： 選出同時通過「市值篩選」和「流通性篩選」的標的
    * 檢測「市值篩選」和「流通性篩選」皆為 "1" 的標的
    * `reset_index()`：重新給予 index 值

        ```python=
        df_May_Nov_lastday = df_May_Nov_lastday[(df_May_Nov_lastday['市值篩選']==1) & (df_May_Nov_lastday['成交值篩選']==1) ].reset_index(drop=True)
        ```
<br>

## 檢測 SmartBeta 因子

* **股利發放品質**
    * 計算最近三年現金股利與盈餘之比值
    * 篩選方式為三者之平均值介於 0 與 1 之間之標的

* **股利率篩選**
    * 計算最近一年現金股利與資料截止日收盤價之比值
    * 篩選方式為遞減排序並取排名前百分之七十且股利率低於 30%之個股。

* **波動度篩選**
    * 篩選方式為遞增排序並取排名前百分之七十的個股。

* **整合三條件**
    * **目標**： 選出同時通過「股利發放品質」、「股利率篩選」和「波動度篩選」的標的
    * 檢測「股利發放品質」、「股利率篩選」和「波動度篩選」皆為 "1" 的標的，若該標的符合此條件則將該標的納入我們最終選股池
    * （此處做法為令三欄位相加，總和為 3 則意謂符合三項條件）

        ```python=
        df_May_Nov_lastday_mergeA['篩選結果'] = (df_May_Nov_lastday_mergeA[['股利發放品質篩選','股利率篩選','波動度篩選']].sum(axis=1) ==3)*1
        ```
<br>

## 計算 SmartBeta 因子

* **SmartBeta**：後面計算權重時會以 SmartBeta 做依據

* **目標**： 高股息、低波動

* 股利率及年化波動度，以 2 : 3 為比重計算綜合分數
    * 因子

        * **股利率**：計算最近一年現金股利與資料截止日收盤價之比值

        * **波動度**：計算20日之年化波動度
    
    * 針對以上兩數值做標準化，方式如下：
        ```python
        (Xi - min) / (max - min) 
        ```
        * Xi  為當日第 i 檔成分股的數值（股利率 / 波動度）
        * min 為當日最小值（股利率 / 波動度）
        * max 為當日最大值（股利率 / 波動度）

    * 處理後的所有值都會介於 [0~1]，把它們分別命名為「股利率標準化」和「波動度標準化」

    * 最後再用「股利率標準化」和「波動度標準化」，搭上 2 : 3 的權重，得出各成分股的 SmartBeta 值。

        ```python=
        df_select['SmartBeta'] = df_select['股利率標準化']*2 + df_select['波動度標準化']*3
        ```
<br> 

## 不定期事件調整與處理

* **目標**：針對每個不同的事件對市值做對應的處理

* **運用到前一日收盤價事件**  

    (一) 新增或刪除採樣股票生效日。
    * 調整市值＝前一日收盤價 × 發行股數

    (三) 員工酬勞增資股或新股權利證書上市日。
    * 調整市值＝除權日前一日收盤×員工分紅(仟股)

    (六) 公司依法註銷股份辦理減資經本公司公告後之除權交易日或次月第三個交易日，並以較先者為準。
    * 調整市值 = 除權日前一日收盤×庫藏股註銷(仟股)

    (八) 公司合併後增資股或新股權利證書上市日。
    * 調整市值 = 除權日前一日收盤×合併(仟股)

    (九) 轉換公司債轉換的債券換股權證換發為普通股的上市日。
    * 調整市值 = 除權日前一日收盤×證券轉換(仟股)

    (一二) 為海外存託憑證而發行的新股上市日。
    * 調整市值 = 除權日前一日收盤× GDR比率% ×現金增資(仟股)

    (一三) 可轉換特別股轉換為普通股的上市日。
    * 調整市值 =除權日前一日收盤×證券轉換(仟股)

* **運用到其他日期事件**  

    (二) 現金增資認購普通股的除權交易日。
    * 調整市值＝現金增資認購價 × 現金增資股數

<br>

## 定期調整

* **目標**：每個半年重新檢驗一次選定標的，將不符合條件剔除並選入新標的

* **成分股定期審核**

    * 每年2次進行成分股審核，分別以6月及12月的第7個交易日為審核日，審核資料繳交截止日分別為5月及11月最後1個交易日

    * 每次定期審核後，維持固定的成分股數目為30檔

* **成分股調整之緩衝區**

    * 排名在第20名以內之股票即納入成為指數成分股，現有成分股若排名在第35名之後則從指數成分股中刪除

* **成分股定期審核結果**

    * 於6月及12月的第7個交易日發布技術通知後，間隔5個交易日後生效

<br>

## 不定期調整

* 將 `特選高股息低波動臺灣中型30指數-二、事件整理.ipynb` 中處理好的調整市值套用至此。

<br>

## 計算指數

* **權重**：透過 SmartBeta 數值來計算各成分股權重

    * 該成分股 SmartBeta 值 / 總和 SmartBeta 值
    
        ```python=
        df_final['Weight'] = df_final.groupby('Data_End_Date')['SmartBeta'].apply(lambda x : x/x.sum())
        ```

* **指數計算**：透過各成分股的權重及市值來計算指數

    * **公式**：

        <img src="https://i.imgur.com/3dJaR8S.png" height=500px></img>

    * **當前指數表現**：

        <img src="https://i.imgur.com/xHctT2a.png" height=300px></img>

    
<br>

## 績效測試

* 目前我找了幾檔指數來做比對：

    * **加權指數、臺灣50**
    * <img src="https://i.imgur.com/oWLZm9b.png" height=300px></img>

    * **台灣中型指數（中型100）**
    * <img src="https://i.imgur.com/GBRanM7.png" height=300px></img>

    * **臺灣高股息指數**
    * <img src="https://i.imgur.com/rFmGPI6.png" height=300px></img>



## SmartBeta指數績效

  * **年化報酬**：`0.015654497804251655`
    * 最後的資本利得+總股息配發，與最初的總投資額去算
    * 年化報酬率(%) = (總報酬率+1)^(1/年數) -1
    
  * **年化標準差**：`0.14748305630744052`
    * 月均標準差 * (12^0.5)
  * **sharp ratio**：`0.03230539102959507`
    * (資產平均年化報酬率－無風險利率)/ 資產年化標準差
    * 無風險利率 => 「五大銀行平均基準利率」：2.63 
  * **treynor ratio**：`0.005918222804879886`
    * (資產平均年化報酬率－無風險利率) / 系統風險
    * 先計算系統風險(Beta) => 指數與大盤跑回歸
<br>

## 完全模擬ETF
  * 本金為10億元
  * 以9億為第一期基準乘上成分股權重，並無條件進位到千位（總額介於9億~10億之間）
  * 第二期開始以每期基金總淨值的0.9重複上述動作

<br>

