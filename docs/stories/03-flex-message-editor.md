# Story 03: LINE Flex Message 編輯器

## 描述
建立一個視覺化的 LINE Flex Message 編輯器，讓使用者能夠透過拖拉或設定介面來設計 Flex Message，而不需要手動編寫 JSON。

## 使用者故事
作為訊息設計者，我希望能夠使用視覺化編輯器來設計 LINE Flex Message，以便更直觀地創建美觀的訊息卡片，而不需要了解複雜的 JSON 結構。

## 驗收標準
- [ ] 提供視覺化編輯介面
- [ ] 支援所有 Flex Message 元件（box, text, image, button, separator, filler, spacer, icon）
- [ ] 即時預覽功能（手機尺寸模擬）
- [ ] 支援拖拉調整元件順序
- [ ] 元件屬性編輯面板（顏色、大小、對齊、間距等）
- [ ] JSON 匯出/匯入功能
- [ ] 預設模板庫（常用樣式）
- [ ] 儲存自訂模板
- [ ] 整合模板語法（可插入變數）
- [ ] 響應式設計（桌面和平板適用）

## UI 功能模組
1. **元件面板**：列出所有可用的 Flex Message 元件
2. **畫布區域**：即時預覽和編輯區域
3. **屬性面板**：選中元件的屬性編輯
4. **模板庫**：預設和自訂模板選擇
5. **程式碼檢視**：顯示生成的 JSON

## 技術需求
- 前端框架（React/Vue/Svelte）
- 拖拉功能庫（dnd-kit, react-beautiful-dnd 等）
- Flex Message JSON 生成邏輯
- 手機預覽模擬器
- 模板儲存 API

## 參考資料
- [LINE Flex Message Simulator](https://developers.line.biz/flex-simulator/)
- [LINE Flex Message 文檔](https://developers.line.biz/en/docs/messaging-api/using-flex-messages/)

## 相依性
- Story 02（模板語法整合）
- 前端開發環境

## 優先級
高

## 預估工時
5-7 天
