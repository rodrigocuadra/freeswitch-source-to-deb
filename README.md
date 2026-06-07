# 📦 Step-by-Step Guide: Compiling FreeSWITCH on Debian 13 (with .deb packages)

This document summarizes the corrected workflow we followed to build FreeSWITCH on Debian 13 (trixie/sid) and generate .deb packages, taking into account the issues encountered (missing dependencies, spandsp, sofia-sip, etc.).

> [!IMPORTANT]
> **Multi-Architecture Support (AMD64 / ARM64):**
> This document describes the native compilation of FreeSWITCH on Debian 13.
> If compiled on an Intel/AMD host (e.g. Proxmox VM), the resulting `.deb` packages will have the `_amd64.deb` suffix.
> If compiled on an ARM64 host (e.g. Mac VM), the installation and packaging commands will generate files with the `_arm64.deb` suffix. Make sure to use the correct file names according to your compilation VM's architecture.

---

## 🛠️ Phase 1: Environment & Base Prerequisites

### 1.1 Prepare the Environment and Install Build Tools

Install the general build tools and packaging utilities required for building Debian packages:

```bash
apt-get update
apt-get install -y \
  sudo git build-essential devscripts dpkg-dev equivs fakeroot \
  autoconf automake libtool libtool-bin cmake pkg-config \
  wget curl unzip python-is-python3 python3-all-dev \
  erlang-dev libtpl-dev libgdbm-dev libdb-dev \
  python3-setuptools docbook-xsl xsltproc \
  libglib2.0-dev graphviz dh-make cmake
```

---

### 1.2 Install FreeSWITCH Base Dependencies

Install the core system libraries:

```bash
apt-get install -y \
  uuid-dev zlib1g-dev libjpeg-dev libsqlite3-dev libcurl4-openssl-dev \
  libpcre2-dev libspeex-dev libspeexdsp-dev libldns-dev libedit-dev \
  libtiff5-dev yasm libopus-dev libsndfile1-dev libavformat-dev \
  libswscale-dev liblua5.2-dev liblua5.2-0 libpq-dev unixodbc-dev \
  libxml2-dev libpq5 sngrep libswresample-dev bison doxygen \
  python3-dev python-is-python3 dh-python libjansson-dev \
  libopencv-dev libhiredis-dev libmemcached-dev libsphinxbase-dev \
  libpocketsphinx-dev libopencore-amrnb-dev libmariadb-dev libperl-dev \
  libgdbm-compat-dev librabbitmq-dev libsnmp-dev libmagickcore-dev \
  libopusfile-dev libmp3lame-dev libshout3-dev libvlc-dev default-jdk mono-mcs \
  libasound2-dev libcodec2-dev python-dev-is-python3
```

> ⚠️ **Note**: Debian 13 no longer includes libpcre3-dev or ntpdate; they are replaced by libpcre2-dev and ntpsec-ntpdate.

---

## 📦 Phase 2: Compile Custom Telephony Libraries (Dependencies)

Some critical dependencies for FreeSWITCH are no longer available in the Debian 13 repository or require custom configurations. We compile them from source and build their corresponding `.deb` packages.

### 2.1 spandsp 3.x

```bash
cd /usr/src
git clone https://github.com/freeswitch/spandsp.git
cd spandsp
./bootstrap.sh
./configure

fakeroot debian/rules clean
dpkg-buildpackage -us -uc -b

# Move packages to /usr/src/freeswitch/deb
mkdir -p /usr/src/freeswitch/deb
mv /usr/src/libspandsp3*.deb /usr/src/freeswitch/deb/

# Install the newly created package so that it is available as a dependency for FreeSWITCH
cd /usr/src/freeswitch/deb
dpkg -i libspandsp3_*.deb libspandsp3-dev_*.deb
apt-get install -f -y
```

### 2.2 libks2

**Step 1: Clone the Repository and Prepare the Directory**

```bash
cd /usr/src
git clone https://github.com/signalwire/libks.git
cd libks
mkdir -p debian
```

**Step 2: Create Debian Packaging Files**

Create the following files inside the `debian/` directory:

*   `debian/control`:
    ```bash
    cat <<'EOF' > debian/control
    Source: libks
    Section: libs
    Priority: optional
    Maintainer: Rodrigo Cuadra <rcuadra@aplitel.com>
    Build-Depends: debhelper-compat (= 13), cmake, libpcre2-dev, uuid-dev
    Standards-Version: 4.6.2
    Homepage: https://github.com/signalwire/libks

    Package: libks2
    Architecture: any
    Depends: ${shlibs:Depends}, ${misc:Depends}
    Description: SignalWire libks library
     A library for communication protocols developed by SignalWire.

    Package: libks2-dev
    Architecture: any
    Depends: libks2 (= ${binary:Version}), ${misc:Depends}
    Description: Development files for libks
     Development headers and libraries for libks.
    EOF
    ```

*   `debian/rules` (make sure to set it executable):
    ```bash
    cat <<'EOF' > debian/rules
    #!/usr/bin/make -f
    %:
    	dh $@

    override_dh_auto_configure:
    	cmake -DCMAKE_INSTALL_PREFIX=/usr -DCMAKE_BUILD_TYPE=Release .

    override_dh_auto_install:
    	dh_auto_install --destdir=debian/tmp

    override_dh_auto_clean:
    	dh_clean

    override_dh_installdocs:
    	dh_installdocs
    	dh_installdocs -plibks2-dev --link-doc=libks2
    EOF
    chmod +x debian/rules
    ```

*   `debian/changelog`:
    ```bash
    cat <<'EOF' > debian/changelog
    libks (2.0-7) unstable; urgency=medium

      * Initial packaging with proper multiarch support.

     -- Rodrigo Cuadra <rcuadra@aplitel.com>  Mon, 08 Sep 2025 20:30:00 +0000
    EOF
    ```

*   `debian/copyright`:
    ```bash
    cat <<'EOF' > debian/copyright
    Format: https://www.debian.org/doc/packaging-manuals/copyright-format/1.0/
    Files: *
    Copyright: 2025 SignalWire, Inc.
    License: MPL-1.1 or GPL-2+

    License: MPL-1.1 or GPL-2+
     This program is dual licensed under MPL 1.1 or GPL 2.0.
     On Debian systems, the complete text of the GNU General Public License
     version 2 can be found in /usr/share/common-licenses/GPL-2.
    EOF
    ```

*   `debian/not-installed`:
    ```bash
    cat <<'EOF' > debian/not-installed
    usr/share/doc/libks2/changelog.Debian.gz
    usr/share/doc/libks2/copyright
    EOF
    ```

*   `debian/libks2.install`:
    ```bash
    cat <<'EOF' > debian/libks2.install
    usr/lib/libks2.so*
    EOF
    ```

*   `debian/libks2-dev.install`:
    ```bash
    cat <<'EOF' > debian/libks2-dev.install
    usr/include/libks2/*
    usr/lib/pkgconfig/libks2.pc
    EOF
    ```

**Step 3: Compile and Install the Debian Package**

```bash
# Clean potential build residuals
rm -rf obj-x86_64-linux-gnu CMakeCache.txt CMakeFiles
fakeroot debian/rules clean

# Compile packages (-us -uc disables GPG signing, nocheck skips timing unit tests)
DEB_BUILD_OPTIONS="nocheck" dpkg-buildpackage -us -uc -b

# Move generated packages to the FreeSWITCH deb cache
mkdir -p /usr/src/freeswitch/deb
mv /usr/src/libks2*.deb /usr/src/freeswitch/deb/

# Install the packages so they can be resolved as build dependencies
cd /usr/src/freeswitch/deb
dpkg -i libks2_*.deb libks2-dev_*.deb
apt-get install -f -y
```

### 2.3 sofia-sip

```bash
cd /usr/src
git clone https://github.com/freeswitch/sofia-sip.git
cd sofia-sip
./bootstrap.sh
./configure

fakeroot debian/rules clean
dpkg-buildpackage -us -uc -b

# Move packages to /usr/src/freeswitch/deb
mkdir -p /usr/src/freeswitch/deb
mv /usr/src/libsofia-sip*.deb /usr/src/freeswitch/deb/
mv /usr/src/sofia-sip-bin*.deb /usr/src/freeswitch/deb/

# Install the newly created package so that it is available as a dependency for FreeSWITCH
cd /usr/src/freeswitch/deb
dpkg -i libsofia-sip-ua0_*.deb libsofia-sip-ua-dev_*.deb sofia-sip-bin_*.deb
apt-get install -f -y
```

### 2.4 libbroadvoice

```bash
cd /usr/src
git clone https://github.com/freeswitch/libbroadvoice.git
cd libbroadvoice
./autogen.sh
./configure

fakeroot debian/rules clean
dpkg-buildpackage -us -uc -b

# Move packages to /usr/src/freeswitch/deb
mkdir -p /usr/src/freeswitch/deb
mv /usr/src/libbroadvoice*.deb /usr/src/freeswitch/deb/

# Install the newly created package so that it is available as a dependency for FreeSWITCH
cd /usr/src/freeswitch/deb
dpkg -i libbroadvoice1_*.deb libbroadvoice-dev_*.deb
apt-get install -f -y
```

### 2.5 signalwire-c

**Step 1: Clone the Repository and Prepare the Directory**

```bash
cd /usr/src
git clone https://github.com/signalwire/signalwire-c.git
cd signalwire-c
mkdir -p debian
```

**Step 2: Create Debian Packaging Files**

Create the following files inside the `debian/` directory:

*   `debian/control`:
    ```bash
    cat <<'EOF' > debian/control
    Source: signalwire-c
    Section: libs
    Priority: optional
    Maintainer: Rodrigo Cuadra <rcuadra@aplitel.com>
    Build-Depends: debhelper-compat (= 13), cmake, libpcre2-dev, uuid-dev, libpcre2-dev, libssl-dev, libjansson-dev
    Standards-Version: 4.6.2
    Homepage: https://github.com/signalwire/signalwire-c

    Package: signalwire-client-c2
    Architecture: any
    Depends: ${shlibs:Depends}, ${misc:Depends}, libks2
    Description: SignalWire C client library (version 2)
     A C library for SignalWire communication protocols.

    Package: signalwire-client-c2-dev
    Architecture: any
    Depends: signalwire-client-c2 (= ${binary:Version}), ${misc:Depends}, libks2-dev
    Description: Development files for SignalWire C client library (version 2)
     Development headers and libraries for signalwire-client-c2.
    EOF
    ```

*   `debian/rules` (make sure to set it executable):
    ```bash
    cat <<'EOF' > debian/rules
    #!/usr/bin/make -f
    %:
    	dh $@

    override_dh_auto_configure:
    	cmake -DCMAKE_INSTALL_PREFIX=/usr \
    	      -DCMAKE_BUILD_TYPE=Release \
    	      -DBUILD_SHARED_LIBS=ON \
    	      -DINSTALL_PKGCONFIG_DIR=/usr/lib/$(DEB_HOST_MULTIARCH)/pkgconfig .

    override_dh_auto_install:
    	dh_auto_install --destdir=debian/tmp

    override_dh_auto_test:
    	: # Skip tests temporarily

    override_dh_auto_clean:
    	dh_clean

    override_dh_installdocs:
    	dh_installdocs
    	dh_installdocs -psignalwire-client-c2-dev --link-doc=signalwire-client-c2
    EOF
    chmod +x debian/rules
    ```

*   `debian/changelog`:
    ```bash
    cat <<'EOF' > debian/changelog
    signalwire-c (1.0-13) unstable; urgency=medium

      * Initial packaging for SignalWire C library.

     -- Rodrigo Cuadra <rcuadra@aplitel.com>  Mon, 08 Sep 2025 22:00:00 +0000
    EOF
    ```

*   `debian/copyright`:
    ```bash
    cat <<'EOF' > debian/copyright
    Format: https://www.debian.org/doc/packaging-manuals/copyright-format/1.0/
    Files: *
    Copyright: 2025 SignalWire, Inc.
    License: MPL-1.1 or GPL-2+

    License: MPL-1.1 or GPL-2+
     This program is dual licensed under MPL 1.1 or GPL 2.0.
     On Debian systems, the complete text of the GNU General Public License
     version 2 can be found in /usr/share/common-licenses/GPL-2.
    EOF
    ```

*   `debian/not-installed`:
    ```bash
    cat <<'EOF' > debian/not-installed
    usr/share/doc/signalwire-client-c2/changelog.Debian.gz
    usr/share/doc/signalwire-client-c2/copyright
    EOF
    ```

*   `debian/signalwire-client-c2.install`:
    ```bash
    cat <<'EOF' > debian/signalwire-client-c2.install
    usr/lib/libsignalwire_client2.so*
    EOF
    ```

*   `debian/signalwire-client-c2-dev.install`:
    ```bash
    cat <<'EOF' > debian/signalwire-client-c2-dev.install
    usr/include/signalwire-client-c2/*
    usr/lib/pkgconfig/signalwire_client2.pc
    EOF
    ```

**Step 3: Compile and Install the Debian Package**

```bash
# Clean potential build residuals
rm -rf obj-x86_64-linux-gnu CMakeCache.txt CMakeFiles
fakeroot debian/rules clean

# Compile packages (-us -uc disables GPG signing)
dpkg-buildpackage -us -uc -b

# Move generated packages to the FreeSWITCH deb cache
mkdir -p /usr/src/freeswitch/deb
mv /usr/src/signalwire-client-c2*.deb /usr/src/freeswitch/deb/

# Install the packages so they can be resolved as build dependencies
cd /usr/src/freeswitch/deb
dpkg -i signalwire-client-c2_*.deb signalwire-client-c2-dev_*.deb
apt-get install -f -y
```

---

## ⚙️ Phase 3: Compiling & Packaging FreeSWITCH

### 3.1 Clone FreeSWITCH Source Code

# Clone a specific stable release tag (e.g. v1.11.1) to ensure a stable build
```bash
cd /usr/src
git clone -b v1.11.1 --depth 1 https://github.com/signalwire/freeswitch.git /usr/src/freeswitch/src
cd /usr/src/freeswitch/src

# Generate configure
./bootstrap.sh -j

# Synchronize file modification timestamps to prevent Autotools clock-skew rebuild loops
find . -type f -exec touch {} +

./configure \
  --enable-core-odbc-support \
  --enable-core-pgsql-support \
  --enable-pcre2 \
  --prefix=/usr \
  --sysconfdir=/etc \
  --localstatedir=/var \
  --disable-maintainer-mode \
  --disable-dependency-tracking
```

> This creates the files debian/control, debian/rules, debian/modules_.conf, etc.

---

### 3.2 Build FreeSWITCH .deb Packages

**Step 1: Exclude Incompatible Modules & Bootstrap Packaging**

Bootstrap the Debian packaging structure for Trixie and exclude modules that are incompatible with Debian 13 (such as JavaScript/V8, Flite, and Silk):

```bash
cd /usr/src/freeswitch/src/debian

# Add incompatible modules to the bootstrap exclude list
sed -i '/xml_int\/mod_xml_ldap/a\  languages/mod_v8\n  languages/mod_basic\n  asr_tts/mod_flite\n  codecs/mod_ilbc\n  codecs/mod_silk\n  languages/mod_managed' bootstrap.sh

# Generate the debian/ structure for Debian 13 (trixie)
./bootstrap.sh -c trixie
```

**Step 2: Apply Debian Configuration Workarounds**

Edit the generated `rules` and `control` files to bypass recursion loops and clean up dependencies that are incompatible with Debian 13:

```bash
cd /usr/src/freeswitch/src/debian

# Disable dh_auto_clean in rules to bypass the infinite clean recursive make loop
echo "" >> rules
echo "override_dh_auto_clean:" >> rules
printf "\t:\n" >> rules

# Remove dependencies not built due to incompatibility with Debian 13
sed -i '/freeswitch-meta-codecs (= ${binary:Version}),/d' control
sed -i '/freeswitch-music,/d' control
sed -i '/freeswitch-sounds,/d' control
sed -i '/freeswitch-mod-flite (= ${binary:Version}),/d' control
```

**Step 3: Update changelog Version**

Create/update the changelog metadata to reflect FreeSWITCH `1.11.1`:

```bash
cat > changelog <<'EOF'
freeswitch (1.11.1-1) unstable; urgency=medium

  * Repackage FreeSWITCH 1.11.1 for Debian 13.
  * Removed obsolete dependencies (flite, silk, ilbc, etc).
  * Adjusted modules to reflect available codecs and system libraries.
  * Local packaging for testing repository.

 -- Rodrigo Cuadra <rcuadra@aplitel.com>  Tue, 10 Sep 2025 20:30:00 +0000
EOF
```

**Step 4: Build Debian Packages**

Sync file modification timestamps to prevent Autotools clock-skew loops, clean, and compile the packages. We force sequential compilation using `DEB_BUILD_OPTIONS="parallel=1"` to prevent write race conditions:

```bash
cd /usr/src/freeswitch/src

# Synchronize timestamps to prevent infinite make loops during dpkg-buildpackage
find . -type f -exec touch {} +

# Clean and remove unnecessary debug packaging targets
fakeroot debian/rules clean
rm -rf debian/python-esl-dbg
rm -rf debian/libfreeswitch1-dbg

# Build the packages sequentially to bypass modules.inc write race conditions
DEB_BUILD_OPTIONS="parallel=1" dpkg-buildpackage -us -uc -b -d
```

**Step 5: Isolate Generated Packages**

Move all generated packages into `/usr/src/freeswitch/deb/`:

```bash
mkdir -p /usr/src/freeswitch/deb
mv /usr/src/freeswitch/*.deb /usr/src/freeswitch/deb/ 2>/dev/null || true
mv /usr/src/freeswitch/libfreeswitch*.deb /usr/src/freeswitch/deb/ 2>/dev/null || true
mv /usr/src/freeswitch/libesl*.deb /usr/src/freeswitch/deb/ 2>/dev/null || true
mv /usr/src/freeswitch/python-esl*.deb /usr/src/freeswitch/deb/ 2>/dev/null || true
```

This generates all .deb packages and moves them to `/usr/src/freeswitch/deb/`. For the complete catalog of generated packages, see the [Complete Package Output Reference](#64-complete-package-output-reference) in Phase 6.

---

## 🚀 Phase 4: Local Installation & Systemd Setup

To install FreeSWITCH on a server directly from the compiled `.deb` packages without using a public repository server, follow these steps:

### 4.1 Install Custom Built Dependencies

Install the custom-built dependency packages first:

```bash
cd /usr/src/freeswitch/deb

# Install dependency packages (spandsp, libks2, sofia-sip, broadvoice, signalwire-c)
apt-get install -y \
  ./libspandsp3_*.deb \
  ./libspandsp3-dev_*.deb \
  ./libks2_*.deb \
  ./libks2-dev_*.deb \
  ./libsofia-sip-ua0_*.deb \
  ./libsofia-sip-ua-dev_*.deb \
  ./sofia-sip-bin_*.deb \
  ./libbroadvoice1_*.deb \
  ./libbroadvoice-dev_*.deb \
  ./signalwire-client-c2_*.deb \
  ./signalwire-client-c2-dev_*.deb

# Fix any missing system dependencies
apt-get install -f -y
```

### 4.2 Install FreeSWITCH Packages

Install the main FreeSWITCH package along with all generated module packages:

```bash
# Install core and all compiled packages
apt-get install -y ./*.deb

# Fix any missing dependencies from Debian repositories
apt-get install -f -y
```

### 4.3 Verify the Local Installation

Check that FreeSWITCH compiles and runs correctly:

```bash
# Check version information
freeswitch -version
fs_cli -V
```

### 4.4 Enable and Configure Systemd Service

Adjust the FreeSWITCH systemd service configuration to ensure it starts automatically:

```bash
# Start FreeSWITCH in the background to initialize
sudo -u freeswitch /usr/bin/freeswitch -ncwait -nonat -c

# Reload systemd and start the service
systemctl daemon-reload
systemctl enable freeswitch
systemctl start freeswitch
```

---

## 🌐 Phase 5: Central Distribution (Optional)

If you want to distribute these compiled `.deb` packages to multiple target servers via a secure, central APT repository server using Nginx, Let's Encrypt SSL, and custom index builders, see the dedicated [Debian APT Repository Setup Guide](file:///Users/rodrigocuadra/Documents/Ring2All/docs/distribution/apt_repository_guide.md).

---

## 📚 Phase 6: Reference & Troubleshooting

### 6.1 Troubleshooting & Known Compilation Workarounds

When building FreeSWITCH on virtualized hosts (e.g., Proxmox, VirtualBox, or Docker) or modern Debian 13 environments, you might encounter infinite recursion loops during `make` or `make clean`, which can exhaust system PIDs and collapse the VM. Here is why they happen and how they are resolved:

#### 1. GNU Make Infinite Recursion Loops (Clock Skew)
*   **The Issue:** Autotools-based systems compare file timestamps to determine if Makefiles or scripts (like `configure` and `config.status`) need to be rebuilt. On VMs or containers where NTP clock updates occur after files are created, or on mounted filesystems, a source template (like `Makefile.am` or `configure.ac`) can get a timestamp in the "future" relative to the system's clock. This causes GNU Make to repeatedly try to regenerate the Makefile, restart itself, and repeat the check, creating an infinite recursion loop (`make[484]`, `make[485]`, etc.).
*   **The Fix:** 
    1. Align all timestamps to the current system time using `find . -type f -exec touch {} +` before starting configuration or clean runs.
    2. Run `./configure` with the `--disable-maintainer-mode` and `--disable-dependency-tracking` flags. This disables the automatic autotools check and regeneration rules entirely, letting `make` compile directly.

#### 2. Parallel Build Race Conditions (`modules.inc`)
*   **The Issue:** FreeSWITCH dynamically generates `src/mod/modules.inc` from `modules.conf` during the build process. Since `modules.inc` is included in the Makefiles, updating it mid-compile under high concurrency (e.g., `make -j8`) causes GNU Make to stop, reload all Makefiles, and restart. Under parallel builds, this triggers an infinite restart/compilation loop in `src/mod`.
*   **The Fix:** Force sequential compilation when building Debian packages by defining `DEB_BUILD_OPTIONS="parallel=1"` before executing `dpkg-buildpackage`. This is the same workaround used in the official SignalWire release Dockerfiles.

#### 3. Infinite Loop on `make clean` (Debian Package Clean Phase)
*   **The Issue:** Running `fakeroot debian/rules clean` or `make clean` triggers GNU Make to inspect Makefile dependencies. If any dependency is considered out of date, it triggers the regeneration loop before the cleaning process can even run.
*   **The Fix:** Since we compile in a freshly cloned directory, running `make clean` is redundant. We bypass it by overriding `dh_auto_clean` as a no-op inside `debian/rules` during Section 5:
    ```bash
    echo "override_dh_auto_clean:" >> rules
    printf "\t:\n" >> rules
    ```

---

### 6.2 Workflow Summary

1. Prepare the environment and install build dependencies.
2. Compile and package **external dependencies** (`spandsp`, `libks2`, `sofia-sip`, `libbroadvoice`, `signalwire-c`) as `.deb`.
3. Configure, compile, and package **FreeSWITCH + modules** as `.deb` using sequential compilation workaround.
4. Install packages locally using `apt-get install ./*.deb`.
5. Test using: `fs_cli -x "status"` and `fs_cli -x "show codecs"`.

---

### 6.3 Expected Result
- ✔ FreeSWITCH runs smoothly on Debian 13.
- ✔ Custom-built `.deb` packages successfully compile and install.
- ✔ Modern codecs like Opus, G.729, CODEC2 are available.
- ✔ Service runs automatically with correct permissions.

---

### 6.4 Complete Package Output Reference

The compiling and packaging process generates the following 191 packages:

#### Dependencies (12 packages)

| Package | Version | Description |
|---------|---------|-------------|
| `libspandsp3` | 3.0.0-42 | SpanDSP telephony library |
| `libspandsp3-dev` | 3.0.0-42 | SpanDSP development headers |
| `libspandsp3-doc` | 3.0.0-42 | SpanDSP documentation |
| `libks2` | 2.0-7 | SignalWire libks library |
| `libks2-dev` | 2.0-7 | libks development headers |
| `libsofia-sip-ua0` | 1.13.17-0 | Sofia-SIP library |
| `libsofia-sip-ua-dev` | 1.13.17-0 | Sofia-SIP development headers |
| `libsofia-sip-ua-glib3` | 1.13.17-0 | Sofia-SIP GLib bindings |
| `libsofia-sip-ua-glib-dev` | 1.13.17-0 | Sofia-SIP GLib dev headers |
| `sofia-sip-bin` | 1.13.17-0 | Sofia-SIP utilities |
| `libbroadvoice1` | 0.1.0-999~u1 | BroadVoice codec library |
| `libbroadvoice-dev` | 0.1.0-999~u1 | BroadVoice development headers |
| `signalwire-client-c2` | 1.0-13 | SignalWire C client library |
| `signalwire-client-c2-dev` | 1.0-13 | SignalWire dev headers |

#### FreeSWITCH Core (6 packages)

| Package | Description |
|---------|-------------|
| `freeswitch` | Main FreeSWITCH package |
| `freeswitch-dbg` | Debug symbols |
| `freeswitch-doc` | Documentation |
| `libfreeswitch1` | Core library |
| `libfreeswitch1-dbg` | Library debug symbols |
| `libfreeswitch-dev` | Development headers |

#### Meta Packages (12 packages)

| Package | Description |
|---------|-------------|
| `freeswitch-all` | All modules |
| `freeswitch-meta-all` | Meta: all modules |
| `freeswitch-meta-bare` | Meta: minimal install |
| `freeswitch-meta-codecs` | Meta: all codecs |
| `freeswitch-meta-conf` | Meta: all configurations |
| `freeswitch-meta-default` | Meta: default modules |
| `freeswitch-meta-lang` | Meta: all languages |
| `freeswitch-meta-mod-say` | Meta: all say modules |
| `freeswitch-meta-sorbet` | Meta: sorbet config |
| `freeswitch-meta-vanilla` | Meta: vanilla config |

#### Configuration Packages (7 packages)

| Package | Description |
|---------|-------------|
| `freeswitch-conf-curl` | XML-CURL config |
| `freeswitch-conf-insideout` | InsideOut config |
| `freeswitch-conf-minimal` | Minimal config |
| `freeswitch-conf-sbc` | SBC config |
| `freeswitch-conf-softphone` | Softphone config |
| `freeswitch-conf-testing` | Testing config |
| `freeswitch-conf-vanilla` | Vanilla config |

#### Language Packs (10 packages)

| Package | Language |
|---------|----------|
| `freeswitch-lang` | Base language pack |
| `freeswitch-lang-de` | German |
| `freeswitch-lang-en` | English |
| `freeswitch-lang-es` | Spanish |
| `freeswitch-lang-fr` | French |
| `freeswitch-lang-he` | Hebrew |
| `freeswitch-lang-pt` | Portuguese |
| `freeswitch-lang-ru` | Russian |
| `freeswitch-lang-sv` | Swedish |

#### Codec Modules (14 packages, + dbg)

| Package | Codec |
|---------|-------|
| `freeswitch-mod-amr` | AMR |
| `freeswitch-mod-amrwb` | AMR-WB |
| `freeswitch-mod-b64` | Base64 |
| `freeswitch-mod-bv` | BroadVoice |
| `freeswitch-mod-codec2` | Codec2 |
| `freeswitch-mod-g723-1` | G.723.1 |
| `freeswitch-mod-g729` | G.729 |
| `freeswitch-mod-opus` | Opus |
| `freeswitch-mod-opusfile` | Opus file |
| `freeswitch-mod-spandsp` | SpanDSP (G.711, T.38) |

#### Application Modules (30+ packages, + dbg)

| Package | Function |
|---------|----------|
| `freeswitch-mod-callcenter` | Call Center |
| `freeswitch-mod-conference` | Conferencing |
| `freeswitch-mod-db` | Database |
| `freeswitch-mod-directory` | Directory |
| `freeswitch-mod-dptools` | Dialplan Tools |
| `freeswitch-mod-fifo` | FIFO Queues |
| `freeswitch-mod-hash` | Hash Storage |
| `freeswitch-mod-httapi` | HTTP API |
| `freeswitch-mod-lcr` | Least Cost Routing |
| `freeswitch-mod-nibblebill` | Billing |
| `freeswitch-mod-valet-parking` | Call Parking |
| `freeswitch-mod-voicemail` | Voicemail |
| `freeswitch-mod-voicemail-ivr` | Voicemail IVR |

#### CDR Modules (6 packages, + dbg)

| Package | Backend |
|---------|---------|
| `freeswitch-mod-cdr-csv` | CSV files |
| `freeswitch-mod-cdr-mongodb` | MongoDB |
| `freeswitch-mod-cdr-pg-csv` | PostgreSQL CSV |
| `freeswitch-mod-cdr-sqlite` | SQLite |
| `freeswitch-mod-json-cdr` | JSON CDR |
| `freeswitch-mod-odbc-cdr` | ODBC CDR |
| `freeswitch-mod-xml-cdr` | XML CDR |

#### Database Modules (4 packages, + dbg)

| Package | Database |
|---------|----------|
| `freeswitch-mod-mariadb` | MariaDB |
| `freeswitch-mod-pgsql` | PostgreSQL ✅ |
| `freeswitch-mod-hiredis` | Redis |
| `freeswitch-mod-memcache` | Memcached |

#### Endpoint Modules (6 packages, + dbg)

| Package | Protocol |
|---------|----------|
| `freeswitch-mod-sofia` | SIP (Sofia-SIP) ✅ |
| `freeswitch-mod-verto` | WebRTC ✅ |
| `freeswitch-mod-rtmp` | RTMP |
| `freeswitch-mod-skinny` | Skinny/SCCP |
| `freeswitch-mod-loopback` | Loopback |
| `freeswitch-mod-alsa` | ALSA audio |

#### Say Modules (18 packages, + dbg)

| Package | Language |
|---------|----------|
| `freeswitch-mod-say-de` | German |
| `freeswitch-mod-say-en` | English |
| `freeswitch-mod-say-es` | Spanish |
| `freeswitch-mod-say-es-ar` | Spanish (Argentina) |
| `freeswitch-mod-say-fa` | Persian |
| `freeswitch-mod-say-fr` | French |
| `freeswitch-mod-say-he` | Hebrew |
| `freeswitch-mod-say-hr` | Croatian |
| `freeswitch-mod-say-hu` | Hungarian |
| `freeswitch-mod-say-it` | Italian |
| `freeswitch-mod-say-ja` | Japanese |
| `freeswitch-mod-say-nl` | Dutch |
| `freeswitch-mod-say-pl` | Polish |
| `freeswitch-mod-say-pt` | Portuguese |
| `freeswitch-mod-say-ru` | Russian |
| `freeswitch-mod-say-sv` | Swedish |
| `freeswitch-mod-say-th` | Thai |
| `freeswitch-mod-say-zh` | Chinese |

#### Event & Logging Modules (8 packages, + dbg)

| Package | Function |
|---------|----------|
| `freeswitch-mod-event-multicast` | Multicast events |
| `freeswitch-mod-event-socket` | Event Socket ✅ |
| `freeswitch-mod-event-test` | Event testing |
| `freeswitch-mod-logfile` | File logging |
| `freeswitch-mod-syslog` | Syslog |
| `freeswitch-mod-graylog2` | Graylog |
| `freeswitch-mod-console` | Console |
| `freeswitch-mod-fail2ban` | Fail2ban |

#### Dialplan Modules (4 packages, + dbg)

| Package | Type |
|---------|------|
| `freeswitch-mod-dialplan-xml` | XML Dialplan ✅ |
| `freeswitch-mod-dialplan-asterisk` | Asterisk-style |
| `freeswitch-mod-dialplan-directory` | Directory |

#### XML Modules (4 packages, + dbg)

| Package | Function |
|---------|----------|
| `freeswitch-mod-xml-curl` | XML via HTTP |
| `freeswitch-mod-xml-rpc` | XML-RPC |
| `freeswitch-mod-xml-scgi` | SCGI |

#### Language Bindings (3 packages, + dbg)

| Package | Language |
|---------|----------|
| `freeswitch-mod-lua` | Lua ✅ |
| `freeswitch-mod-perl` | Perl |
| `freeswitch-mod-python3` | Python 3 |
| `freeswitch-mod-java` | Java |
| `freeswitch-mod-erlang-event` | Erlang |
| `libesl-perl` | ESL Perl bindings |
| `python-esl` | ESL Python bindings |

#### Other Modules

| Package | Function |
|---------|----------|
| `freeswitch-mod-av` | Audio/Video |
| `freeswitch-mod-avmd` | AMD detection |
| `freeswitch-mod-blacklist` | Blacklisting |
| `freeswitch-mod-cidlookup` | Caller ID lookup |
| `freeswitch-mod-commands` | CLI commands |
| `freeswitch-mod-curl` | HTTP client |
| `freeswitch-mod-cv` | OpenCV video |
| `freeswitch-mod-distributor` | Load balancing |
| `freeswitch-mod-easyroute` | Easy routing |
| `freeswitch-mod-enum` | ENUM lookup |
| `freeswitch-mod-esf` | ESF |
| `freeswitch-mod-esl` | ESL server |
| `freeswitch-mod-expr` | Expressions |
| `freeswitch-mod-format-cdr` | CDR formatting |
| `freeswitch-mod-fsk` | FSK signaling |
| `freeswitch-mod-fsv` | FS video |
| `freeswitch-mod-http-cache` | HTTP cache |
| `freeswitch-mod-imagick` | ImageMagick |
| `freeswitch-mod-local-stream` | Local streams |
| `freeswitch-mod-native-file` | Native files |
| `freeswitch-mod-png` | PNG images |
| `freeswitch-mod-pocketsphinx` | Speech recognition |
| `freeswitch-mod-posix-timer` | POSIX timer |
| `freeswitch-mod-prefix` | Prefix routing |
| `freeswitch-mod-random` | Random |
| `freeswitch-mod-redis` | Redis |
| `freeswitch-mod-rtc` | RTC |
| `freeswitch-mod-shell-stream` | Shell streams |
| `freeswitch-mod-shout` | Shoutcast/MP3 |
| `freeswitch-mod-signalwire` | SignalWire |
| `freeswitch-mod-sms` | SMS |
| `freeswitch-mod-snapshot` | Snapshots |
| `freeswitch-mod-sndfile` | Sound files |
| `freeswitch-mod-snmp` | SNMP |
| `freeswitch-mod-spy` | Call spy |
| `freeswitch-mod-test` | Testing |
| `freeswitch-mod-timerfd` | Timer FD |
| `freeswitch-mod-tone-stream` | Tone generation |
| `freeswitch-mod-translate` | Translation |
| `freeswitch-mod-tts-commandline` | TTS command |
| `freeswitch-mod-video-filter` | Video filters |
| `freeswitch-mod-vlc` | VLC playback |
| `freeswitch-mod-vmd` | Voicemail detection |
| `freeswitch-mod-yuv` | YUV video |
| `freeswitch-systemd` | Systemd integration |
| `freeswitch-timezones` | Timezone data |

> ✅ = Essential for Ring2All/Softswitch platform
