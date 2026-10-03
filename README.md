# AsteroidOS Aurora Build

Pixel Watch 2 (codename **aurora**) 用 AsteroidOS / Yocto ビルド環境の実験用リポジトリです。

## iPhone から実行

1. [Actions → Aurora | runner check + source preparation](https://github.com/TeamWearFreedomProject/AsteroidOS-Aurora-Build/actions/workflows/aurora-bootstrap.yml) を開く。
2. **Run workflow** → **Run workflow** で実行する（設定ファイル変更時は自動実行）。
3. 最新の実行を開き、**prepare** ジョブのステップを確認する。
4. 完了後、**Artifacts → aurora-bootstrap-logs** に環境・clone・容量のログがある。

## 現在の処理

- Ubuntu 24.04 ランナーの CPU / RAM / ストレージ確認
- AsteroidOS の `whinlatter` ブランチと、公式 `prepare-build.sh` に対応するレイヤーの shallow clone
- Pixel Watch 2 (`MACHINE=aurora`) のビルド設定初期化
- ソースを GitHub Actions キャッシュに保存（容量制限やキャッシュ失効で消える場合あり）

**現段階では `bitbake asteroid-image` を実行せず、時計に書き込むイメージも生成しません。** Actions ランナーは実行終了後に破棄されるため、ソースはキャッシュが保存できた場合にのみ次回復元できます。ログの Artifact は7日保持されます。

ソースは各 upstream の固定されていないブランチ先頭から取得します。将来、再現性を高めるにはコミット SHA の固定が必要です。


## Stage 2: BitBake dry-run

[Actions → Aurora | BitBake metadata and modules dry run](https://github.com/TeamWearFreedomProject/AsteroidOS-Aurora-Build/actions/workflows/aurora-parse.yml)

This workflow restores the Stage 1 cached layers, installs minimal host metadata tools, then runs:

```sh
bitbake -p
bitbake -n linux-aurora-modules
```

The commands **do not compile or flash** the Pixel Watch 2. Inspect the logs in the run's `aurora-bitbake-check-logs` artifact. If the source cache has expired, rerun the Stage 1 workflow first. A green dry run means BitBake can parse metadata and plan tasks; it **does not** demonstrate that the kernel modules compile.



## Confirmed result (2026-10-03)

- [Ubuntu 22.04 BitBake validation run #37104845098](https://github.com/TeamWearFreedomProject/AsteroidOS-Aurora-Build/actions/runs/37104845098): **success**
- `bitbake -p`: **success** (BitBake metadata parsing)
- `bitbake -n linux-aurora-modules`: **success**, 1,103 tasks checked in dry-run mode
- [Validation logs artifact](https://github.com/TeamWearFreedomProject/AsteroidOS-Aurora-Build/actions/runs/37104845098/artifacts/11267946206) (7-day retention)

Ubuntu 24.04 initially failed on its unprivileged user-namespace/AppArmor restriction. The parse/dry-run workflow uses Ubuntu 22.04 to avoid changing the host security configuration.

**Important:** A passing `-n` means the planned dependency/task graph was accepted. No module binaries or watch firmware were built or tested yet.
