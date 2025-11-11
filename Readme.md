# n8n 終極實戰：打造全自動「AI 撰稿 + SEO 發文」機器人

這是一個「怪物級」的 n8n 工作流程。它的目的非常明確：全自動化地決定「該寫什麼」，讓 AI「寫出來」，最後「完美地發布」到 WordPress，連 SEO 和精選圖片都幫你搞定。

本流程串接了 Google Sheets、WordPress、Gemini AI 和圖片生成服務，堪稱是自動化內容行銷的完全體。

---

## 流程概覽：大腦如何運作？

這個流程會分成四大階段：

1.  決策 (Decision)：找出哪個文章分類最「弱」，需要補充內容。
2.  創作 (Creation)：命令 AI 針對這個分類，寫一篇 SEO 完美的文章。
3.  發布 (Deployment)：將文章、SEO 資料、精選圖片發布到 WordPress。
4.  回報 (Reporting)：更新 Google Sheet 上的計數，並 Email 通知管理員。

---

## 節點詳解：一步步拆解

#### 階段一：決策 (找出目標)

1.  HTTP Request (抓取 WP 分類)
     作用：流程一開始，先去 `example.com` 抓回「所有」分類的即時資料，包含名稱、ID 和目前的文章總數 (count)。

2.  Update row in sheet (更新 WP 即時戰情)
     作用：把上一步抓到的「即時」文章數，更新回我們的「Google Sheet 總表」。這確保我們的決策是基於最新數據。

3.  找出各分類數量 (讀取 Sheet 決策表)
     作用：這是我們的大腦！它去讀取 Google Sheet，把所有分類的「IDs」和「數量」全部載入 n8n。

4.  篩選文章數量最少 (Code 節點 - 決策核心)
     作用：這段 Code 會分析上一步從 Sheet 拿到的所有資料，找出「數量」欄位最小的那個分類。接著，它會從 `HTTP Request` 抓來的完整資料中，挑出「勝利者」的完整分類物件，交給下一步。這就是我們的 `target_category`。

#### 階段二：創作 (AI 撰稿)

5.  Message a model1 (Gemini AI 寫手)
     作用：它會接收 `target_category`，然後根據我們設定的「嚴格 SEO 撰寫規則」，命令 Gemini 產出包含 `focus_keyword`、`title`、`content`、`meta_description`、`slug` 和 `image_prompt` 的完整 JSON 包。

6.  解析Gemini 分解為json (Code 節點 - 開罐器)
     作用：AI 回傳的是包在 `text` 欄位裡的一大串「文字」。這個節點 負責用 `JSON.parse()` 把這串文字「撬開」，變回 n8n 看得懂的乾淨 JSON 物件。

#### 階段三：發布 (部署到 WordPress)

7.  發文 (WordPress 節點)
     作用：接收 AI 的 `title`、`content`、`slug`，並指定 `target_category.id`，在 WordPress 網站上建立一篇文章。

8.  關鍵前置作業：授權 Rank Math API
     注意：下一個步驟「更新 SEO」會 100% 失敗，除非你先在你的 WordPress 網站動手腳。
     原因：基於安全考量，Rank Math 預設「不允許」外部 API (像 n8n) 讀寫它的 SEO 欄位。
     解法：你必須在你的 WordPress 網站中，加入以下 PHP 程式碼（推薦使用 `WPCode` 外掛來新增，或是加入子主題的 `functions.php`）：

    ```php
    /
      允許 Rank Math SEO 欄位在 REST API 中顯示與編輯
      來源：Moksa (via Gemini)
     /
    function moksa_register_rank_math_meta_fields() {
        // 註冊 SEO 標題
        register_post_meta( 'post', 'rank_math_title', array(
            'show_in_rest' => true,
            'single' => true,
            'type' => 'string',
            'auth_callback' => function() {
                return current_user_can( 'edit_posts' );
            }
        ));
    
        // 註冊 SEO 描述
        register_post_meta( 'post', 'rank_math_description', array(
            'show_in_rest' => true,
            'single' => true,
            'type' => 'string',
            'auth_callback' => function() {
                return current_user_can( 'edit_posts' );
            }
        ));
    
        // 註冊 SEO 關鍵字
        register_post_meta( 'post', 'rank_math_focus_keyword', array(
            'show_in_rest' => true,
            'single' => true,
            'type' => 'string',
            'auth_callback' => function() {
                return current_user_can( 'edit_posts' );
            }
        ));
    }
    add_action( 'rest_api_init', 'moksa_register_rank_math_meta_fields' );
    ```

9.  更新文章RankMathSEO (HTTP 節點)
     作用：這個節點會呼叫 WP API，把 AI 產生的 `rank_math_title`、`rank_math_description` 和 `rank_math_focus_keyword` 強制寫入文章。

10. 生成文章精選圖片 (HTTP 節點)
     作用：抓取 AI 產生的 `image_prompt`，丟給圖片生成 API，開始算圖。

11. 下載圖片 → 上傳圖片 (HTTP 節點)
     作用：等待圖片算完，下載圖檔，然後把它上傳到 WordPress 媒體庫，並取得新的媒體 ID。

12. 更新文章圖片 (HTTP 節點)
     作用：把上一步的媒體 ID，設定為我們文章的「精選圖片」。

13. 更新文章圖片SEO (HTTP 節點)
     作用：最後一步 SEO 優化，把 AI 產生的 `focus_keyword` 填入精選圖片的 `alt_text` 欄位，Rank Math 會很高興。

#### 階段四：回報 (完成閉環)

14. 抓取分類數量 → 更新分類數量 (HTTP + Google Sheet)
     作用：重新抓一次 WP 分類的最新文章數，並更新回 Google Sheet 總表。這確保下次流程啟動時，會去挑下一個最弱的分類。

15. 寄信通知 (Gmail 節點)
     作用：流程跑完，寄一封 HTML 信 給管理員，包含文章標題、分類和檢查連結，完美收工。

---

## 如何安裝與設定

1.  匯入 JSON：將這份 `example.com` 版本的 JSON 檔案 匯入你的 n8n。
2.  設定憑證：你必須在 n8n 的 `Credentials` 中設定：WordPress、Google Sheets、Google Gemini API Key、圖片生成 API Key 和 Gmail。
3.  準備 Google Sheet：建立一個 Google Sheet，欄位至少要有 `分類`, `IDs`, `數量`。
4.  （最重要） 在你的 WordPress 網站加入上述的 PHP 授權碼。
5.  客製化：把所有 `example.com` 和 `example` ID 換成你自己的網址和 Google Sheet ID。
6.  啟動！
