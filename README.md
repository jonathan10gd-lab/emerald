name: Build Emerald IPA
on:
  workflow_dispatch:
jobs:
  build:
    runs-on: macos-latest
    steps:
      - name: Write project files
        run: |
          mkdir -p Sources
          cat > project.yml <<'EOF'
          name: PokeEmu
          options:
            deploymentTarget: {iOS: "16.0"}
          packages:
            ZIPFoundation: {url: "https://github.com/weichsel/ZIPFoundation", from: "0.9.19"}
          targets:
            PokeEmu:
              type: application
              platform: iOS
              sources: [Sources]
              dependencies: [{package: ZIPFoundation}]
              info:
                path: Info.plist
                properties:
                  CFBundleDisplayName: Emerald
                  UILaunchScreen: {}
                  UIRequiresFullScreen: true
                  UIStatusBarHidden: true
                  UIFileSharingEnabled: true
                  LSSupportsOpeningDocumentsInPlace: true
                  UISupportedInterfaceOrientations: [UIInterfaceOrientationLandscapeLeft, UIInterfaceOrientationLandscapeRight]
                  UISupportedInterfaceOrientations~ipad: [UIInterfaceOrientationLandscapeLeft, UIInterfaceOrientationLandscapeRight]
              settings:
                base:
                  PRODUCT_BUNDLE_IDENTIFIER: com.example.emerald
                  SWIFT_OBJC_BRIDGING_HEADER: Sources/Bridge.h
                  TARGETED_DEVICE_FAMILY: "1,2"
                  HEADER_SEARCH_PATHS: ["$(SRCROOT)/mgba/include", "$(SRCROOT)/build/include"]
                  OTHER_LDFLAGS: ["$(SRCROOT)/build/libmgba.a", "-lz"]
          EOF
          cat > Sources/Bridge.h <<'EOF'
          #include <stdint.h>
          int shim_load(const char *path);
          unsigned shim_width(void);
          unsigned shim_height(void);
          const uint32_t *shim_video(void);
          int shim_frame(uint32_t keys, int16_t *audioOut, int maxFrames);
          uint8_t shim_read8(uint32_t addr);
          uint16_t shim_read16(uint32_t addr);
          uint32_t shim_read32(uint32_t addr);
          int shim_hunt_run(int mode, uint32_t dir, int maxD);
          int shim_hunt_progress(void);
          uint32_t shim_hunt_diag(int i);
          int shim_hunt_cap(int mode);
          int shim_hunt_key(int mode, uint32_t dir, int f);
          void shim_hunt_restore(void);
          uint32_t shim_avg_color(void);
          void shim_reload_save(void);
          EOF
          cat > Sources/shim.c <<'EOF'
          #include <mgba/core/core.h>
          #include <mgba/core/blip_buf.h>
          #include <stdlib.h>
          #include "Bridge.h"
          static struct mCore *core; static uint32_t *vbuf; static unsigned W, H;
          int shim_load(const char *path) {
            core = mCoreFind(path); if (!core) return 0;
            core->init(core); mCoreInitConfig(core, NULL);
            W = 240; H = 160;  // GBA screen size is fixed
            core->setAudioBufferSize(core, 1024);
            vbuf = calloc(W * H, 4); core->setVideoBuffer(core, vbuf, W);
            if (!mCoreLoadFile(core, path)) return 0;
            mCoreAutoloadSave(core);
            core->reset(core);
            blip_set_rates(core->getAudioChannel(core, 0), core->frequency(core), 48000);
            blip_set_rates(core->getAudioChannel(core, 1), core->frequency(core), 48000);
            return 1;
          }
          unsigned shim_width(void) { return W; }
          unsigned shim_height(void) { return H; }
          const uint32_t *shim_video(void) { return vbuf; }

          #define PARTY_ADDR 0x020244ECu
          #define ENEMY_ADDR 0x02024744u
          static void *S0, *TT; static uint32_t base[7], rngAddr, lastAddr, diagN; static volatile int progress;
          static uint32_t lcg(uint32_t s) { return s * 0x41C64E6Du + 0x6073u; }
          static void readMons(uint32_t *b) {
            for (int i = 0; i < 6; i++) b[i] = core->busRead32(core, PARTY_ADDR + i * 100);
            b[6] = core->busRead32(core, ENEMY_ADDR);
          }
          static int poll(void) {
            uint32_t c[7]; readMons(c);
            for (int i = 0; i < 7; i++)
              if (c[i] != base[i] && c[i] != 0) { lastAddr = i < 6 ? PARTY_ADDR + i * 100 : ENEMY_ADDR; return 1; }
            return 0;
          }
          int shim_hunt_poll(void) { return poll(); }
          void shim_hunt_restore(void) { if (S0) core->loadState(core, S0); }
          int shim_hunt_progress(void) { return progress; }
          uint32_t shim_hunt_diag(int i) { return i == 0 ? rngAddr : diagN; }
          // mode 0: mash A (static encounters, gifts). mode 1: tap a direction once (step into grass).
          int shim_hunt_cap(int mode) { return mode == 0 ? 900 : 40; }
          int shim_hunt_key(int mode, uint32_t dir, int f) { return mode == 0 ? ((f % 8) < 4 ? 1 : 0) : (f < 4 ? (int)dir : 0); }
          static void idle(void) { core->setKeys(core, 0); core->runFrame(core); }
          static int isShiny(uint32_t pid, uint32_t ot) { return (((ot >> 16) ^ (ot & 0xFFFF) ^ (pid >> 16) ^ (pid & 0xFFFF)) & 0xFFFF) < 8; }
          static int runScript(int mode, uint32_t dir, uint32_t *pid, uint32_t *ot) {
            int cap = shim_hunt_cap(mode);
            for (int f = 0; f < cap; f++) {
              core->setKeys(core, shim_hunt_key(mode, dir, f)); core->runFrame(core);
              if (poll()) { *pid = core->busRead32(core, lastAddr); *ot = core->busRead32(core, lastAddr + 4); return 1; }
            }
            return 0;
          }
          // Find the RNG word in IWRAM: the one that moves along the LCG sequence while we idle.
          static int detectRng(void) {
            static uint32_t a[8192];
            for (int i = 0; i < 8192; i++) a[i] = core->busRead32(core, 0x03000000u + i * 4);
            for (int f = 0; f < 8; f++) idle();
            rngAddr = 0;
            for (int i = 0; i < 8192; i++) {
              uint32_t b = core->busRead32(core, 0x03000000u + i * 4), s = a[i];
              if (b == s) continue;
              for (int t = 1; t <= 4096; t++) {
                s = lcg(s);
                if (s == b) { uint32_t ad = 0x03000000u + i * 4; if (!rngAddr || ad == 0x03005D80u) rngAddr = ad; break; }
              }
            }
            core->loadState(core, S0);
            return rngAddr != 0;
          }
          // Returns the delay d (frames to idle) that gives a verified shiny, leaving the core at that moment.
          // Errors: -1 no RNG, -2 nothing happened, -3 couldn't align RNG, -4 no shiny found.
          int shim_hunt_run(int mode, uint32_t dir, int maxD) {
            progress = 0; diagN = 0;
            size_t sz = core->stateSize(core);
            free(S0); free(TT); S0 = malloc(sz); TT = malloc(sz);
            if (!core->saveState(core, S0)) return -1;
            readMons(base);
            if (!detectRng()) return -1;
            uint32_t pid = 0, ot = 0, baseR = 0; int d0 = 0, got = 0;
            for (;;) {  // baseline: first delay where something actually spawns
              baseR = core->busRead32(core, rngAddr);
              core->saveState(core, TT);
              if (runScript(mode, dir, &pid, &ot)) { got = 1; break; }
              core->loadState(core, TT);
              if (mode == 0 || d0 >= 3000) break;
              idle(); d0++; progress = d0;
            }
            if (!got) { core->loadState(core, S0); return -2; }
            core->loadState(core, TT);
            if (isShiny(pid, ot)) return d0;
            uint32_t lo = pid & 0xFFFF, hi = pid >> 16, tsid = (ot >> 16) ^ (ot & 0xFFFF), t = baseR;
            int n = -1;
            for (int i = 0; i < 400000; i++) {
              uint32_t a = lcg(t), b = lcg(a), f = a >> 16, g = b >> 16;
              if ((f == lo && g == hi) || (f == hi && g == lo)) { n = i; break; }
              t = a;
            }
            if (n < 0) { core->loadState(core, S0); return -3; }
            diagN = (uint32_t)n;
            uint32_t An = 1, Cn = 0;  // lcg^n(x) = An*x + Cn
            for (int i = 0; i < n; i++) { An *= 0x41C64E6Du; Cn = Cn * 0x41C64E6Du + 0x6073u; }
            for (int d = d0 + 1; d <= maxD; d++) {
              idle(); progress = d;
              uint32_t x = An * core->busRead32(core, rngAddr) + Cn, a = lcg(x), b = lcg(a);
              if ((((tsid ^ (a >> 16) ^ (b >> 16)) & 0xFFFF) < 8)) {
                core->saveState(core, TT);
                if (runScript(mode, dir, &pid, &ot) && isShiny(pid, ot)) { core->loadState(core, TT); return d; }
                core->loadState(core, TT);
              }
            }
            core->loadState(core, S0);
            return -4;
          }
          uint32_t shim_avg_color(void) {
            if (!vbuf) return 0;
            uint64_t r = 0, g = 0, b = 0; unsigned n = 0;
            for (unsigned y = 0; y < H; y += 2)
              for (unsigned x = 0; x < W; x += 2) {
                uint32_t p = vbuf[y * W + x]; r += p & 255; g += (p >> 8) & 255; b += (p >> 16) & 255; n++;
              }
            return n ? (uint32_t)(((r / n) << 16) | ((g / n) << 8) | (b / n)) : 0;
          }
          void shim_reload_save(void) { if (core) { mCoreAutoloadSave(core); core->reset(core); } }
          uint8_t shim_read8(uint32_t a) { return core->busRead8(core, a); }
          uint16_t shim_read16(uint32_t a) { return core->busRead16(core, a); }
          uint32_t shim_read32(uint32_t a) { return core->busRead32(core, a); }
          int shim_frame(uint32_t keys, int16_t *out, int max) {
            core->setKeys(core, keys); core->runFrame(core);
            blip_t *l = core->getAudioChannel(core, 0), *r = core->getAudioChannel(core, 1);
            int n = blip_samples_avail(l); if (n > max) n = max;
            blip_read_samples(l, out, n, 1); blip_read_samples(r, out + 1, n, 1);
            return n;
          }
          EOF
          cat > Sources/App.swift <<'EOF'
          import SwiftUI
          import GameController
          import AVFoundation
          import ZIPFoundation
          import UniformTypeIdentifiers

          @main struct EmeraldApp: App {
            @StateObject var emu = Emu()
            @State private var picking = false
            @State private var pickingSav = false
            @State private var share: ShareItem?
            var body: some Scene {
              WindowGroup {
                ZStack {
                  (emu.ambient && emu.running ? emu.avgColor : Color.black)
                    .animation(.linear(duration: 0.2), value: emu.avgRGB)
                  if emu.running {
                    GeometryReader { geo in
                      let d = min(geo.size.height / 6.6, geo.size.width * 0.12)
                      HStack(spacing: 0) {
                        ScreenRep(view: emu.screen)
                        PartyPanel(party: emu.party, d: d).frame(width: d + 16)
                      }
                    }
                  }
                  else {
                    VStack(spacing: 16) {
                      Text("Pokémon Emerald").font(.title).foregroundColor(.white)
                      Button("Load ROM (.zip or .gba)") { picking = true }.buttonStyle(.borderedProminent)
                      Button("Import save (.sav)") { pickingSav = true }.buttonStyle(.bordered)
                      Button("Export save") { if let u = emu.exportSave() { share = ShareItem(url: u) } }.buttonStyle(.bordered)
                      Toggle("Ambient background (average screen color)", isOn: $emu.ambient)
                        .foregroundColor(.white).frame(width: 380)
                      if let e = emu.error { Text(e).foregroundColor(.red) }
                    }
                  }
                }
                .overlay(alignment: .topLeading) {
                  if emu.running {
                    Menu {
                      Button("Static / gift (presses A)") { emu.startHunt(mode: 0, dir: 0) }
                      Button("Wild: step up") { emu.startHunt(mode: 1, dir: 64) }
                      Button("Wild: step down") { emu.startHunt(mode: 1, dir: 128) }
                      Button("Wild: step left") { emu.startHunt(mode: 1, dir: 32) }
                      Button("Wild: step right") { emu.startHunt(mode: 1, dir: 16) }
                      Picker("Speed", selection: $emu.speed) {
                        Text("1x").tag(1); Text("2x").tag(2); Text("4x").tag(4); Text("8x").tag(8)
                      }
                      Divider()
                      Button("Export save") { if let u = emu.exportSave() { share = ShareItem(url: u) } }
                      Button("Import save") { pickingSav = true }
                      Toggle("Ambient background", isOn: $emu.ambient)
                    } label: { Text("✦ Menu") }.font(.caption).padding(6)
                      .background(.black.opacity(0.5)).foregroundColor(.yellow).cornerRadius(8).padding(12)
                  }
                }
                .overlay(alignment: .top) {
                  if let m = emu.huntMsg {
                    Text(m).font(.callout).padding(8).background(.black.opacity(0.75))
                      .foregroundColor(.yellow).cornerRadius(8).padding(.top, 12)
                  }
                }
                .overlay(alignment: .topTrailing) {
                  if emu.speed > 1 {
                    Text("▶▶ \(emu.speed)x").font(.caption.bold()).foregroundColor(.yellow).padding(6)
                      .background(.black.opacity(0.5)).cornerRadius(8).padding(12)
                  }
                }
                .ignoresSafeArea().statusBarHidden(true).persistentSystemOverlays(.hidden)
                .sheet(isPresented: $picking) {
                  DocPicker { url in picking = false; emu.importROM(url) }.ignoresSafeArea()
                }
                .sheet(isPresented: $pickingSav) {
                  DocPicker { url in pickingSav = false; emu.importSave(url) }.ignoresSafeArea()
                }
                .sheet(item: $share) { ShareSheet(url: $0.url) }
                .onAppear { UIApplication.shared.isIdleTimerDisabled = true; emu.autoStart() }
              }
            }
          }

          struct DocPicker: UIViewControllerRepresentable {
            let onPick: (URL) -> Void
            func makeCoordinator() -> Coord { Coord(onPick) }
            func makeUIViewController(context: Context) -> UIDocumentPickerViewController {
              let p = UIDocumentPickerViewController(forOpeningContentTypes: [.zip, .item], asCopy: true)
              p.delegate = context.coordinator
              return p
            }
            func updateUIViewController(_ c: UIDocumentPickerViewController, context: Context) {}
            final class Coord: NSObject, UIDocumentPickerDelegate {
              let onPick: (URL) -> Void
              init(_ f: @escaping (URL) -> Void) { onPick = f }
              func documentPicker(_ c: UIDocumentPickerViewController, didPickDocumentsAt urls: [URL]) {
                if let u = urls.first { onPick(u) }
              }
            }
          }

          struct ScreenRep: UIViewRepresentable {
            let view: UIView
            func makeUIView(context: Context) -> UIView { view }
            func updateUIView(_ v: UIView, context: Context) {}
          }

          final class ScreenView: UIView {
            override init(frame: CGRect) {
              super.init(frame: frame)
              layer.magnificationFilter = .linear
              layer.minificationFilter = .linear
              layer.contentsGravity = .resizeAspect
              backgroundColor = .clear
            }
            required init?(coder: NSCoder) { fatalError() }
          }

          final class Emu: NSObject, ObservableObject {
            @Published var running = false
            @Published var error: String?
            @Published var party: [Mon] = []
            private var frame = 0
            @Published var hunting = false
            @Published var speed = 1
            @Published var avgRGB: UInt32 = 0
            @Published var ambient = UserDefaults.standard.bool(forKey: "ambient") { didSet { UserDefaults.standard.set(ambient, forKey: "ambient") } }
            @Published var huntMsg: String?
            var mashing = false
            var mashF = 0
            var xDown = false
            var r3Down = false
            var yDown = false
            var huntMode: Int32 = 0
            var huntDir: UInt32 = 0
            func resetAudio() { lock.lock(); ring.removeAll(); rp = 0; lock.unlock() }
            let screen = ScreenView()
            private var link: CADisplayLink?
            private let engine = AVAudioEngine()
            private let lock = NSLock()
            private var ring = [Float](); private var rp = 0
            private var abuf = [Int16](repeating: 0, count: 4096)
            private let cs = CGColorSpaceCreateDeviceRGB()
            // 4x nearest-neighbour upscale, then smooth scaling to the screen = sharp, even pixels at any size
            private lazy var hi = CGContext(data: nil, width: 960, height: 640, bitsPerComponent: 8, bytesPerRow: 0, space: cs, bitmapInfo: CGImageAlphaInfo.noneSkipLast.rawValue)!
            var docs: URL { FileManager.default.urls(for: .documentDirectory, in: .userDomainMask)[0] }

            func autoStart() {
              let p = docs.appendingPathComponent("game.gba")
              if FileManager.default.fileExists(atPath: p.path) { start(p.path) }
            }

            func importROM(_ url: URL) {
              let ok = url.startAccessingSecurityScopedResource()
              defer { if ok { url.stopAccessingSecurityScopedResource() } }
              let fm = FileManager.default
              let dest = docs.appendingPathComponent("game.gba")
              do {
                var src = url
                if url.pathExtension.lowercased() == "zip" {
                  let tmp = fm.temporaryDirectory.appendingPathComponent(UUID().uuidString)
                  try fm.createDirectory(at: tmp, withIntermediateDirectories: true)
                  try fm.unzipItem(at: url, to: tmp)
                  let found = fm.enumerator(at: tmp, includingPropertiesForKeys: nil)?
                    .compactMap { $0 as? URL }.first { ["gba", "agb", "bin"].contains($0.pathExtension.lowercased()) }
                  guard let f = found else { error = "No .gba file found in zip"; return }
                  src = f
                }
                try? fm.removeItem(at: dest)
                try fm.copyItem(at: src, to: dest)
                start(dest.path)
              } catch { self.error = error.localizedDescription }
            }

            func start(_ path: String) {
              guard shim_load(path) != 0 else { error = "Could not load ROM"; return }
              error = nil
              let fmt = AVAudioFormat(commonFormat: .pcmFormatFloat32, sampleRate: 48000, channels: 2, interleaved: false)!
              let node = AVAudioSourceNode(format: fmt) { [weak self] _, _, frames, abl in
                let b = UnsafeMutableAudioBufferListPointer(abl)
                let l = b[0].mData!.assumingMemoryBound(to: Float.self)
                let r = b[1].mData!.assumingMemoryBound(to: Float.self)
                guard let s = self else { return noErr }
                s.lock.lock(); defer { s.lock.unlock() }
                for i in 0..<Int(frames) {
                  if s.ring.count - s.rp >= 2 { l[i] = s.ring[s.rp]; r[i] = s.ring[s.rp + 1]; s.rp += 2 }
                  else { l[i] = 0; r[i] = 0 }
                }
                return noErr
              }
              try? AVAudioSession.sharedInstance().setCategory(.playback)
              try? AVAudioSession.sharedInstance().setActive(true)
              engine.attach(node); engine.connect(node, to: engine.mainMixerNode, format: fmt)
              try? engine.start()
              link = CADisplayLink(target: self, selector: #selector(tick)); link?.preferredFrameRateRange = CAFrameRateRange(minimum: 60, maximum: 60, preferred: 60)
              link?.add(to: .main, forMode: .common)
              running = true
            }

            @objc func tick() {
              if hunting { return }
              pollHuntButtons()
              if hunting { return }
              var k = readKeys()
              if mashing { k = UInt32(shim_hunt_key(huntMode, huntDir, Int32(mashF))); mashF += 1 }
              var n = Int(shim_frame(k, &abuf, 2048))
              if speed > 1 && !mashing {   // fast-forward: extra frames per tick, no audio
                for _ in 1..<speed { _ = shim_frame(readKeys(), &abuf, 2048) }
                n = 0
              }
              if mashing && (shim_hunt_poll() != 0 || mashF >= Int(shim_hunt_cap(huntMode))) {
                mashing = false; huntMsg = "Done!"
                DispatchQueue.main.asyncAfter(deadline: .now() + 5) { self.huntMsg = nil }
              }
              lock.lock()
              if rp > 16384 { ring.removeFirst(rp); rp = 0 }
              if ring.count - rp > 9600 { ring.removeAll(); rp = 0 }
              for i in 0..<(n * 2) { ring.append(Float(abuf[i]) / 32768) }
              lock.unlock()
              frame += 1
              if frame % 6 == 0 { let p = readParty(); if p != party { party = p } }
              if ambient && frame % 6 == 3 { updateAvg() }
              let w = Int(shim_width()), h = Int(shim_height())
              if let ptr = shim_video(),
                 let ctx = CGContext(data: UnsafeMutableRawPointer(mutating: ptr), width: w, height: h, bitsPerComponent: 8,
                                     bytesPerRow: w * 4, space: cs, bitmapInfo: CGImageAlphaInfo.noneSkipLast.rawValue) {
                if let img = ctx.makeImage() {
                  hi.interpolationQuality = .none
                  hi.draw(img, in: CGRect(x: 0, y: 0, width: 960, height: 640))
                  screen.layer.contents = hi.makeImage()
                }
              }
            }

            // GBA bits: A0 B1 Select2 Start3 Right4 Left5 Up6 Down7 R8 L9
            // Apple's buttonA/B are positional (bottom/right), so right button -> GBA A matches both Xbox and Switch Pro.
            func readKeys() -> UInt32 {
              guard let g = GCController.controllers().first?.extendedGamepad else { return 0 }
              var k: UInt32 = 0
              if g.buttonB.isPressed { k |= 1 }
              if g.buttonA.isPressed { k |= 2 }
              if g.buttonOptions?.isPressed == true { k |= 4 }
              if g.buttonMenu.isPressed { k |= 8 }
              let x = g.leftThumbstick.xAxis.value, y = g.leftThumbstick.yAxis.value
              if g.dpad.right.isPressed || x > 0.5 { k |= 16 }
              if g.dpad.left.isPressed || x < -0.5 { k |= 32 }
              if g.dpad.up.isPressed || y > 0.5 { k |= 64 }
              if g.dpad.down.isPressed || y < -0.5 { k |= 128 }
              if g.rightShoulder.isPressed { k |= 256 }
              if g.leftShoulder.isPressed { k |= 512 }
              return k
            }
          }
          EOF
          cat > Sources/Party.swift <<'EOF'
          import SwiftUI

          struct Mon: Identifiable, Equatable { let id: Int; let dex: Int; let hp: Int; let maxHP: Int; let level: Int; let shiny: Bool }

          // Emerald internal species 277...411 (Hoenn order) -> National Dex number
          private let hoennToDex: [Int] = [252,253,254,255,256,257,258,259,260,261,262,263,264,265,266,267,268,269,270,271,272,273,274,275,290,291,292,276,277,285,286,327,278,279,283,284,320,321,300,301,352,343,344,299,324,302,339,340,370,341,342,349,350,318,319,328,329,330,296,297,309,310,322,323,363,364,365,331,332,361,362,337,338,298,325,326,311,312,303,307,308,333,334,360,355,356,315,287,288,289,316,317,357,293,294,295,366,367,368,359,353,354,336,335,369,304,305,306,351,313,314,345,346,347,348,280,281,282,371,372,373,374,375,376,377,378,379,382,383,384,380,381,385,386,358]

          func nationalDex(_ s: Int) -> Int? {
            if s >= 1 && s <= 251 { return s }
            if s >= 277 && s <= 411 { return hoennToDex[s - 277] }
            return nil   // 0, unused slots, and eggs (412)
          }

          private let orders = ["GAEM","GAME","GEAM","GEMA","GMAE","GMEA","AGEM","AGME","AEGM","AEMG","AMGE","AMEG",
                                "EGAM","EGMA","EAGM","EAMG","EMGA","EMAG","MGAE","MGEA","MAGE","MAEG","MEGA","MEAG"]

          extension Emu {
            // Emerald (USA): gPlayerPartyCount 0x020244E9, gPlayerParty 0x020244EC, 100 bytes per Pokemon
            func readParty() -> [Mon] {
              let n = Int(shim_read8(0x020244E9))
              if n < 1 || n > 6 { return [] }
              var out: [Mon] = []
              for i in 0..<n {
                let b = UInt32(0x020244EC + i * 100)
                let pid = shim_read32(b), ot = shim_read32(b + 4)
                let key = pid ^ ot
                let order = Array(orders[Int(pid % 24)])
                guard let g = order.firstIndex(of: "G") else { continue }
                let word = shim_read32(b + 0x20 + UInt32(g * 12)) ^ key
                guard let dex = nationalDex(Int(word & 0xFFFF)) else { continue }
                out.append(Mon(id: i, dex: dex, hp: Int(shim_read16(b + 0x56)),
                               maxHP: Int(shim_read16(b + 0x58)), level: Int(shim_read8(b + 0x54)),
                               shiny: ((ot >> 16) ^ (ot & 0xFFFF) ^ (pid >> 16) ^ (pid & 0xFFFF)) < 8))
              }
              return out
            }
          }

          struct PartyPanel: View {
            let party: [Mon]; let d: CGFloat
            var body: some View {
              VStack(spacing: d * 0.08) { ForEach(party) { Ring(m: $0, d: d) } }
                .frame(maxHeight: .infinity).allowsHitTesting(false)
            }
          }

          struct Ring: View {
            let m: Mon; let d: CGFloat
            var frac: CGFloat { min(1, CGFloat(m.hp) / CGFloat(max(m.maxHP, 1))) }
            var color: Color { frac > 0.5 ? .green : (frac > 0.2 ? .orange : .red) }
            var body: some View {
              ZStack {
                Circle().fill(Color.black.opacity(0.55))
                Circle().stroke(Color.white.opacity(0.2), lineWidth: d * 0.09)
                Circle().trim(from: 0, to: frac)
                  .stroke(color, style: StrokeStyle(lineWidth: d * 0.09, lineCap: .round))
                  .rotationEffect(.degrees(-90))
                  .animation(.easeInOut(duration: 0.7), value: frac)
                AsyncImage(url: URL(string: "https://raw.githubusercontent.com/PokeAPI/sprites/master/sprites/pokemon/versions/generation-vii/icons/\(m.dex).png")) { img in
                  img.resizable().interpolation(.none).scaledToFit()
                } placeholder: { Text("#\(m.dex)").font(.system(size: d * 0.22)).foregroundColor(.white) }
                .padding(d * 0.08)
              }.frame(width: d, height: d)
              .overlay(alignment: .topTrailing) { if m.shiny { Text("✦").font(.system(size: d * 0.3)).foregroundColor(.yellow) } }
            }
          }
          EOF
          cat > Sources/Hunt.swift <<'EOF'
          import Foundation
          import GameController

          extension Emu {
            func dirBits(_ g: GCExtendedGamepad) -> UInt32 {
              let sx = g.leftThumbstick.xAxis.value, sy = g.leftThumbstick.yAxis.value
              if g.dpad.right.isPressed || sx > 0.5 { return 16 }
              if g.dpad.left.isPressed || sx < -0.5 { return 32 }
              if g.dpad.up.isPressed || sy > 0.5 { return 64 }
              if g.dpad.down.isPressed || sy < -0.5 { return 128 }
              return 0
            }

            // Y = static / gift hunt.  X = wild hunt in the direction held on the d-pad / left stick.
            func pollHuntButtons() {
              guard let g = GCController.controllers().first?.extendedGamepad else { return }
              let x = g.buttonX.isPressed, y = g.buttonY.isPressed
              if x && !xDown { startHunt(mode: 1, dir: dirBits(g)) }
              if y && !yDown { startHunt(mode: 0, dir: 0) }
              xDown = x; yDown = y
              let r3 = g.rightThumbstickButton?.isPressed ?? false
              if r3 && !r3Down { cycleSpeed() }
              r3Down = r3
            }

            // R3 (right stick click) cycles 1x -> 2x -> 4x -> 8x
            func cycleSpeed() {
              speed = speed >= 8 ? 1 : speed * 2
              huntMsg = "Speed \(speed)x"
              DispatchQueue.main.asyncAfter(deadline: .now() + 1.5) { if self.huntMsg == "Speed \(self.speed)x" { self.huntMsg = nil } }
            }

            func startHunt(mode: Int32, dir: UInt32) {
              guard running, !hunting, !mashing else { return }
              if mode == 1 && dir == 0 {
                huntMsg = "Hold the direction toward the grass, then press X."
                DispatchQueue.main.asyncAfter(deadline: .now() + 4) { self.huntMsg = nil }
                return
              }
              hunting = true; huntMsg = "Shiny hunt: scanning…"
              let timer = Timer.scheduledTimer(withTimeInterval: 0.5, repeats: true) { _ in
                self.huntMsg = "Searching… frame \(shim_hunt_progress())"
              }
              DispatchQueue.global(qos: .userInitiated).async {
                let d = shim_hunt_run(mode, dir, 150000)
                let addr = String(format: "%08X", shim_hunt_diag(0)), n = shim_hunt_diag(1)
                DispatchQueue.main.async {
                  timer.invalidate()
                  self.resetAudio()
                  self.hunting = false
                  if d >= 0 {
                    self.huntMode = mode; self.huntDir = dir; self.mashF = 0; self.mashing = true
                    self.huntMsg = "✦ Shiny locked in (waited \(d) frames)"
                  } else {
                    switch d {
                    case -1: self.huntMsg = "E1: can't find the game's RNG. Stand still and try again."
                    case -2: self.huntMsg = mode == 0
                        ? "E2: nothing appeared after pressing A. Face the ball or Pokémon."
                        : "E2: no wild encounter. Face the grass, stand still, hold that direction."
                    case -3: self.huntMsg = "E3: couldn't line up the RNG (addr \(addr))."
                    default: self.huntMsg = "E4: no shiny verified (addr \(addr), n \(n)). Try again."
                    }
                    DispatchQueue.main.asyncAfter(deadline: .now() + 8) { self.huntMsg = nil }
                  }
                }
              }
            }
          }
          EOF
          cat > Sources/Save.swift <<'EOF'
          import SwiftUI

          struct ShareItem: Identifiable { let id = UUID(); let url: URL }

          struct ShareSheet: UIViewControllerRepresentable {
            let url: URL
            func makeUIViewController(context: Context) -> UIActivityViewController {
              UIActivityViewController(activityItems: [url], applicationActivities: nil)
            }
            func updateUIViewController(_ c: UIActivityViewController, context: Context) {}
          }

          extension Emu {
            var savURL: URL { docs.appendingPathComponent("game.sav") }

            func flash(_ m: String) {
              huntMsg = m
              DispatchQueue.main.asyncAfter(deadline: .now() + 6) { self.huntMsg = nil }
            }

            // Copies the save to a nicely named temp file for the share sheet.
            func exportSave() -> URL? {
              let fm = FileManager.default
              guard fm.fileExists(atPath: savURL.path) else { flash("No save file yet. Save in the game first."); return nil }
              let dst = fm.temporaryDirectory.appendingPathComponent("Pokemon Emerald.sav")
              try? fm.removeItem(at: dst)
              do { try fm.copyItem(at: savURL, to: dst); return dst } catch { flash("Couldn't export: \(error.localizedDescription)"); return nil }
            }

            func importSave(_ url: URL) {
              let fm = FileManager.default
              let ok = url.startAccessingSecurityScopedResource()
              defer { if ok { url.stopAccessingSecurityScopedResource() } }
              guard let data = try? Data(contentsOf: url) else { flash("Couldn't read that file."); return }
              guard data.count >= 131072 else { flash("That doesn't look like an Emerald save (needs 128 KB)."); return }
              if fm.fileExists(atPath: savURL.path) {   // keep a backup of the current save
                let bak = docs.appendingPathComponent("game.sav.bak")
                try? fm.removeItem(at: bak); try? fm.copyItem(at: savURL, to: bak)
              }
              do { try data.prefix(131072).write(to: savURL) } catch { flash("Couldn't import: \(error.localizedDescription)"); return }
              if running { shim_reload_save(); flash("Save imported. Game restarted with it.") }
              else { flash("Save imported. It loads when you start the game.") }
            }

            // Average screen color, smoothed so the background fades instead of flickering.
            func updateAvg() {
              let c = shim_avg_color(), o = avgRGB
              func ch(_ v: UInt32, _ sh: UInt32) -> Double { Double((v >> sh) & 255) }
              func mix(_ sh: UInt32) -> UInt32 { UInt32((ch(o, sh) + (ch(c, sh) - ch(o, sh)) * 0.3).rounded()) }
              avgRGB = (mix(16) << 16) | (mix(8) << 8) | mix(0)
            }

            var avgColor: Color {
              Color(red: Double((avgRGB >> 16) & 255) / 255, green: Double((avgRGB >> 8) & 255) / 255, blue: Double(avgRGB & 255) / 255)
            }
          }
          EOF
      - name: Build mGBA for iOS
        run: |
          git clone --depth 1 --branch 0.10.3 https://github.com/mgba-emu/mgba mgba
          grep -rl popcount32 mgba/src | xargs sed -i '' '/inline/!s/popcount32(/__builtin_popcount(/g' || true
          cmake -S mgba -B build -DCMAKE_SYSTEM_NAME=iOS -DCMAKE_OSX_SYSROOT=iphoneos \
            -DCMAKE_OSX_ARCHITECTURES=arm64 -DCMAKE_OSX_DEPLOYMENT_TARGET=16.0 \
            -DCMAKE_TRY_COMPILE_TARGET_TYPE=STATIC_LIBRARY -DCMAKE_POLICY_VERSION_MINIMUM=3.5 \
            -DBUILD_SHARED=OFF -DBUILD_STATIC=ON -DBUILD_QT=OFF -DBUILD_SDL=OFF -DBUILD_GL=OFF \
            -DBUILD_GLES2=OFF -DBUILD_GLES3=OFF -DBUILD_LIBRETRO=OFF -DBUILD_PERF=OFF -DBUILD_TEST=OFF \
            -DBUILD_EXAMPLE=OFF -DBUILD_HEADLESS=OFF -DBUILD_UPDATER=OFF -DM_CORE_GB=OFF -DM_CORE_GBA=ON \
            -DUSE_FFMPEG=OFF -DUSE_PNG=OFF -DUSE_LIBZIP=OFF -DUSE_MINIZIP=OFF -DUSE_LZMA=OFF -DUSE_SQLITE3=OFF \
            -DUSE_ELF=OFF -DUSE_EDITLINE=OFF -DUSE_EPOXY=OFF -DUSE_DISCORD_RPC=OFF -DUSE_DEBUGGERS=OFF \
            -DENABLE_SCRIPTING=OFF -DENABLE_GDB_STUB=OFF
          cmake --build build --target mgba -j4 -- -k
          ls -la build/*.a
      - name: Build app
        run: |
          brew install xcodegen
          xcodegen
          xcodebuild -project PokeEmu.xcodeproj -scheme PokeEmu -sdk iphoneos -configuration Release \
            -derivedDataPath out CODE_SIGNING_ALLOWED=NO CODE_SIGN_IDENTITY="" build
          mkdir Payload && cp -R out/Build/Products/Release-iphoneos/PokeEmu.app Payload/
          zip -qr Emerald.ipa Payload
      - uses: actions/upload-artifact@v4
        with:
          name: Emerald-ipa
          path: Emerald.ipa
