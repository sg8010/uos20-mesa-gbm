# UOS 20 ARM64 private Mesa GBM test build

Builds Mesa 21.1.8 GBM and a smoke probe in Debian Buster on a native ARM64 GitHub runner. Artifacts require GLIBC <= 2.28.

See [the build and target-machine test guide](uos20-chrome-mesa21-build-guide.md). The workflow includes a local Mesa build-selection change to retain the DRI backend without compiling render drivers. CI success does not establish UOS GPU or Chrome compatibility.

Run `Build Mesa GBM for UOS20 arm64` manually in Actions.

Verified build: [Actions run 37001963086](https://github.com/sg8010/uos20-mesa-gbm/actions/runs/37001963086), commit `29d76ad`, 2026-10-02. Both GBM and the probe are AArch64 and require at most GLIBC 2.17. DRI backend compilation and Buster dynamic loading passed. Target GPU and Chrome behavior remain untested.
