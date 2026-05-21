# Browser Extension JS Simple

## Khi nào dùng

- extension nhỏ-vừa
- popup, content script, background script cơ bản

## Cây thư mục

```text
<extension-name>/
├── docs/                 # tài liệu, flow, ghi chú kỹ thuật
├── infra/                # config build/package (zip, CRX)
├── scripts/              # script build/watch/package
├── config/               # file cấu hình mẫu
├── build/                # output bundle — gitignore
├── src/                  # vùng source chính
│   ├── background/       # service worker / background script (MV3)
│   ├── content/          # content script inject vào trang web
│   ├── popup/            # UI popup khi click icon extension
│   ├── options/          # trang Options của extension
│   ├── assets/           # icon, ảnh, css tĩnh
│   └── manifest.json     # manifest extension (MV3)
├── tests/                # test nằm ngoài source chính
├── package.json          # dependency + script build
├── README.md             # hướng dẫn repo
└── .gitignore            # bỏ qua node_modules/, build/, *.zip
```

## Vai trò thư mục

- `background/`: background service worker / background script
- `content/`: content scripts
- `popup/`: popup UI
- `options/`: options page
- `manifest.json`: extension manifest

## Rule

- không nhét toàn bộ logic vào content script
- `build/` là output bundle, phải gitignore
- popup logic và background logic tách rõ
- `manifest.json` phải xin quyền tối thiểu; quyền mới phải gắn với feature thật.
- Message contract giữa popup/content/background phải có test hoặc ghi chú rõ.
- Config endpoint/key public đặt trong config mẫu; secret thật không nhúng vào bundle extension.
- Script package/deploy store phải tạo artifact từ source sạch, không dùng file build thủ công.
- Nếu bắt đầu nhiều feature, thêm `features/` và nâng sang `structured.md`.
