# Design: `/flow` FE Versioning + Conflict-Free Promotion

- **Date:** 2026-07-22
- **Status:** Draft (pending review)
- **Author:** agaaaptr + Claude
- **Related:** ai-toolkit plugin `ai-dev-workflow` skill `/flow`; partnership repo `lib-uii-gateway-partnership-angular` (`@uiigateway/partnership`, Angular 13.3)
- **Source docs:** `STANDARDS.md`, `DEVELOPMENT_WORKFLOW_GUIDE.md` (partnership repo)

---

## 1. Problem (root cause)

Merge conflict saat promotion (`develop → staging → master`) terjadi di **baris `version` pada `package.json`** dan **header `CHANGELOG.md`**. Akar masalah:

- Suffix env (`-bN` / `-rcN` / polosan) **di-commit ke `projects/uiigateway/partnership/package.json` per branch** → 3 long-lived branch punya 3 string `version` berbeda di file yang sama yang saling di-merge.
- Job `publish` di CI membaca `version` dari `package.json` apa adanya (tidak derive dari tag) → suffix **WAJIB** di-commit → konflik *inherent*, bukan kelalaian.
- `CHANGELOG.md`: tiap env prepend entry suffix-nya sendiri → pasangan `X.Y.Z-b0` + `X.Y.Z-rc0` per versi, plus tabrakan di puncak tiap promotion.

**Bukti (repo partnership, branch `develop`):**
- `projects/uiigateway/partnership/package.json` → `"version": "2.0.23-b0"` (suffix di-commit).
- Edit history `CHANGELOG.md` berisi pasangan: `2.0.19-b0` & `2.0.19-rc0`; `2.0.15-b0` & `2.0.15-rc0`; `2.0.14-b0` & `2.0.14-rc0`; …
- Merge commit CHANGELOG beruntun (`Merge branch 'staging' into 'develop'`, `Merge develop into staging for 2.0.18-rc0 release`) = titik resolusi konflik manual yang berulang.

---

## 2. Goals & Non-goals

**Goals**
- Hilangkan konflik baris `version` antar env (suffix hidup **hanya di tag**).
- Hilangkan duplikasi entry CHANGELOG per-suffix.
- `/flow` melakukan **commit per scope**, dan menawarkan **version-bump + tagging otomatis** untuk Angular FE v13 `@uiigateway/*`.

**Non-goals (di luar scope task ini)**
- Mengubah model branch (dual-track `develop`/`staging`/`master` tetap).
- Mengotomasi merge promotion (`develop→staging→master`) — tetap manual.
- Improve skill selain `/flow` (task terpisah, akan dispesifikaskan kemudian).
- Wrapper repo (downstream; hanya membaca artifact npm).

---

## 3. Design

### 3.1 CI fix (partnership `.gitlab-ci.yml`) — derive version from tag

Pada job `publish`, set `version` dari `$CI_COMMIT_TAG` **setelah `cd dist/...` dan sebelum `npm publish`**:

```yaml
    - cd dist/uiigateway/partnership
    - npm version --no-git-tag-version --allow-same-version "$CI_COMMIT_TAG"
    - echo '//npm.uii.ac.id/:_authToken=${NPM_AUTH_TOKEN}'>.npmrc
    - npm publish --registry https://npm.uii.ac.id
```

- **Suffix source = `$CI_COMMIT_TAG`** (tag sudah bawa suffix per konvensi branch). CI **tidak membaca branch**: di tag pipeline GitLab `$CI_COMMIT_BRANCH` kosong, dan counter `-bN` sulit di-auto-derive. Tag = sumber tunggal.
- Job sudah tag-triggered (`rules` match `-bN` / `-rcN` / polosan) → `$CI_COMMIT_TAG` selalu terisi saat job ini jalan.
- `--allow-same-version` wajib agar **master** (tag polosan == version polosan) tidak error "Version not changed".

### 3.2 Version model

- `projects/uiigateway/partnership/package.json` → **version polosan** (tanpa suffix). Bump hanya menaikkan base version.
- **Tag** → bawa suffix sesuai konvensi (develop `-bN`, staging `-rcN`, master polosan). Konvensi lama, tidak berubah.
- **CHANGELOG** → entry **polosan**, **satu per versi** (dibuat sekali di `develop` saat bump, terbawa `staging → master` tanpa entry baru).
- Pemetaan env→suffix tetap **konvensi manusia saat tagging** (bukan logika CI).

### 3.3 `/flow` skill changes (Option A)

#### (a) Phase 5 (Execute) — commit-per-scope, disuntikkan sebelum delegasi

Sebelum `Invoke superpowers:executing-plans`, berikan instruksi **binding** ke eksekusi:

> "Commit perubahan dikelompokkan **per scope**. Dilarang membungkus banyak perubahan tidak terkait dalam satu commit. Kategorikan per conventional-commit type/scope (`feat` / `fix` / `refactor` / `docs` / `test` / `chore`) dan commit tiap scope terpisah. Gabungkan hanya perubahan yang termasuk satu unit logis."

Tambahkan ke **Hard rules**:
> "Commit per scope; dilarang satu commit untuk banyak perubahan tidak terkait."

#### (b) Phase 7 (Document) — version-bump bersyarat (Angular FE v13 lib)

Di akhir Phase 7 (setelah update docs + tidy), jalankan langkah rilis bersyarat.

**Deteksi** — project adalah Angular FE v13 `@uiigateway/*` library publishable **iff** ketiganya benar:
1. root `package.json`: `@angular/core` version `~13.x` / `^13.x`, **dan**
2. ada `projects/<scope>/<lib>/package.json` dengan field `version`, **dan**
3. `name` di package.json library itu match `@uiigateway/*`.

Jika **tidak match** → skip (behavior `/flow` seperti sedia kala).
Jika **match** → `AskUserQuestion`: "Version action?" — opsi `patch` / `minor` / `major` / `next-pre-release` / `skip` + tampilkan current plain version.
- `patch`/`minor`/`major` → new plain version; suffix **reset ke 0** (versi baru tidak punya tag).
- `next-pre-release` → plain version **tetap**; suffix +1 (`b0→b1`, `rc0→rc1`).
- `skip` → selesai.
Lalu jalankan **Version-bump procedure** (§3.4).

### 3.4 Version-bump procedure (dijalankan skill)

1. Baca `version` (polosan) dari `projects/<scope>/<lib>/package.json`.
2. Tentukan **target version** & perubahan file:
   - `patch`/`minor`/`major` → target = new plain version; `package.json` + `CHANGELOG.md` diubah.
   - `next-pre-release` → target = current plain version (tidak berubah); `package.json` tidak diubah.
3. **(hanya patch/minor/major)** Edit `package.json` → new plain version; prepend entry `CHANGELOG.md` `## <tanggal hari ini> (<new plain version>)` + bullet ringkasan (dari task/commits sesi; **match gaya entry terbaru**; **polosan, tanpa suffix**); commit `chore: bump version to <new version>`. **(next-pre-release)** `package.json` tetap; CHANGELOG opsional append bullet ke entry versi itu (konfirmasi user).
4. Branch saat ini: `git rev-parse --abbrev-ref HEAD`. Suffix kind: `develop`→`b`, `staging`→`rc`, `master`→(none/polosan), **branch lain**→`AskUserQuestion` (jangan tebak).
5. **Counter `N` (deterministik):** `git tag -l "<target>-<kind>*"` → daftar tag untuk versi target + suffix kind; parse **integer** setelah `<kind>` dari tiap match; `N = max + 1`, atau `0` bila kosong. Bandingkan **numerik** (`b10` > `b9`, bukan leksikal).
   - **Reset otomatis saat versi berubah:** versi baru tidak punya tag match → `N=0`. Jadi urutan `2.1.0-b0 → 2.1.0-b1` (next-pre-release) `→ 2.1.1-b0` (patch bump, reset) bekerja tanpa kasus khusus.
6. Tag: `develop`→`<target>-b<N>`; `staging`→`<target>-rc<N>`; `master`→`<target>` (polosan, tanpa counter).
7. `git push origin <branch>` (jika ada commit baru) agar tag menunjuk commit yang ada di remote.
8. `git tag -a "<tag>" -m "release: <target>"` ; `git push origin "<tag>"`.
9. Konfirmasi (informasional) bahwa CI publish ter-trigger oleh tag.

### 3.5 CHANGELOG mitigation (repo-level)

Buat/tambah `.gitattributes` di repo partnership:
```
CHANGELOG.md merge=union
```
Driver `union` bawaan git menyatukan kedua sisi hunk yang konflik → untuk pola "prepend entry baru di atas", **kedua entry disimpan tanpa conflict marker**. Aman karena pola edit CHANGELOG = menambah entry, bukan mengedit entry lama.

Lapis utama tetap model polosan (§3.2) yang menghilangkan duplikasi per-suffix; `union` hanya menangani **residual** top-prepend saat `develop` (2.1.x) dan `master` (2.0.x) divergen.

---

## 4. Files affected

### Repo partnership (`lib-uii-gateway-partnership-angular`), branch `develop`
- `.gitlab-ci.yml` — derive-from-tag (§3.1).
- `projects/uiigateway/partnership/package.json` — set version **polosan** sekali (bersihkan suffix yang ada).
- `CHANGELOG.md` — entry berikutnya polosan (tidak rewrite historis).
- `.gitattributes` — **baru**, `CHANGELOG.md merge=union` (§3.5).
- `STANDARDS.md` — update (lihat §5).
- `DEVELOPMENT_WORKFLOW_GUIDE.md` — update (lihat §5).

### Repo ai-toolkit
- `plugins/ai-dev-workflow/skills/flow/SKILL.md` — Phase 5 commit-per-scope (§3.3a), Phase 7 version-bump (§3.3b + §3.4), Hard rules.

---

## 5. Doc deltas (`STANDARDS.md` & `DEVELOPMENT_WORKFLOW_GUIDE.md`)

- **Version Management / `package.json`:** version **polosan**; hapus instruksi commit suffix ke `package.json`.
- **Tag Naming:** tetap (suffix di tag).
- **CI/CD:** tambah penjelasan derive-from-tag (`npm version --no-git-tag-version --allow-same-version "$CI_COMMIT_TAG"`).
- **CHANGELOG:** entry polosan, satu per versi; **tidak ada** entry per `-rcN`/`-bN`.
- **Baru:** `.gitattributes` `CHANGELOG.md merge=union`.
- **Workflow (feature/hotfix/release):** Maintainer bump = plain version + CHANGELOG + tag per branch; suffix di-inject CI saat publish.
- **Perbaiki inkonsistensi:** §8.2 (menulis plain `2.1.1` di package.json sambil tag `2.1.1-b0` — sekarang konsisten: package.json selalu polosan) & §9.3 langkah 2 (komen membingungkan "ubah 2.1.5 → 2.1.5 (tetap)").

---

## 6. Execution order

1. **Repo partnership:** `git checkout develop && git pull origin develop` → apply perubahan §4 (partnership) → **commit per scope** → `git push origin develop`.
2. **Repo ai-toolkit:** apply perubahan §4 (flow) → commit → push (branch sesuai konvensi ai-toolkit).
3. Improve skill lainnya = **task terpisah** (akan dispesifikaskan kemudian).

---

## 7. Risks / caveats

- `npm version` menerima literal version ber-prerelease (`2.0.26-b1`) — valid.
- Tag di branch non-standar: skill **bertanya**, tidak menebak.
- `merge=union` berisiko duplikat **hanya** bila kedua sisi menambah entry versi yang sama — di model polosan tidak terjadi (satu entry per versi, dibuat sekali).
- Histori CHANGELOG lama (ber-suffix) **tidak** di-rewrite; hanya entry baru yang polosan.
- `npm pack` (sebelum `cd dist`) membuat `.tgz` bernama versi polosan — kosmetik saja, tidak memengaruhi version ter-publish (`npm publish` memakai `package.json` di cwd, bukan tgz).
- Histori commit `chores:` / `bumpe` **tidak** diikuti; standardisasi `chore:` sesuai `STANDARDS.md`.

---

## 8. Decision log

| # | Decision | Rationale |
|---|---|---|
| Skill | Option A (`/flow`), bukan `/wrap` | `/flow` = tempat commit terjadi (Phase 5 Execute) & project-aware (Phase 0 `/sync`); `/wrap` tetap cleanup/report |
| CI | derive-from-tag (`$CI_COMMIT_TAG`) | Hilangkan suffix dari `package.json` → hilangkan konflik; wrapper downstream tak terdampak (hanya baca npm) |
| Tag | skill **buat + push** per branch | Otomatis sesuai konvensi `develop`/`staging`/`master` |
| CHANGELOG | polosan + `.gitattributes merge=union` | Hilangkan duplikasi per-suffix + auto-resolve residual top-prepend |
| Commit msg | `chore: bump version to X.Y.Z` | Sesuai `STANDARDS.md` + conventional commits |
| Deteksi | Angular 13 + lib publishable + `@uiigateway/*` | Pilihan user |
| Scope | `/flow` saja | Improve skill lain = task terpisah |
