## UI5バージョンの確認
## 目的
SAP Build Work Zoneで実行するUI5アプリのバージョンを固定する方法について検証する

## 方法
1. manifest.jsonの`sap.platform.cf.ui5VersionNumber`で固定
2. SAP Build Work Zoneを開く時、URLパラメータに`&sap-ui-version=<version>&sap-iframe-params=sap-ui-version`を指定

## サマリ
- 1,2の方法いずれでも、指定したUI5バージョンでアプリが実行されることを確認した
- 1の方法では、アプリごとに異なるUI5バージョンを指定できた
- 1,2を併用するとエラーになった（URLパラメータとアプリが同じバージョンでもエラー）

## 注意事項
- サイトが"Spaces and Pages - New Experience"の場合、Space and Pages URLパラメータの指定では、1.146未満のバージョンを指定した場合、サイトそのものがロードできなかった（Groupレイアウトでは表示可能）

## アプリの設定
- noversion
```
  "sap.ui5": {
    ...
    "dependencies": {
      "minUI5Version": "1.148.1",
      ...
    },
```

- version136
```
  "sap.ui5": {
    ...
    "dependencies": {
      "minUI5Version": "1.136.0",
      ...
    },
  "sap.platform.cf": {
        "ui5VersionNumber": "1.136.x"
  }    
```

- version145
```
  "sap.ui5": {
    ...
    "dependencies": {
      "minUI5Version": "1.145.0",
      ...
    },
  "sap.platform.cf": {
        "ui5VersionNumber": "1.145.x"
  }  
```

## 検証結果
### ローカル実行時
- デフォルトのUI5バージョンは、manifest.jsonの`minUI5Version`で指定したバージョン

- noversion: 1.148.1
- version136: 1.136.0
- version145: 1.145.0

### HTML5 Applicationsで実行時 (index.html)

- noversion: 1.149.0 *UI5バージョンが上がったため
- version136: 1.149.0
- version145: 1.149.0

### BWZ実行時
- Shellのバージョン：1.148.1
- noversion: 1.148.1
- version136: 1.136.19
- version145: 1.145.3

### BWZのパラメータでバージョン指定
`&sap-ui-version=1.147&sap-iframe-params=sap-ui-version`を指定
`https://3d7fa4ddtrial.launchpad.cfapps.us10.hana.ondemand.com/site?siteId=57d3a691-61c5-4675-9459-3fb6ef65eab6&sap-ui-version=1.147&sap-iframe-params=sap-ui-version#Shell-home`

- Shellのバージョン：1.147.2
- noversion: 1.147.2
- version136: エラー
- version145: エラー

## アプリ側でバージョンを固定した場合、URLパラメータのバージョンと衝突してエラーになる

ui5appruntimeというリソースに着目

![Internal Server Error](docs/images/internal%20server%20error.png)

原因と回避策の詳細は [docs/原因の調査.md](docs/原因の調査.md) を参照。
