# パターン1：完全バニラ構成（HTML / CSS / JavaScript のみ）

## 概要

フレームワークや Node を使わず、  
**HTML・CSS・素の JavaScript（ES Modules）だけで完結**する構成。

静的ページが多く、UIロジックも軽いプロジェクト向け。

---

## 利用技術

- **HTML**
- **CSS**
- **JavaScript（ES Modules）**
- `fetch` を利用した API 呼び出し
- バンドラなし（npm 不要）

すべてブラウザ標準機能で動くため、学習コストが最小。

---

## ディレクトリ構成（例）

```
public/
  index.html
  users.html
  settings.html

  css/
    base.css
    layout.css
    components.css

  js/
    main.js              // 共通初期化（メニュー等）

    api/                 // ← API呼び出し（HTTP層）
      httpClient.js
      usersApi.js

    services/            // ← 画面ユースケース（業務ロジック）
      usersService.js

    ui/
      components/        // UIパーツ（DOM操作）
        table.js
        modal.js

      pages/             // 各HTMLのエントリポイント
        usersPage.js

    utils/
      dom.js
      validators.js
```

---

## レイヤー構造

### 1. API層（`api/`）

- `fetch` を包んだ **通信専用レイヤー**
- API仕様書と 1 対 1 に対応する関数を定義

```js
// js/api/httpClient.js
export const request = async (path, options = {}) => {
  const res = await fetch(path, {
    headers: { "Content-Type": "application/json" },
    ...options,
  });

  if (!res.ok) {
    throw new Error(`HTTP Error: ${res.status}`);
  }
  return res.json();
};
```

```js
// js/api/usersApi.js
import { request } from "./httpClient.js";

export const fetchUsers = () => request("/api/users");
export const updateUser = (id, name) =>
  request(`/api/users/${id}`, {
    method: "PUT",
    body: JSON.stringify({ name }),
  });
```

---

### 2. サービス層（`services/`）

- **画面単位の仕事**（一覧取得、更新、整形など）をまとめる
- UI に渡すデータを整備する役割

```js
// js/services/usersService.js
import { fetchUsers, updateUser } from "../api/usersApi.js";

export const loadUserList = async () => {
  const users = await fetchUsers();
  return users.map((u) => ({
    id: u.id,
    displayName: `${u.lastName} ${u.firstName}`,
  }));
};

export const changeUserName = async (id, name) => {
  await updateUser(id, name);
  return loadUserList();
};
```

---

### 3. UI 層（`ui/`）

#### UIパーツ（components）

DOM操作を行う「部品」。

```js
// js/ui/components/table.js
export const renderTable = (root, rows) => {
  root.innerHTML = "";

  const table = document.createElement("table");

  rows.forEach((row) => {
    const tr = document.createElement("tr");
    tr.innerHTML = `
      <td>${row.displayName}</td>
    `;
    table.appendChild(tr);
  });

  root.appendChild(table);
};
```

#### ページエントリ（pages）

HTMLごとに 1 つ対応する「初期化スクリプト」。

```js
// js/ui/pages/usersPage.js
import { loadUserList, changeUserName } from "../../services/usersService.js";
import { renderTable } from "../components/table.js";

const init = async () => {
  const tableRoot = document.querySelector("#users-table");

  const reload = async () => {
    const rows = await loadUserList();
    renderTable(tableRoot, rows);
  };

  await reload();

  const form = document.querySelector("#user-form");
  form.addEventListener("submit", async (e) => {
    e.preventDefault();

    const id = form.elements["id"].value;
    const name = form.elements["name"].value;

    await changeUserName(id, name);
    await reload();
  });
};

document.addEventListener("DOMContentLoaded", init);
```

HTML 側：

```html
<script type="module" src="/js/ui/pages/usersPage.js"></script>
```

---

## メリット

- 現在のHTML/CSS/JSのスキルだけで開発できる
- API仕様書ベースの開発と相性が良い  
  → API層を見れば通信処理が完結
- UIロジックが小さく、静的ページが多いケースで十分機能する
- 将来 Node / TS / React などへ **段階的に発展できる設計**

---

## デメリット

- 大規模になるとファイル管理や依存関係が複雑になりやすい
- コンポーネント・状態管理などは自前で実装する必要がある
- テストを書く場合は工夫が必要

---

## まとめ

**バニラJSでも、API層・サービス層・UI層の3分割を意識するだけで、  
フレームワーク並みに読みやすく拡張しやすい構成になる。**

- 小〜中規模ならこれで十分
- フレームワーク移行も見据えた“育てられるアーキテクチャ”
