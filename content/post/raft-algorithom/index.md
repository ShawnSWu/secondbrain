---
title: Raft Algorithom
description: A kind of consensus Algorithom
slug: raft-algorithom
date: 2022-03-06 00:00:00+0000
image: images/cover.jpeg
categories:
    - Algorithom
weight: 1       # You can add weight to some posts to override the default sorting (date descending)
---

不同於Paxos，Raft使用Leader(領導者)、Follower(追隨者)等更直觀的術語
並且簡化了複雜的流程，主要還是有三個流程

1. {{< font color="#E6CC93" size="22px">}}Leader Election{{< /font >}}
2. {{< font color="#E6CC93" size="22px">}}Log Replication{{< /font >}}
3. {{< font color="#E6CC93" size="22px">}}Log Re{{< /font >}}


在Raft中，有以下三個角色代表不同節點
{{< figure src="images/Raft_01.png" height="500" width="600">}}


## 1. Leader Election

   1. **{{< font color="#E6CC93" size="20px">}}初始化狀態{{< /font >}}**： - 系統中的所有節點開始時都處於Follower狀態。 
{{< figure src="images/Raft_02.png" height="300" width="400">}}

   2. **{{< font color="#E6CC93" size="20px">}}超時觸發選舉{{< /font >}}**： - 每個Follower節點在一定時間內沒有收到來自Leader的心跳訊號(Heartbeat)，它會轉變為Candidate並發起選舉，
	   以下舉例為節點初始化時的狀態(沒任何Leader 傳 heaerbeat)，每個節點的進度條則代表沒收到心跳訊號的時間
{{< figure src="images/Raft_03.gif" height="500" width="600">}}

   3. **{{< font color="#E6CC93" size="20px">}}發送投票請求{{< /font >}}**： - Candidate節點向其他所有節點發送RequestVote請求，並附帶其當前的日誌索引和任期號。 
{{< figure src="images/Raft_04.gif"  height="500" width="600">}}


4. **{{< font color="#E6CC93" size="19px">}}接受投票{{< /font >}}**： - 其他節點（Followers）收到RequestVote請求後，會根據Candidate的日誌索引和任期號決定是否投票   
	   如果該Candidate的日誌比自己更新，且尚未投票給其他Candidate，則會投票給該Candidate。
      
      > {{< font color="#88dba3" size="20px">}}這裡是Raft算法的核心之一，透過網路時間差來投票選出Leader，像上面gif顯示的結尾部分，Candidate變成Leader{{< /font >}}
	   
5. **{{< font color="#E6CC93" size="20px">}}當選為Leader{{< /font >}}**： - Candidate節點如果獲得多數節點的投票（超過半數），則成為Leader。
	當選後，它會立即向其他節點發送心跳訊號，通知其成為新的Leader，以下為正常Leader持續傳送心跳訊號的樣子
{{< figure src="images/Raft_05.gif"  height="300" width="400">}}

	   
6. **{{< font color="#E6CC93" size="20px">}}處理失敗情況{{< /font >}}**： - 如果Candidate在一定時間內沒有獲得足夠的投票，它會重新進入Follower狀態，並等待下一次選舉超時再次發起選舉。
	   
	以下是當Leader掛掉 不再傳送心跳訊號時時，各節點會再重新選出新Leader的選舉機制（意義上就是回到 **1.初始化狀態** 開始)	   
{{< figure src="images/Raft_06.gif"  height="300" width="400">}}

> {{< font color="#EF9C66" size="18px">}}以下影片為多節點觸發投票機制的情況{{< /font >}}

{{< video autoplay="true" loop="true" src="images/Raft_07.mp4" >}}

## 2. Log Replication 日誌複製

> {{< font color="#EF9C66" size="18px">}}當有了領導者之後，一旦系統有發生改變時，我們就需要將系統的所有變更複製到所有節點{{< /font >}}

1. **{{< font color="#E6CC93" size="20px">}}日誌條目追加{{< /font >}}**：
   當 Leader 接收到一個新的數據變更請求(例如,增加一筆資料),它會將該變更記錄為一個新的日誌條目,並將其追加到其本地日誌中。
   
   以下例子 (綠色節點=Client) 是Client 傳一筆資料 = 5 的資料給 Leader  
{{< figure src="images/Raft_08.gif"  height="500" width="600">}}

   
2. **{{< font color="#E6CC93" size="20px">}}發送 AppendEntries 請求{{< /font >}}**：
   Leader 會並行地將新的日誌條目發送給所有 Follower 節點,通過發送 AppendEntries RPC 請求。每個 Follower 在收到 AppendEntries 請求後,會將新的日誌條目附加到其本地日誌中
{{< figure src="Raft_09.gif"  height="500" width="600">}}


3. **{{< font color="#E6CC93" size="20px">}}回覆成功或失敗{{< /font >}}**：
   Follower 處理完 AppendEntries 請求後,會向 Leader 回覆是否成功附加了新的日誌條目。
{{< figure src="Raft_10.gif"  height="500" width="600">}}

   如果大多數(>50%)Follower 成功複製了日誌條目，則該條目被認為已經被認同了，並將回應傳送給Client端，狀態為已提交(Committed)。

4. **{{< font color="#E6CC93" size="20px">}}應用日誌條目{{< /font >}}**：
   一旦某個日誌條目被提交,Leader 會通知所有 Follower 應用該條目到狀態機器(State Machine)中。這樣,所有節點的數據狀態就保持一致。
{{< figure src="images/Raft_11.gif"  height="500" width="600">}}

      
5. **{{< font color="#E6CC93" size="20px">}}處理網絡分區{{< /font >}}**：
   Raft 甚至可以在網路分區時保持一致性
   如果在日誌複製過程中出現網絡分區,導致 Leader 無法與部分 Follower 通信,則 Leader 會無限期等待這些 Follower 重新上線。一旦重新建立連接,Leader 會自動將它們的日誌複製過來。
   
   讓我們新增一個分割區來將 A 和 B 與 C、D 和 E 分開例子：
      由於我們的分裂，我們現在有兩位不同任期的Leader
{{< figure src="images/Raft_12.gif"  height="500" width="600">}}

   
   我們新增另一個客戶端並嘗試更新兩個Leader的資料
{{< figure src="images/Raft_13.png"  height="500" width="600">}}


   > 各分區開始各做各的
   
   圖中下面的一個Client端將嘗試將節點 B 的值設為『3』，但由於節點 B 因為節點數量不夠多而無法使選舉機制成功，所以無法複製資料到多數節點，因此其日誌條目一直保持未提交狀態   
   (專注看下面的分區)
{{< figure src="images/Raft_14.gif"  height="500" width="600">}}
   
   再來看上面的Client端分區
   Client 將嘗試將節點E的值設為“8”，因為結點多到可以使選舉機制成功，所以其他Follwer也會更新
   
   (專注看上面的分區)
{{< figure src="images/Raft_15.gif"  height="500" width="600">}}

   > 當我們修復網路分割區時

   節點 B 將會看到更新的『選舉任期』，所以無條件接受別人的資料版本，並接受新領導者的日誌，接著，我們的日誌在整個叢集中就變成是一致的了。
{{< figure src="images/Raft_16.gif"  height="500" width="600">}}



通過以上過程,Raft 確保了在任何時候,大多數節點的日誌都是完全一致的。這樣就保證了系統的數據一致性和容錯性。
