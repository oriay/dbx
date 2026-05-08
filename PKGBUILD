# Maintainer: jinzhongjia <mail@nviemr.org>
pkgname=dbx
pkgver=0.4.3
pkgrel=1
pkgdesc="Open-source database management tool (Tauri-based)"
arch=('x86_64')
url="https://github.com/t8y2/dbx"
license=('AGPL-3.0-only')
depends=(
    'webkit2gtk-4.1'
    'gtk3'
    'sqlite'
    'unixodbc'
    'openssl'
    'hicolor-icon-theme'
)
makedepends=(
    'rust'
    'cargo'
    'nodejs'
    'pnpm'
    'pkgconf'
    'git'
)
provides=("$pkgname")
conflicts=("$pkgname-bin")
# makepkg's global LTO option injects -flto=auto / -C linker-plugin-lto, which
# breaks final-link symbol resolution against several bundled native static
# archives in our dep tree (ring, aws-lc-sys). Disable for this package.
options=('!lto')
source=("$pkgname-$pkgver.tar.gz::$url/archive/refs/tags/v$pkgver.tar.gz")
sha256sums=('7f2545b3a977691f79f2167a793853506fa095fa860640b75d0362b8bdbdcd29')

prepare() {
    cd "$pkgname-$pkgver"

    # Keep all build state inside $srcdir, never touch user $HOME
    export CARGO_HOME="$srcdir/.cargo"
    export npm_config_cache="$srcdir/.npm"
    pnpm config --location project set store-dir "$srcdir/.pnpm-store"

    # Pre-fetch JS and Rust deps so build() can run without network.
    pnpm install --frozen-lockfile
    (
        cd src-tauri
        cargo fetch --locked --target "$CARCH-unknown-linux-gnu"
    )
}

build() {
    cd "$pkgname-$pkgver"

    export CARGO_HOME="$srcdir/.cargo"
    export RUSTUP_TOOLCHAIN=stable

    # Force libsqlite3-sys to use system sqlite via pkg-config rather than
    # bundling. sqlx 0.8 unconditionally enables sqlx-sqlite/bundled when
    # the `sqlite` feature is on, but cargo fails to link the resulting
    # static libsqlite3.a into both the proc-macro cdylib (sqlx_macros.so)
    # and the final binary, leaving sqlite3_* symbols undefined. The env
    # var is checked first in libsqlite3-sys's build.rs and overrides the
    # bundled code path.
    export LIBSQLITE3_SYS_USE_PKG_CONFIG=1

    # Disable LTO. With lto="thin" (set in workspace Cargo.toml) plus rust-lld,
    # native static libs from build scripts (aws-lc-sys, ring, etc.) end up
    # with undefined symbols at final link. Disabling LTO restores normal
    # static-archive resolution.
    export CARGO_PROFILE_RELEASE_LTO=false

    # Frontend + backend in one step; skip bundling, we install files ourselves
    pnpm exec tauri build --no-bundle
}

package() {
    cd "$pkgname-$pkgver"

    # Binary (workspace target dir is at repo root, not under src-tauri/)
    install -Dm755 "target/release/dbx" \
        "$pkgdir/usr/bin/dbx"

    # Icons (hicolor)
    install -Dm644 "src-tauri/icons/32x32.png" \
        "$pkgdir/usr/share/icons/hicolor/32x32/apps/dbx.png"
    install -Dm644 "src-tauri/icons/128x128.png" \
        "$pkgdir/usr/share/icons/hicolor/128x128/apps/dbx.png"
    install -Dm644 "src-tauri/icons/128x128@2x.png" \
        "$pkgdir/usr/share/icons/hicolor/256x256/apps/dbx.png"
    install -Dm644 "src-tauri/icons/icon.png" \
        "$pkgdir/usr/share/pixmaps/dbx.png"

    # Desktop entry
    install -dm755 "$pkgdir/usr/share/applications"
    cat > "$pkgdir/usr/share/applications/dbx.desktop" <<EOF
[Desktop Entry]
Name=DBX
Comment=$pkgdesc
Exec=dbx %U
Icon=dbx
Terminal=false
Type=Application
Categories=Development;Database;
StartupWMClass=DBX
EOF

    # License
    install -Dm644 LICENSE \
        "$pkgdir/usr/share/licenses/$pkgname/LICENSE"
}
