Windows 11 Pro での Hyper-V 環境構築手順
===

## 前提条件

- Windows 11 Pro エディション（HomeエディションではHyper-Vを利用できません）
- CPU が仮想化をサポート（Intel VT-x/AMD-V）していて BIOS で仮想化が有効になっていること
- RAM 4GB以上（推奨8GB以上）
- ストレージ容量に十分な空き


## Hyper-V 環境構築手順

### 手順 1

1. **スタートメニュー** ⇒ **設定** をクリック。
2. **システム** ⇒ **オプション機能** ⇒ **Windows のその他の機能** をクリック。
3. 表示されたダイアログボックスのリストから **Hyper-V** のチェックボックスにチェックを入れ **OK** をクリック。
4. システムの再起動を求められるので再起動します。


### 手順 2

管理者モードで Windows PowerShell または ターミナル を起動して下記コマンドを実行します。

```powershell
Enable-WindowsOptionalFeature -Online -FeatureName Microsoft-Hyper-V -All
```

システムの再起動を求められたら「Y」を入力して再起動します。


## Hyper-V マネージャーの起動

- **スタートメニュー** ⇒ **Hyper-V マネージャー** を検索して起動する。
- または、**Windows キー + R** ⇒ **virtmgmt.msc** を入力し **OK** をクリックする。


## 仮想スイッチの作成
