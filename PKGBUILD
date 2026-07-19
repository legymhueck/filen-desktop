# Maintainer: MicLeh <micleh at proton dot me>
pkgname=filen-desktop
_pkgname=filen-desktop
pkgver=3.0.52
pkgrel=1
pkgdesc="Cloud storage desktop client for Filen (Built from source tag v3.0.49-rc9)"
arch=('x86_64')
url="https://filen.io"
license=('AGPL-3.0-or-later')
depends=('electron' 'glibc' 'zlib' 'hicolor-icon-theme') 
makedepends=('git' 'npm' 'nodejs')
provides=("${_pkgname}")
conflicts=("${_pkgname}" "${_pkgname}-git" "${_pkgname}-bin-custom")

_tagver="v${pkgver/_/-}"
source=("${_pkgname}::git+https://github.com/FilenCloudDienste/filen-desktop.git#tag=${_tagver}")
sha256sums=('SKIP')

prepare() {
    cd "${srcdir}/${_pkgname}"
    
    # Directly rewrite SSH endpoints to HTTPS inside the manifest files to bypass npm restrictions
    sed -i 's|git+ssh://git@github.com/|git+https://github.com/|g' package.json package-lock.json
    sed -i 's|git@github.com:|git+https://github.com/|g' package.json package-lock.json

    # Move @filen/web to devDependencies and update files array in package.json to prevent packaging all of node_modules
    node -e '
      const fs = require("fs");
      const pkg = JSON.parse(fs.readFileSync("package.json"));
      if (pkg.dependencies && pkg.dependencies["@filen/web"]) {
        pkg.devDependencies["@filen/web"] = pkg.dependencies["@filen/web"];
        delete pkg.dependencies["@filen/web"];
      }
      if (pkg.build && pkg.build.files) {
        pkg.build.files = pkg.build.files.filter(f => f !== "node_modules/**/*");
        pkg.build.files.push("web-dist/**/*");
      }
      fs.writeFileSync("package.json", JSON.stringify(pkg, null, 2));
    '

    # Patch serve.ts to look for web assets in web-dist/ instead of node_modules/@filen/web/dist/
    sed -i 's|"node_modules", "@filen", "web", "dist"|"web-dist"|g' src/lib/serve.ts

    # Patch binary.ts so it looks for rclone in the system-wide filen-desktop location rather than relative to process.resourcesPath
    sed -i 's|return pathModule.join(process.resourcesPath, "app.asar.unpacked", "bin", "rclone")|return "/usr/lib/filen-desktop/app.asar.unpacked/bin/rclone"|g' src/lib/rclone/binary.ts
}

build() {
    cd "${srcdir}/${_pkgname}"
    
    # Run npm installer using the corrected modern allow-git configuration setting
    npm ci --allow-git=all

    # Copy the pre-built static web assets to a separate directory outside node_modules
    cp -r node_modules/@filen/web/dist web-dist
    
    # Manually trigger the rclone binary fetching step
    node build/rclone/fetch.mjs
    
    # Run production build compilation steps manually
    npm run clear
    npm run verify:versions
    npm run lint
    npm run tsc
    
    # Call electron-builder directly targeting ONLY the raw directory structure
    npx electron-builder --linux --dir --publish never
}

package() {
    cd "${srcdir}/${_pkgname}"

    local target_dir="${pkgdir}/usr/lib/${_pkgname}"
    install -d "${target_dir}"
    install -d "${pkgdir}/usr/bin"
    install -d "${pkgdir}/usr/share/applications"
    install -d "${pkgdir}/usr/share/pixmaps"

    # Copy files out of electron-builder's unpacked production directory resources (only copy app.asar and unpacked dependencies)
    cp -a prod/linux-unpacked/resources/* "${target_dir}/"

    # Create launcher script wrapper using system electron
    cat <<EOF > "${pkgdir}/usr/bin/${_pkgname}"
#!/bin/sh
exec electron /usr/lib/${_pkgname}/app.asar "\$@"
EOF
    chmod +x "${pkgdir}/usr/bin/${_pkgname}"

    # Install launcher art if present
    if [ -f "build/icons/icon.png" ]; then
        install -m644 "build/icons/icon.png" "${pkgdir}/usr/share/pixmaps/${_pkgname}.png"
    elif [ -f "assets/icon.png" ]; then
        install -m644 "assets/icon.png" "${pkgdir}/usr/share/pixmaps/${_pkgname}.png"
    fi

    # Native System Launcher Desktop Spec
    cat <<EOF > "${pkgdir}/usr/share/applications/${_pkgname}.desktop"
[Desktop Entry]
Name=Filen
Exec=/usr/bin/${_pkgname} %U
Terminal=false
Type=Application
Icon=${_pkgname}
StartupWMClass=Filen
Comment=Secure cloud storage client
Categories=Network;FileTransfer;
EOF
}
