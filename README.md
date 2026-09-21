# Pixel Perth Quest — 維護與部署說明

生日點數遊戲：單檔 `v4.html`（無 build 步驟），帳號系統使用 Firebase（Email/Password + Firestore）。

## 檔案角色

| 檔案 | 說明 |
|---|---|
| `v4.html` | 主遊戲檔（唯一改動對象） |
| `v4.txt` | `v4.html` 的 byte 同步副本（用於若干舊部署），改完一定要重新複製 |
| `index.html`/`v2.html`/`v3.html` | 舊版，不要改 |
| `background/json/map*.json` | 部署後實際使用的碰撞資料（file:// 離線時改用 v4 內嵌備援） |

## 每次改檔後的同步（一定要做）

```powershell
Copy-Item v4.html v4.txt -Force
# 比對兩者 MD5 應相同
(Get-FileHash v4.html -Algorithm MD5).Hash
(Get-FileHash v4.txt  -Algorithm MD5).Hash
```

## Firebase 設定

- 專案：`w-bd-7bded`
- Authentication → Sign-in method → **Email/Password** 已啟用
- config 已內嵌在 `v4.html` 的 `<head>`（SDK 用 12.18.0 compat CDN）

### Firestore 安全規則（Firestore → Rules → Publish）

```firestore
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {
    match /profiles/{uid} {
      allow read, update, delete: if request.auth != null && request.auth.uid == uid;
      allow create: if request.auth != null && request.auth.uid == uid && request.time < timestamp(2026, 10, 3, 16, 0, 0);
    }
  }
}
```

`timestamp(2026,10,3,16,0,0)` = 10/04 00:00 台北時間（UTC+8），即「去年 10/4 起不能再建新角色」。

### 加測試帳號

1. console → Authentication → Users → **Add user**，填信箱+密碼。
2. 用該信箱登入遊戲 → 系統會自動建立角色（無角色時跳創角）。
3. 或直接在遊戲登入頁填新信箱按 **新玩家註冊**（活動期間內）。

## 測試與部署

- **不要用 file:// 測 Firebase**（SDK 載入不穩定）。本機測試：
  ```
  npx http-server -p 8080
  ```
  再開 http://localhost:8080/v4.html
- 部署：上傳 GitHub Pages（https）後全部功能正常，fetch JSON 無 CORS 問題。
- 測試流程：註冊 → 創角 → 遊玩 → 登出重登 → 資料（點數/角色/照片）應在。

## 已知架構決策

- 照片：拍照即壓成 320px / jpeg q0.4，存進 Firestore（單份 profile < 1MiB）；已移除 IndexedDB 那套只寫不讀的備援。
- 存檔：`saveProfile()` 為 400ms debounce 寫入 `profiles/{uid}`。
- 日期閘門：client（`todayStr()` 字串比較）只當 UX 提示，真正擋板在 Firestore 規則。
- 每日問答 `dailyAnswers` 每過日自動清空。
- `photoView` 已納入 overlay 管理（`closePhotoView`）。