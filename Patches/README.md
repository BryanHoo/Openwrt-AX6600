# QCA8081 port clock backport

`qualcommax-qca8081-clock-fix.patch` backports OpenWrt
[PR #24566](https://github.com/openwrt/openwrt/pull/24566), captured on
2026-11-06. It contains commits
`6985f61db8aeb2911a7e79515bace5690d35cd1b` and the PPE parent-selection
commit recorded in the second mail header. Original authorship, commit
messages and Signed-off-by trailers are retained. Only the include hunk's
context was reduced to accommodate this fork's additional QCA PPE header;
the added C code and GCC patches are unchanged.

On IPQ601x the bootloader can leave port 5's TX RCG parent on the UNIPHY RX
clock. Merely setting an already matching rate does not correct the parent,
so the QCA8081 can send corrupt frames while its own TX error counters stay
zero. The receiver reports CRC errors. This series selects the PCS RX/TX
parents before the first port rate change and keeps port 5 on UNIPHY1 when
switching to SGMII rates. Both parts are required.

The common build workflow applies the checked-in series before configuring
PURE or PLUS. It skips non-PPE source trees, accepts an exactly complete
existing backport, and stops on unsupported kernels, partial backports or
source conflicts. It never fetches a mutable PR patch during the build.
When upstream incorporates or changes the implementation, review and remove
or refresh this backport rather than bypassing a failed check.

## Validation

Run against an unmodified real source checkout:

```sh
git clone --depth=1 --branch owrt https://github.com/ones20250/immortalwrt_ipq.git source
bash tests/test_qca8081_clock_fix.sh "$PWD/source"
```

`QCA8081 clock validation` runs the regression suite on PRs. Manually dispatch
it with `FULL_BUILD=true` to compile both profiles; these builds upload
artifacts without publishing firmware releases. Configuration-only builds
are not kernel or firmware compile validation.

Hardware acceptance requires a maintenance window and the exact compiled
image: preserve a working firmware/configuration backup, connect the same
known-good cable and peer at 2.5G, and verify
`/sys/kernel/debug/clk/nss_port5_tx_clk_src/clk_parent` is
`uniphy1_gcc_tx_clk` (RX remains `uniphy1_gcc_rx_clk`). Check receiver CRC
increments and TCP retransmissions during bidirectional traffic, then
exercise 1G/2.5G renegotiation and cold/warm boot. Successful compilation
does not constitute hardware acceptance; unrelated cable/PHY faults can
still produce CRC errors.
