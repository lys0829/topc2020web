2020 ICPC Asia Taiwan Online Programming Contest 競賽事故彙整與提問回覆說明
===
###### tags: `ICPC` `TOPC` `DOMjudge` `Competitive Programming`

本文選擇以發生時間順序說明來彙整 2020 ICPC Asia Taiwan Online Programming Contest 過程中發生的一些事故，另有夾帶一些 Clarification 與主辦方對應回應的邏輯。

## 18:13:55 系統基礎設定未改為網路賽模式、未明確聲明使用 Standard I/O

當時發的 clarification
![](/image/b65J6YW.png)

網路賽因開發機與裁判機環境不同，因此通常不計 compilation error 的 penalty 。另因目前裁判系統主流已經全使用標準輸出入，但因為沒有聲明，補正後人工移除前面使用檔案輸出入的兩筆 submission ，之後使用檔案 I/O 的提交均使用正常批改流程，該算錯誤就算錯誤。

## 18:43:58 提問 Problem E 

![](/image/rdLciqm.png)

因題本上有寫成下面這樣， another 是指「另一個」。

![](/image/oBxL7Uy.png)

## 18:59:29 Problem F 誤下標

![](/image/U8d6b1j.png)
經隊伍 `NCTU__` 發現後更正。

## 19:02:14 索取額外測資

![](/image/k2ayETS.png)

裁判組回答錯誤，應回答 "No" ，而非 "No comment" 。提供額外測資是特定裁判系統有的功能，高層級的比賽並沒有這項服務。需要明確的告訴隊伍，這場比賽沒有提供額外的測試資料。

## 19:06:36 NTUT_Kn1ghts 因 checker 有誤通過 Problem E

因原本的 checker 漏掉一行檢查是否有重複出現過的字串，因此被錯誤的程式通關。找出錯誤後，於 18:27 分修補完成更新。在賽中並無其他隊伍的提交能透過相同方式通過 Problem E。依據慣例，因該隊伍與命題組沒有利害關係，且此次錯誤是命題組的問題，該題目視為答對。

## 19:45:55 發現凍結計分板時間沒宣告

![](/image/YNk6WaX.png)

## 19:48:14 索取額外測資

![](/image/Gx7ZTJJ.png)
裁判組回答再次錯誤。

## 20:04:00 後：燃燒的裁判環境

本次競賽進入第三小時之後，裁判系統的負擔開始提高，裁判組發現 judge queue 開始變長，於是 20:14 時現場決定從 10 台 `judgehost` 提升到 15 台，並且在狀況沒有好轉的情況下， 20:23 繼續調整到 20 台，之後情況更進一步惡化，使用者前端介面明顯卡頓。著手調整前端伺服器 `domserver` 資源，但在 20:42 發現撞到 GCP 資源上限，只能放棄。最終在 20:50 分時，為了降低批改負擔，切換 Lazy evaluation mode ，將碰到的第一個錯誤回報給系統，而非原先的把最糟結果給系統 (TLE、RE、MLE 比 WA 系列的要糟) 。最終在此設定下，所有的賽內提交在 21:07:31 時批改完成。 

經過賽後的技術調查，我們發現當初「多開 judgehost 來增加 judge queue 消化速度」的直覺是錯誤的。因為 DOMjudge 的系統設計，提供網頁服務的 `domeserver` 效能，會嚴重的影響 `judgehost` 批改一筆 submission 的時間。下圖是所有提交批改開始到結束的時間 (藍線為各筆、橘線為前後十筆的平均值，單位是秒，不包含在 queue 裡面的時間)，橫軸刻度是開始批改的時間 (自開賽起算) ，兩條垂直的紅線是兩次增加 `judgehost` 數量的時間。可以看到有少數幾個提交被批改了近 20 分鐘。

![](/image/2cmyPbX.png)

下圖則是各提交自系統收到後，直到開始批改的時間，也就是每個提交在 judge queue 內的時間 (單位是秒)。橫軸座標是收到提交的時間。可以看到在增加 judgehost 後災難性的提升，直接將平均值推升到超過 20 分鐘，有些極端的甚至將近 40 分鐘才出 judge queue。
另外可能會有隊伍提出質疑，為什麼不一開始就多開一些 `judgehost`，原因是要保留調整空間，且希望不要造成太多的資源空閒，畢竟在 GCP 上只要機器開著就是會算錢。且就這次的分析結果來看，一開始就開多並不會讓參賽者感覺到變快，甚至還有可能因為 `judgehost` 太多拖累 `domserver` 的效能導致 judge ssystem 更慢。這邊補充說明，在 `domjudge` 上，每筆 submission 只會由一台 `judgehost` 負責，並不會因為有空閒的 `judgehost` 而將 submission 拆開給不同的 `judgehost` 跑，所以在 2 小時前幾乎大部分 submission 在上傳後都能立刻被 judge 的情況下，10 台 `judgehost` 的配置應該是合理的。

![](/image/n7TkpYp.png)

至於提交數量，請參考下圖，其實在進入第三小時前後，並無特別的多。

![](/image/eSuPJKg.png)

DOMjudge 分配提交到 `judgehost` 的機制，並非單純 FIFO ，相關的分配規則，[請參考本文說明](https://hackmd.io/@lys0829/r16zLiT8w)。


造成上述現象的主因，讓我們從本次裁判系統的架構圖看起。

![](/image/w9EzcAq.png)

其實 `judgehost` 本身跟參賽者一樣，都是通過 HTTP load balancer 去存取 `domserver` ，當我們增加 `judgehost` 時，其實就是增加 `domserver` 的負擔，進而影響到使用者以及裁判速度：因為批改時，每一筆測試資料 `judgehost` 都需要跟 `domserver` 透過 HTTP 去取：一題測資 100 筆，那就需要花 200 個 HTTP request 。因此，在測試資料最少 21 筆，最多 100 筆的題組，增加這麼多 `judgehost` ，幾乎就是倍增 HTTP request ，因此每個 request 的回應時間變長，導致 `judgehost` 的時間都花在等測資上了。

我們賽後評估，當時正確的措施應該是增加 `domserver` 的資源，甚至是同時下修 `judgehost` 的數量。在打算調整 `domserver` 時，我們有比較多的糾結在準備要如何發出公告、幾分鐘時要調整等等，但最後我們發現我們剛剛好把免費的 GCP 資源用盡，無法進行 `domserver` 機器規格調整。其實在 `domserver` 機器規格無法往上調的時候應該還有一個解決方案是增加 `domserver` 在 `k8s` 上的 `pod` 數量來同時處理更多的 HTTP Request，但由於事前沒做到妥善設定以至於只要開了大於一個 `pod` 就會導致使用者一直被登出（PHP session 儲存的問題），雖然在比賽前就知道多開 `pod` 會有這樣的問題，但我們當時的評估是覺得應該能夠撐得住，因此就沒有在賽前調整。在比賽結束前 10 分鐘，無法在做系統調整的情況下，只能選擇最後手段，修改 Lazy evaluation 降低工作量。

## 20:34:54 詢問 pending 過久

![](/image/mjUVSDJ.png)

我們在 20:10 前後確實有看到一個隊伍送了相同的程式碼幾次，但這個問題的根本理由是我們增加 `judgehost` 導致的，且根據 `domjudge` 的排程方式，同一個隊伍連續送出 `code` 並不會導致後面的 submission 被卡住，我們做出的現場回應與現實不符。在此向提出 Clarification 的隊伍致歉。

## 20:50:28 詢問 pending 過久

![](/image/QCrI7ZW.png)

如前所述，這個問題的根本理由是我們增加 `judgehost` 導致的，當時我們做出的回應與現實不符。在此向提出 Clarification 的隊伍致歉。
