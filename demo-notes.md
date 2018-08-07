* Projects involved in this demonstration
  * [Slide overview](https://github.com/nth10sd/2015-Fuzzing-and-Mozilla)
  * [mozilla-central](https://hg.mozilla.org/mozilla-central/)
  * [funfuzz](https://github.com/MozillaSecurity/funfuzz) (Main harness)
    * [jsfunfuzz](https://github.com/MozillaSecurity/funfuzz/tree/master/src/funfuzz/js/jsfunfuzz)
      * ([original blogpost from 2007](https://www.squarefree.com/2007/08/02/introducing-jsfunfuzz/))
    * compare_jit
      * Compares the output of the js shell using different runtime flags
    * randorderfuzz (Random order fuzzer)
      * Combines fuzzer output with tests from mozilla-central (specifically jit-tests)
    * autobisectjs
      * Bisects patches back in time to find the regressing changeset
  * [FuzzManager](https://github.com/MozillaSecurity/FuzzManager) (Reporting tool)
  * Most security bugs found by the public are eligible for bounties via the [Mozilla Bug Bounty program](https://www.mozilla.org/en-US/security/client-bug-bounty/) for US$2,000 - $5,000 depending on the severity of the bug, as judged by the Mozilla security team

===

* How to create the VM:
  * Install [VirtualBox](https://www.virtualbox.org/) - version used is 5.2.16
    * Install "VirtualBox 5.2.16 Oracle VM VirtualBox Extension Pack" as well
  * Download and install [Ubuntu Linux 18.04 Desktop](https://www.ubuntu.com/desktop) into a VM
    * VM settings: minimal installation, download updates during install, install 3rd party packages
    * VM specs: 4 processors, 64GB disk space, ~6GB RAM (for reference)
    * Make sure VirtualBox Guest Additions is installed (should be present, if you installed via default dialog)
  * Update it via `apt-get update` and `apt-get upgrade`

* Custom settings:
  * To prepare for this demo, large files/projects were obtained in advance
    * apt-get packages were downloaded ahead of time
  * `apt-get install terminator` (My favourite terminal client in Ubuntu)
  * `mkdir ~/trees`
  * Clone [mozilla-central](https://hg.mozilla.org/mozilla-central/) (via apt-get Mercurial)
    * `cd ~/trees`
    * `hg clone https://hg.mozilla.org/mozilla-central/ mozilla-central`
    * Note that the harness might make assumptions for repository being in `~/trees`
    * ~~Then **Mercurial was removed** (via apt-get)~~
    * ~~Reason: We should install and update Mercurial using pip instead (will be shown during the demo)~~
  * [Rust](https://rustup.rs/) was installed (`bootstrap.py` requires this)
    * You might need to first `apt-get install curl`
    * Also run `rustup target add i686-unknown-linux-gnu` to compile 32-bit builds
    * `echo 'source /home/<your username>/.cargo/env' >> ~/.bashrc`
  * Testcase for [bug 1465693](https://bugzilla.mozilla.org/show_bug.cgi?id=1465693) downloaded in advance
    * Demo bug
  * Restart the VM

===

* ~~Start of demo (without slide overview)~~

* Run `bootstrap.py`
  * `wget https://hg.mozilla.org/mozilla-central/raw-file/default/python/mozboot/bin/bootstrap.py`
  * `python bootstrap.py`
  * Press `2` for Firefox for Desktop development
  * `bootstrap.py` will prompt to install prerequisites (mostly via apt-get)
  * Recommended to install Mercurial via pip
  * Press `1` to create `.mozbuild` directory
  * Run Mercurial configuration wizard
    * Accept all but reject `curses` interface, `evolve` and `firefoxtree` extensions for now
    * They are untested

* Install more prerequisites
  * `apt-get install git ccache gdb lib32z1 gcc-7-multilib g++-7-multilib apache2-utils`
    * Framework binaries: `git`, `ccache`, `gdb` (ensure it is present)
    * `lib32z1 gcc-7-multilib g++-7-multilib` (for compiling 32-bit binaries on 64-bit host)
    * `apache2-utils` (for FuzzManager)

* Clone [funfuzz](https://github.com/MozillaSecurity/funfuzz) off GitHub and install it via pip
  * `git clone https://github.com/MozillaSecurity/funfuzz ~/funfuzz`
  * `pushd ~/funfuzz && python3 -m pip install --user --upgrade -r requirements.txt && popd`
    * You need to be in the funfuzz directory for the requirements to be installed properly

* Enable some Mercurial extensions
  * `mq` - for testing a single patch
  * `progress` - for nice progress information
  * `purge` - ensures that repositories are clean but **destroys data**

* Define `LD_LIBRARY_PATH` in the `.bashrc` startup script
  * `echo 'export LD_LIBRARY_PATH=.' >> ~/.bashrc`

* Make sure corefiles are appended with the pid number as the extension to differentiate between core dumps
  * Run as root (e.g. `sudo su`)
  * `echo '1' > /proc/sys/kernel/core_uses_pid`
  * `echo 'kernel.core_uses_pid = 1' >> /etc/sysctl.conf`
  * **Exit** root mode by typing `exit`

* Restart

* Compile a 64-bit debug deterministic SpiderMonkey shell: (ideally without specifying `-r`, it is for demo purposes)
  * `python3 -u -m funfuzz.js.compile_shell -b "--enable-debug --enable-more-deterministic"` -r 3931f461c8e8

* FuzzManager
  * Ideally we should set it up properly using Apache/WSGI
    * However for demonstration purposes we shall use `runserver` and run FuzzManager in development mode locally
  * Clone FuzzManager
    * `git clone https://github.com/MozillaSecurity/FuzzManager ~/FuzzManager`

===

* Start of demo (with slide overview)

* Set up [FuzzManager](https://github.com/MozillaSecurity/FuzzManager) to run as a server
  * Install FuzzManager server requirements
    * `cd ~/FuzzManager`
    * `python3 -m pip install --user --upgrade -r server/requirements.txt`
    * `cd server/`
  * Configure FuzzManager
    * `python3 manage.py migrate`
    * `python3 manage.py createsuperuser`
      * Use `fuzzmanager` as a sample username
      * Create a sample password
    * `python3 manage.py get_auth_token fuzzmanager`
      * Note the auth token
    * `pushd ~ && htpasswd -cb .htpasswd fuzzmanager <token> && popd`
  * Create a FuzzManager configuration file
```
[Main]
serverhost = 127.0.0.1
serverport = 8000
serverproto = http
serverauthtoken = <token>
sigdir = /home/<username>/sigdir/
tool = jsfunfuzz
```
  * Make sure to use **https** when deploying a production FuzzManager server
  * Still in the `server/` directory, start up FuzzManager in developmental mode (only for this presentation purpose):
    * `python3 manage.py runserver`

* If bugs are hit, they should be submitted to your instance of FuzzManager.
  * If they need to be submitted manually, use:
    * `python3 -u -m funfuzz.js.js_interesting --submit DUMMYVARIABLE <path to js shell> <runtime flags> <testcase>`

===

* Areas that can be improved:
  * Audit tested language features in jsfunfuzz
  * Migrate completely to using Python 3, possibly 3.6
    * Currently on dual 2.7/3.5+ code
    * funfuzz has a long history, hailing from >10 years ago
      * Ported from bash scripts during the Python 2.4/2.5 days
  * Switch to Docker (Orion?)
  * Add more tests (currently only 45-50%)
    * We already have Travis/AppVeyor CI integration
    * The projects also integrate with `codecov.io`

* Caveats:
  * It is recommended to fuzz on a machine in as native a manner as possible
    * Known to work on Linux/macOS/Windows
    * Known to work on Amazon EC2
    * Used to work even on small ARM boards (e.g. ODROID)
  * VirtualBox is used for demonstration only
    * I hit a crash involving Python 2.7 and `blas_thread_shutdown_()` after fuzzing for a short time
    * Still have not figured out the cause yet

===

Note to self - Packages to download ahead of time:

```
sudo apt-get install -d python3-pip

sudo apt-get install -d autoconf2.13 build-essential nodejs python-dev python-pip python-setuptools unzip uuid zip

sudo apt-get install -d autoconf automake autopoint autotools-dev debhelper dh-autoreconf dh-strip-nondeterminism gconf-service gconf-service-backend gconf2  gconf2-common gir1.2-gconf-2.0 gir1.2-gtk-2.0 gir1.2-harfbuzz-0.0 icu-devtools libarchive-cpio-perl libatk-bridge2.0-dev libatk1.0-dev libatspi2.0-dev libcairo-script-interpreter2 libcairo2-dev libcurl4 libdrm-dev libegl1-mesa-dev libepoxy-dev libfile-stripnondeterminism-perl libfontconfig1-dev libfreetype6-dev libgconf-2-4 libgconf2-doc libgdk-pixbuf2.0-dev libglib2.0-dev libglib2.0-dev-bin libglvnd-core-dev libglvnd-dev libgraphite2-dev libharfbuzz-dev libharfbuzz-gobject0 libice-dev libicu-dev libicu-le-hb-dev libicu-le-hb0 libiculx60

sudo apt-get install -d libltdl-dev libmail-sendmail-perl libopengl0 libpango1.0-dev libpcre16-3 libpcre3-dev libpcre32-3 libpcrecpp0v5 libpixman-1-dev libpng-dev libpng-tools libpthread-stubs0-dev libsm-dev libsys-hostname-long-perl libtool libwayland-bin libwayland-dev libx11-dev libx11-doc libxau-dev libxcb-dri2-0-dev libxcb-dri3-dev libxcb-glx0-dev libxcb-present-dev libxcb-randr0-dev libxcb-render0-dev libxcb-shape0-dev libxcb-shm0-dev libxcb-sync-dev libxcb-xfixes0-dev libxcb1-dev libxcomposite-dev libxcursor-dev libxdamage-dev libxdmcp-dev libxext-dev libxfixes-dev libxft-dev libxi-dev libxinerama-dev libxkbcommon-dev libxml2-utils libxrandr-dev

sudo apt-get install -d libxrender-dev libxshmfence-dev libxtst-dev libxxf86vm-dev pkg-config po-debconf python3-distutils python3-lib2to3 wayland-protocols x11proto-composite-dev x11proto-core-dev x11proto-damage-dev x11proto-dev x11proto-dri2-dev x11proto-fixes-dev x11proto-gl-dev x11proto-input-dev x11proto-randr-dev x11proto-record-dev x11proto-xext-dev x11proto-xf86vidmode-dev x11proto-xinerama-dev xorg-sgml-doctools xtrans-dev zlib1g-dev

sudo apt-get install -d libasound2-dev libcurl4-openssl-dev libdbus-glib-1-dev libgconf2-dev libgtk-3-dev libgtk2.0-dev libpulse-dev libxt-dev xvfb yasm

sudo apt-get install -d git ccache gdb lib32z1 gcc-7-multilib g++-7-multilib apache2-utils
```
