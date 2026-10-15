# SGLang 貢獻實況檢驗：Slack × GitHub 對照（10/14）

目的是驗證先前的幾個假設：「社群 PR 多半不會被 merge」、「內部 PR 會把社群的工作吃掉」、「@rainj-me 回覆很慢」，並找出實際被 merge 的 PR 有什麼共同點。

資料來源：
- Slack：`#rust-migration` 全部訊息；`#pull-requests` 9/1–10/1 的訊息。
- GitHub：上面提到的每個 PR 的最終狀態，用 `gh` 查詢。

## 1. 假設檢驗結果

| 假設 | 結論 | 證據 |
|---|---|---|
| 社群 PR 多半不會被 merge | **成立，但關鍵是 `run-ci` label** | 9/1–9/15 在 `#pull-requests` 求助的 60 個 PR，一個月後的狀況：merged 15（25%）、closed 5、**仍 open 40**。<br>有拿到 `run-ci` 的：9/14 merged（64%）。<br>沒拿到的：6/46 merged（13%）。 |
| 內部 PR 會吃掉社群的工作 | **沒看到直接案例。真正的風險是「重構讓 PR 失效」和「超出範圍」** | Rain 回 Pai Liu 的 API-key PR：「not urgent… still under refactor, this one may need huge change」，#38276 至今仍 open。<br>sqjian 沒先討論，就做了 PD 管理面的 MVP（#32923）。Rain 當天回覆「現階段不打算把 PD 管理面搬到 Rust」，PR 隔天被關掉。 |
| @rainj-me 回覆很慢 | **在 GitHub 上成立，在 Slack 上不成立** | Slack 上的回覆時間：Sage 當天、sqjian 當天、Brayden 1 分鐘、Pai Liu 8 天。GitHub #23206 底下的認領留言幾乎都沒有回覆。 |
| 核心討論在公開頻道 | **不成立** | Ishan Dhanani（10/6，#general）：「Let's please use the rust refactor channel… that's where all of this conversation is already happening」。那是一個**私人頻道**，搜尋找不到。 |

## 2. 同一個人在 Slack 和 GitHub 兩邊的對照

### Pai Liu（GitHub @Kare0638）：RFC 功能項目，卡住了

1. 8/19：在 GitHub #23206 認領 item 15，Rain 當天回覆，叫他以 #35239 為基礎。
2. 8/31：發 PR #37202。在 GitHub 上沒有任何 review。
3. 9/6：在 Slack `#rust-migration` tag Rain；9/14 又到 `#pull-requests` 發一次。
4. 9/14：Rain 在 Slack 道歉，同一天在 PR 上留了 7 則 inline review。
5. 9/22：Pai Liu 全部修完。
6. **到 10/14 為止仍未 merge，也沒有 `run-ci`。** 另一個 API-key 的 PR 則被 Rain 判定為「重構中，不急」。

→ Slack 能換到 review，但 **RFC 功能項目卡在「重構期，不是優先」**。

### Sage（GitHub @sagearc）：從真實部署痛點切入，變成常客

1. 8/13：在 `#rust-migration` 貼出 k8s 部署時 port range 的問題和 PR #34430，cc 三位 owner。
2. Rain 2 小時內回覆「I believe it is OK for me」。
3. 9/9：merged。
4. 之後又 merge 了 8 個以上的 Rust PR，包括 #39993、#42989、#43004 等，等於已經成了半個核心成員。

→ **真實部署遇到的問題，加上小 PR，再在 Slack 直接 cc owner，是最成功的模式。**

### Bojiang Li：同一個人，兩種求助方式

- **點名該領域的 oncall：** 9/14 tag Liangsheng Yin 看 #39534（小型 IPC weight-update 修正）→ **當天 merged**。另一則 tag BBuf 看 #39482（有在 DGX Spark 上重現）→ 4 天 merged。
- **不點名、一次丟一批：** 9/6 一次貼 4 個 PR，沒有指名任何人（#37702、#37643、#37642、#37641）→ **全部仍 open**。

### sohom-cs（Slack 上是 Suraj Mishra 和 sohomc.iitkgp）：observability bug 一路成功

1. #39312：修正 retract 後 `queue_time` 變成負數的問題，附 unit test 和 before/after。拿到 approval 後去 Slack 求 CI → 9/22 merged。
2. #41380：follow-up，+21/−1，訊息寫明「a follow-up to #39312 (which you merged)」並 tag BBuf → **3 天 merged**。
3. 範圍比較大的 #41290（+211）、#41768、#41771 則還在 open。

→ **latency/metrics 類的 bug fix 是被證實行得通的路，而且正好是你的方向。**

## 3. 被 merge 的 PR 有什麼共同點

1. **Bug fix 多於功能**，而且**附上在 main 上會失敗、修完會通過的測試**。
2. **小**：+10 到 +120 行最常見。
3. **連到一個 issue**，或附上真實環境的重現，例如 production、DGX、B200。
4. **在 Slack 點名該領域的人**：MAINTAINER.md 的 oncall，或最近 merge 過同一區塊的人。
5. **有前一個 merged PR 的延續**，會快很多，因為 reviewer 已經認得你。

被晾著的 PR 則有這些特徵：大型功能、RFC 項目、一次丟一批不點名、碰到正在重構的程式碼（rust-server）。

## 4. 對你的影響

- **item 10（Rust 前端 request latency metrics）的風險偏高。** 它是 RFC 的功能項目，大小 M，而且位在正在重構的 `rust/sglang-server`，跟 Pai Liu 的處境一模一樣。**先在 Slack 問，不要先寫程式。**
- **第一個 PR 建議做 latency/metrics/observability 的小 bug fix。** 可以在 Python 端，也可以是 Rust 跟 Python 之間 metrics 的 parity 差異。條件是：附一個在 main 上會失敗的測試，+100 行以內，修完後在 Slack 點名該領域的 oncall。參考 sohom-cs 的路徑。
- **第二步再提 item 10。** 有了一個 merged PR，再到 `#rust-migration` 問 Rain：「重構期間要不要這個？要的話範圍是什麼？」得到 OK 才開工。

## 5. 新手常識（填補知識盲區）

- **CI 權限：** 新手沒有權限加 `run-ci`，CI 不會跑，GitHub 上的 pr-gate 就會顯示紅叉。**這是正常的**，要去 Slack 請人加。
- **Slack 頻道分工：**
  - `#pull-requests`：求 review 和 CI。訊息很多，**一定要點名**。
  - `#rust-migration`：Rust 前端相關，Rain 會在這裡回覆。
  - `#dev`：一般開發討論。
  - `#ci-cd-build-release`：CI 本身出問題時用。
- **求助訊息的寫法（成功範例的共同結構）：** 一句話講 bug 和影響 → 改動大小（+N/−M）→ 測試「fails on main, passes with fix」→ 連到 issue → 點名 1–2 個人 → 「Could you add run-ci?」
- **認領 issue 沒有正式流程：** 在 issue 底下留言，等同告知其他人，但不保證有人會回應。
