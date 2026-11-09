# Tom Clancy's H.A.W.X — Progreso del port (iOS → PS Vita)

## 2026-11-04 — Inicialización y runtime completo (todo offline; falta solo probar en consola)

### Hecho
- `psvita-toolkit init --ipa "IPA/Tom Clancy's H.A.W.X_v.1.1.3.ipa" --name "Tom Clancy's H.A.W.X" --titleid PSVHAWX01 --slug hawx --vita-ip 192.168.3.15` (commit del scaffold).
- Binario `Hawx`: armv6 (ARM), CON símbolos (14k funciones), cryptid 0 (descifrado). MinimumOSVersion 2.2.1.
- Motor: Gameloft sobre Irrlicht (`irr::`) + gameswf (menús Flash `.swf`, fuentes `.ttf`). Binario de 12 MB con símbolos: hooks por nombre posibles (`macho_hook`).
- Arranque: `MainWindow.nib` declara `ComAppDelegate`, que NO existe en el binario: el delegate real llega por `UIApplicationMain` (AppController). El nib igual crea UIWindow+EAGLView (como en iOS).
- **El boilerplate iOS que copia el toolkit no compila ni tiene runtime ObjC** → se reemplazó por el loader probado de Inotia2-vita + un runtime nuevo y común a los 6 ports iOS (copia maestra en `Ferrari-GT-Evolution-vita/source`, se sincroniza con `tools/sync_runtime.sh <port>`). Detalle en `PORTING_PLAN.md` §1.
- `tools/ios_prep.py` → `rebase.bin` **73307** palabras, `binds.bin` **8267** slots / 536 símbolos. Modo: clásico sin relocs (heurístico + estructural). Cobertura de `__text`: 97.7%; 29368 literales puntero, 2263 palabras de tablas de salto; 10418 candidatos rechazados (líneas `DUDA` de `report.txt`).
  - El clasificador se validó contra HAWX (que trae símbolos): 0 punteros perdidos, 0 falsos positivos en datos.
- Cobertura de imports: todos resuelven (implementación, clase vacía autogenerada, NSString constante sintetizada o stub `unimpl` que lo registra en el log).
- C++: `tools/gen_cxx.py` mapea la libstdc++ de iOS (ABI COW) a la de VitaSDK (`source/cxx_imports.c`).
- Audio: OpenAL (1056 `.msadpcm`, los decodifica el juego) + música `.ogg` propia (decoder del juego) por `AudioQueue` PCM.
- Video: Sin videos en el bundle (`demodata`).
- Controles: Avión por acelerómetro (inclinación) → stick izquierdo (roll/pitch); disparo/misil → `controls.txt`.
- Rotación del drawable: 1 (no declarada: confirmar con SELECT+START).
- Build OK: `psvita-toolkit build --preset debug` → `build/hawx.vpk`.
- Datos preparados: `tools/stage_data.sh` → `ux0_data/hawx/` (bundle, `rebase.bin`, `binds.bin`, `nib.txt`, `controls.txt`, música `.ogg`).

### Para probar en consola (Vita 192.168.3.15, VitaShell FTP en 1337)
1. Instalar `build/hawx.vpk`.
2. `python3 -I tools/deploy_data.py` (sube `ux0_data/hawx/` → `ux0:data/hawx/`).
3. Requisitos en la consola: `kubridge.skprx` en `ur0:tai/config.txt` y `libshacccg.suprx` (ShaRKBR33D).
4. Abrir el juego; si se cierra o queda en negro: `python3 -I tools/deploy_data.py --logs` y leer `logs/hawx_NNN.log` (+ `.psp2dmp`).
5. Rotación incorrecta: mantener SELECT+START 2 s (cicla y guarda `rotation.txt`).
6. Botones: editar `ux0:data/hawx/controls.txt` por FTP (coordenadas de lo que se ve), sin recompilar.

### Riesgos conocidos
- Imagen de 11 MB: más presión de memoria.
- `AudioQueue` con formato distinto de PCM 16 bits → se registra y queda sin música.

### Revisión independiente del runtime (misma sesión, antes de la consola)
Un revisor aparte leyó asm.S/objc.c/macho.c. Se corrigió:
- **Crítico:** las cadenas ObjC (selectores, nombres de clase, UTF-16) viven en `__TEXT` y se leían antes de `macho_finalize()` (el bloque RX todavía estaba vacío). Ahora `foundation_fix_cfstrings` + `objc_init_image` corren después; las escrituras en `__TEXT` van por `macho_write_word` (kubridge).
- Pool de autorelease por hilo (TLS); antes era uno solo compartido → use-after-free con hilos/GCD.
- Caché de métodos sin carrera (cls=0 → imp/sel → cls, con barreras y re-chequeo).
- `-dealloc` no puede correr dos veces (rc centinela durante el dealloc).
- `methodForSelector:` devuelve un thunk con `ml_align_call` para IMPs nativas; `objc_ivar_set` libera el valor anterior.
- Pendiente (latente, sin uso en estos binarios): `+initialize`/`+load` no se envían; `nil_imp` en llamadas `_stret` deja el struct sin tocar.
- Verificado correcto por el revisor: balance de pila/registros de todos los trampolines, `ml_dispatch_imp` (Thumb incluido), `objc_msgSendSuper2`, setjmp, cadena de metaclases, mapeo y thunks de macho.c, firmas nativas con CGRect/NSRange/double.

## 2026-11-09 — Corrida 1 en consola: crash en `mod_init[0]`
- Data abort en PC 0x917bc4 (`__symbol_stub4` de `___cxa_atexit`), r12 = 0x0096f39c sin rebasear.
- Causa: los stubs de HAWX son de 12 bytes (`ldr ip,[pc]; ldr pc,[ip]; .long lazy_ptr`) y `ios_prep.py`
  solo rebaseaba los de 8. Se tomó el arreglo de la copia maestra (8 y 12 bytes) → rebase 73307 → **73738** (+431 stubs).
- Pendiente: probar en consola (`deploy_data.py` para subir el `rebase.bin` nuevo).

## 2026-11-09 — Corrida 2: crash en `IStoreable::Save` (playlist)
- Pasan los 191 `mod_init` y arranca `UIApplicationMain`. Al no existir `Documents/playlist`, `IStoreable::Load`
  llama a `Save`, que hace `item->Save(FILE*)` sobre un `CStorageItemVector<CSong>`: `vt[1]` = 0x889150 (mitad de
  `gameswf::edit_text_character::display`) → data abort.
- Causa: relocs externas clásicas contra símbolos **propios** (weak de C++: vtables/VTT/typeinfo). La palabra ya trae
  `val + addend` y `local_or_ext` volvía a sumar `val` → valor duplicado (0x4448a8 → 0x889150). Afecta 1396 relocs.
  Arreglo en `classic_binds`: para símbolo propio se usa la palabra tal cual. Mismo bug en la copia maestra (Ferrari).

## 2026-11-09 — Corrida 3: mismo crash (ahora desde `IStoreable::Load`)
- No era un bug nuevo: `deploy_data.py` solo subía archivos con otro tamaño y el `binds.bin` corregido mide igual
  → la consola seguía con el viejo. Ahora `rebase.bin`, `binds.bin` y `nib.txt` se suben siempre.

## 2026-11-09 — Corrida 5: `std::bad_alloc` en `GSInit::OnEnter`
- Pasa `playlist` (fix weak OK en consola), crea UIWindow/EAGLView/GLES1/NSTimer, carga `data.bar` y `hawx_strings.bar`.
- `GSInit::OnEnter` hace `new(0xc10c)`: el literal 0xc10c cae dentro de `__text` y la heurística lo rebaseaba
  → `new(0x9800c10c)` → bad_alloc → abort.
- Arreglo portado de la copia maestra (`alloc_stubs`): lo que se pasa a `new`/`malloc`/`gl*`/`al*` es tamaño o enum.
  rebase 73738 → 73371 (−367: tamaños y enums GL como 0x8074/0x8d40/0x857x; ninguno fuera de `__text`).
- Ojo en el log: OpenAL "Error creating device/context" aunque `alcOpenDevice` devuelve un device.

## 2026-11-09 — Corrida 7: rutas de Documents con basura (`<ptr><ptr>/hawx/Documents/Settings`)
- rebase 73371 en consola: el bad_alloc se fue. Ahora `CSettings::Load`/`CProfileManager` hacen
  `c = [[docs stringByAppendingPathComponent:..] UTF8String]; [pool release]; fopen(c)`. El free() de newlib
  pisa los 8 primeros bytes ("ux0:data" → fd/bk) → fopen falla → crash posterior en `_free_r`.
- Arreglo (runtime común, también en la copia maestra): `-[NSString dealloc]` pone el buffer en cuarentena
  (anillo de 256) antes del free. Requiere eboot nuevo.

## 2026-11-09 — Corrida 8: crash en `free` desde `MenuStringManager::SetLanguage`
- Rutas de Documents OK (cuarentena de NSString funciona); se crean `Profiles` y `Settings`.
- El string de Gameloft (SSO, buffer interno en `+13`) compara su puntero con `&str+13` (literal impar en
  `__DATA`) para decidir si hace `delete`. La heurística descartaba esos literales (`lit-int-use`: impar + `cmp`)
  → `delete` de memoria estática (`m_CurrentLanguage+13`) → `_free_r` revienta.
- Arreglo: en `__DATA`, impar con uso `int` cuenta como puntero. rebase 73371 → **77712** (+4341, todos `&str+13`).

## 2026-11-09 — Corrida 10: `free` desde `inflateEnd` (CZipReader::openFile)
- Pasa SetLanguage (fix SSO OK). Al abrir un archivo del zip, `inflate` desborda la salida (~25 KB) y pisa su propio
  `inflate_state` con texto → `inflateEnd` libera 0x72657070 ("pper").
- Causa: `lbase` de zlib (u16) tiene 131,163 → palabra 0x00a30083, que cae en `__common` → se rebaseaba.
  Igual con la tabla de pasos ADPCM y charsets/encodings de FreeType.
- Arreglo: palabra de `__TEXT,__const` que apunta a zerofill (`__bss`/`__common`) no es puntero. rebase 77712 → 77686 (−26).

## 2026-11-09 — Corrida 11: crash en `CCommonGLDriver::setTexture`
- Pasan zip/inflate (fix lbase OK), `applicationDidFinishLaunching` vuelve, renderbuffer 320x480, arranca el loop.
- `[driver+0x180]` (unidad de textura activa) = 0x980084c0: el literal `GL_TEXTURE0` (0x84c0) cae en `__text`
  y se rebaseaba (guardado en un campo, no pasa por un stub gl*).
- Arreglo: literal < 0x10000 hacia `__text` solo es puntero si es símbolo/IMP/destino de bl (no basta prólogo).
  rebase 77686 → 77629 (−57: enums GL, 20000/30000/48000/60000, constantes de división). Ninguno era función.

## 2026-11-09 — Corrida 12: ¡llega al title! Pantalla girada 180° y sin táctil/botones
- Rotación: el juego elige orientación por acelerómetro; con x≈-0.88 pidió `setStatusBarOrientation:4` y la
  pantalla seguía en rotación 1 (para 3) → imagen al revés. Arreglo: la pantalla sigue la orientación pedida
  (3→1, 4→2) y el signo del acelerómetro se invirtió para que en reposo coincida con la orientación actual
  (sin oscilar). Solo en este port (el signo puede depender del juego; no se pasó a la maestra).
- Táctil: `hit_view` elegía la última subvista sin mirar el punto → los toques iban a un `UITextField` 0x0.
  Arreglo: hit-test real por frame (solo en `began`; moved/ended siguen a la vista del began). Los botones
  (`controls.txt`) son toques simulados, así que dependían de lo mismo.

## 2026-11-09 — Corrida 13: nombre de perfil no avanza + orientación oscilando 3↔4 cada frame
- `-[EAGLView drawView]` usa `[[UIDevice currentDevice] orientation]` (no el acelerómetro) y la pasa a
  `setOrientation:`. Nuestro `dv_orientation` devolvía 4 con rotación 1 (que muestra derecha la 3) → imagen al
  revés (corrida 12) y, con el acople de la corrida 12, oscilación. Arreglo: rotación 1→3, 2→4 (mismos números
  de UIDevice/UIInterface). Se revirtió el cambio de signo del acelerómetro (el original era el físico).
- Teclado: sceImeDialog funcionaba ("Wonder"), pero HAWX espera `textField:shouldChangeCharactersInRange:
  replacementString:` en el delegate (marca un flag y luego lee `text`). Ahora se llama al confirmar.
