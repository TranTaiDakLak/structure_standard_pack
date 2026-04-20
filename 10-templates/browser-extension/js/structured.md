# Browser Extension JS Structured

## Khi nào dùng

- extension có nhiều feature
- cần chia rõ shared/platform/feature logic

## Cây thư mục

```text
<extension-name>/
├── docs/                       # tài liệu, flow, ghi chú kỹ thuật
├── infra/                      # config build/package
├── scripts/                    # script build/watch/package
├── config/                     # file cấu hình mẫu
├── build/                      # output bundle — gitignore
├── src/                        # vùng source chính
│   ├── background/             # service worker MV3 — chỉ dispatch event
│   ├── content/                # content script — bridge trang ↔ extension
│   ├── popup/                  # UI popup
│   ├── options/                # trang Options
│   ├── features/               # logic theo domain — mỗi feature độc lập
│   │   ├── auth/               # ví dụ feature auth
│   │   ├── collector/          # ví dụ feature thu thập dữ liệu
│   │   └── sync/               # ví dụ feature đồng bộ server
│   ├── platform/               # wrapper browser API
│   │   ├── storage/            # wrap chrome.storage
│   │   ├── messaging/          # wrap chrome.runtime.sendMessage
│   │   └── browser-api/        # wrap các API khác (tabs, alarms...)
│   ├── shared/                 # helper dùng chung — rất tiết chế
│   ├── assets/                 # icon, ảnh, css
│   └── manifest.json           # manifest extension (MV3)
├── tests/                      # test nằm ngoài source chính
│   ├── unit/                   # unit test feature + platform
│   └── integration/            # integration test (mock browser API)
├── package.json                # dependency + script build
├── README.md                   # hướng dẫn repo
└── .gitignore
```

## Vai trò thư mục

- `features/`: feature logic
- `platform/`: browser APIs, messaging, storage wrappers
- `shared/`: shared helpers rất tiết chế

## Rule

- không để `shared/` thành thùng rác
- browser API wrappers nên đi qua `platform/`
- feature logic không rải khắp background/content/popup
