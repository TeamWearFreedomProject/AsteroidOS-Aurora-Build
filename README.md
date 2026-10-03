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


## Stage 3: Real kernel modules compilation

- [GitHub Actions → Aurora | compile linux-aurora-modules](https://github.com/TeamWearFreedomProject/AsteroidOS-Aurora-Build/actions/workflows/aurora-compile-modules.yml)
- [First compilation attempt, 2026-10-03](https://github.com/TeamWearFreedomProject/AsteroidOS-Aurora-Build/actions/runs/37105380000)

This job runs `bitbake linux-aurora-modules` **for real** on a standard Ubuntu 22.04 runner. It restores the cached `aurora` layer sources, and builds any required toolchain/kernel dependencies, with parallelism 3, an 8 GiB free-space cutoff, a **320-minute command timeout**, and a **350-minute overall job limit**. **This job does not flash the watch or install software on it.**

Build diagnostics are uploaded in the `aurora-modules-build-logs` artifact even on failure. If compilation completes, generated `linux-aurora-modules*.ipk` packages smaller than 150 MB are uploaded separately as `aurora-modules-ipk-not-flashable`. IPK packages are not boot or flash images. All artifacts are set to expire after 7 days.

**Limitations:** GitHub Actions runners are ephemeral. Finished tasks may now be reused through saved Yocto `sstate-cache`, and downloaded sources can be restored between runs. **Interrupted `do_compile` tasks and build/tmp are not preserved**, so long compilations can still restart. Caches may expire, be evicted, exceed size limits, or fail to upload. The previous stage's cached *layer source repositories* are separate from these caches. A successful build is not proof that any output is safe to flash to a watch.


## Stage 3 troubleshooting: disk-monitor inode threshold (2026-10-03)

[First compilation attempt #37105380000](https://github.com/TeamWearFreedomProject/AsteroidOS-Aurora-Build/actions/runs/37105380000) stopped **before executing any BitBake tasks**. This was caused by our workflow's `BB_DISKMON_DIRS` value `HALT,${TMPDIR},8G,1G`: the second threshold specifies **free inode count**, so `1G` meant one billion available inodes, not an additional 1 GiB of storage. The runner still had approximately 86 GiB free and ample memory.

Corrected to `HALT,${TMPDIR},8G,100K HALT,${DL_DIR},8G,100K`, retaining disk and inode safety thresholds.

[Retry #37106259719](https://github.com/TeamWearFreedomProject/AsteroidOS-Aurora-Build/actions/runs/37106259719) was launched automatically by that change. The retry's outcome must be verified separately; the fix does not establish that the kernel modules compile successfully.

## Stage 3: extended-time retry

[Extended compilation attempt #37116955017](https://github.com/TeamWearFreedomProject/AsteroidOS-Aurora-Build/actions/runs/37116955017) runs with 180 minutes allotted to BitBake, within a 200-minute GitHub Actions job. The earlier [70-minute build #37106259719](https://github.com/TeamWearFreedomProject/AsteroidOS-Aurora-Build/actions/runs/37106259719) was stopped by the timeout after starting task 880 of 1,103, with approximately 68 GiB of free disk remaining. That count indicates scheduled/started tasks, **not completion percentage**. The longer retry begins from scratch (only the source-layer cache is reused).


## Stage 4: resumable completed tasks through Yocto caches (2026-10-03)

[Cached compilation retry #37127840228](https://github.com/TeamWearFreedomProject/AsteroidOS-Aurora-Build/actions/runs/37127840228) is the **first run that saves completed-task caches**. It restores `asteroid/build/downloads` and the most recent `asteroid/build/sstate-cache` before building. It runs BitBake for up to 320 minutes and reserves up to 30 extra minutes for collecting logs and uploading caches, within GitHub's 6-hour hosted-runner job limit.

- `downloads`: saved under a stable cache key the first time; only reused later, not incrementally updated under the same key. Max size checked: 6 GiB.
- `sstate-cache`: restored using a shared key prefix and saved under a fresh unique run key **even if BitBake times out**. Max size checked: 4 GiB.
- `tmp`: intentionally *not* cached (19 GiB at the end of the previous run); **partially running compiler tasks cannot resume from a checkpoint**.
- Cache save failures are non-fatal; check the `Save ... cache` steps and console logs for actual saved/loaded bytes. Restored-cache matches and task reuse are **not guaranteed**.
- GitHub's default repository-wide cache limit is 10 GiB. This workflow **does not change repository billing settings or increase that limit**. Older caches may be evicted automatically.
- The earlier [180-minute retry #37116955017](https://github.com/TeamWearFreedomProject/AsteroidOS-Aurora-Build/actions/runs/37116955017) reached the LLVM/Clang native toolchain dependency and timed out with approximately 63 GiB free disk and 24 GiB in `asteroid/build`. Reached task 1070 of 1103 is a task-start count, **not 97% of build time completed**.

No watch-flashing step is present. Produced IPK packages, if any, are not boot images.
