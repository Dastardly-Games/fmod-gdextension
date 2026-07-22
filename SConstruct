#!/usr/bin/env python
import os
import shutil
import subprocess

from SCons.Script import SConscript, ARGUMENTS, Action, Copy, CacheDir, Decider

# ---- SCons build-node cache (CI) ----
# Content-addressed .o/.a cache (godot-cpp's SConstruct pattern) so unchanged
# sources skip recompilation. ccache can't reach the Windows mingw build
# (absolute-path compiler, no launcher hook); SCons' CacheDir keyed on MD5 does
# the caching instead. Dir from $SCONS_CACHE (set by the CI setup, persisted
# outside the checkout); no-op locally when unset.
scons_cache_path = os.environ.get("SCONS_CACHE")
if scons_cache_path is not None:
    CacheDir(scons_cache_path)
    Decider("MD5")

target_path = ARGUMENTS.pop("target_path", "demo/addons/fmod/libs/")
target_name = ARGUMENTS.pop("target_name", "libGodotFmod")
fmod_lib_dir = ARGUMENTS.pop("fmod_lib_dir", "../libs/fmod/")

# FANTASY_TW: reuse a prebuilt godot-cpp (e.g. the host's double-precision build)
# instead of this submodule's own godot-cpp (pinned godot-4.5). Lets the FMOD
# wrapper match the host engine's precision/version — see
# docs/toolchain_double_precision.md. Linux-focused; other platforms keep the
# upstream SConscript path. The FMOD SDK .so's are precision-agnostic.
_reuse_gc = ARGUMENTS.get("reuse_godot_cpp", "")
_precision = ARGUMENTS.get("precision", "single")
if _reuse_gc:
    _plat = ARGUMENTS.get("platform", "linux")
    _tgt = ARGUMENTS.get("target", "template_debug")
    _arch = ARGUMENTS.get("arch", "x86_64")
    _prec_infix = ".double" if _precision == "double" else ""
    _gc_lib = "{}/bin/libgodot-cpp.{}.{}{}.{}.a".format(_reuse_gc, _plat, _tgt, _prec_infix, _arch)
    if not os.path.exists(_gc_lib):
        print("ERROR: reuse_godot_cpp lib missing: " + _gc_lib)
        Exit(1)
    env = Environment()
    env.Append(CXXFLAGS=["-fno-gnu-unique", "-std=c++17", "-fno-exceptions",
                         "-fPIC", "-fvisibility=hidden", "-O2"])
    _defs = ["LINUX_ENABLED", "UNIX_ENABLED", "THREADS_ENABLED", "GDEXTENSION"]
    _defs += (["DEBUG_ENABLED", "HOT_RELOAD_ENABLED"] if _tgt == "template_debug" else ["NDEBUG"])
    if _precision == "double":
        _defs.append("REAL_T_IS_DOUBLE")
    env.Append(CPPDEFINES=_defs)
    env.Append(CPPPATH=[_reuse_gc + "/include", _reuse_gc + "/gen/include"])
    env.Append(LIBS=[File(_gc_lib)])
    env["platform"] = _plat
    env["target"] = _tgt
    env["arch"] = _arch
    env["SHLIBSUFFIX"] = ".so"
    print("FMOD: REUSING prebuilt godot-cpp -> " + _gc_lib)
else:
    env = SConscript("godot-cpp/SConstruct")

# (Cache-safe MSVC debug info used to be handled here, but the windows platform
#  block below re-appends /FS + /Zi afterwards and silently undid it. Moved to
#  _cache_safe_debug() further down, after the whole platform chain.)

# Add those directory manually, so we can skip the godot_cpp directory when including headers in C++ files.
# In reuse mode these must come from the REUSED godot-cpp tree (not this
# submodule's 4.5 one) or both versions' headers collide (redefinition errors).
_gc_base = _reuse_gc if _reuse_gc else "godot-cpp"
source_path = [
    os.path.join(_gc_base, "include","godot_cpp"),
    os.path.join(_gc_base, "gen", "include","godot_cpp")
]
env.Append(CPPPATH=[env.Dir(d) for d in source_path])

env.Replace(fmod_lib_dir = fmod_lib_dir)

# For the reference:
# - CCFLAGS are compilation flags shared between C and C++
# - CFLAGS are for C-specific compilation flags
# - CXXFLAGS are for C++-specific compilation flags
# - CPPFLAGS are for pre-processor flags
# - CPPDEFINES are for pre-processor defines
# - LINKFLAGS are for linking flags

# tweak this if you want to use different folders, or more folders, to store your source code in.
env.Append(CPPPATH=["src/"])
sources = [
    Glob('src/*.cpp'),
    Glob('src/callback/*.cpp'),
    Glob('src/core/*.cpp'),
    Glob('src/data/*.cpp'),
    Glob('src/tools/*.cpp'),
    Glob('src/helpers/*.cpp'),
    Glob('src/nodes/*.cpp'),
    Glob('src/resources/*.cpp'),
    Glob('src/studio/*.cpp'),
    Glob('src/plugins/*.cpp')
    ]

lfix = ""
debug = False
if env["target"] == "template_debug" or env["target"] == "editor":
    lfix = "L"
    debug = True

if env["platform"] == "macos":
    libfmod = 'libfmod%s.dylib' % lfix
    libfmodstudio = 'libfmodstudio%s.dylib' % lfix

    env.Append(CPPPATH=[env['fmod_lib_dir'] + 'osx/core/inc/', env['fmod_lib_dir'] + 'osx/studio/inc/'])
    env.Append(LIBPATH=[env['fmod_lib_dir'] + 'osx/core/lib/', env['fmod_lib_dir'] + 'osx/studio/lib/'])
    env.Append(LIBS=[libfmod, libfmodstudio])

    env.Append(
        LINKFLAGS=[
            "-framework",
            "Cocoa",
            "-Wl,-undefined,dynamic_lookup",
            "-rpath", "@loader_path/.."
        ]
    )

elif env["platform"] == "linux":
    libfmod = 'libfmod%s.so'% lfix
    libfmodstudio = 'libfmodstudio%s.so'% lfix

    env.Append(CPPPATH=[env['fmod_lib_dir'] + 'linux/core/inc/', env['fmod_lib_dir'] + 'linux/studio/inc/'])
    env.Append(LIBPATH=[env['fmod_lib_dir'] + 'linux/core/lib/' + env["arch"], env['fmod_lib_dir'] + 'linux/studio/lib/' + env["arch"]])
    env.Append(LIBS=[libfmod, libfmodstudio])

    env.Append(CCFLAGS=["-fPIC", "-Wwrite-strings"])
    env.Append(LINKFLAGS=["-Wl,-R,'$$ORIGIN'"])
    env.Append(LINKFLAGS=["-m64", "-fuse-ld=gold"])

elif env["platform"] == "windows":
    libfmod = 'fmod%s_vc'% lfix
    libfmodstudio = 'fmodstudio%s_vc'% lfix
    fmod_info_table = {
        "x86_64" : "x64",
        "x86_32" : "x86",
    }
    arch_suffix_override = fmod_info_table[env["arch"]]

    env.Append(CPPPATH=[env['fmod_lib_dir'] + 'windows/core/inc/', env['fmod_lib_dir'] + 'windows/studio/inc/'])
    env.Append(LIBPATH=[env['fmod_lib_dir'] + 'windows/core/lib/' + arch_suffix_override, env['fmod_lib_dir'] + 'windows/studio/lib/' + arch_suffix_override])
    env.Append(LIBS=[libfmod, libfmodstudio])

    env.Append(LINKFLAGS=["/WX"])
    if debug:
        env.Append(CCFLAGS=["/FS", "/Zi"])

elif env["platform"] == "ios":
    libfmod = 'libfmod%s_iphoneos.a' % lfix
    libfmodstudio = 'libfmodstudio%s_iphoneos.a' % lfix

    env.Append(CPPPATH=[env['fmod_lib_dir'] + 'ios/core/inc/', env['fmod_lib_dir'] + 'ios/studio/inc/'])
    env.Append(LIBPATH=[env['fmod_lib_dir'] + 'ios/core/lib/', env['fmod_lib_dir'] + 'ios/studio/lib/'])
    env.Append(LIBS=[libfmod, libfmodstudio])

    env.Append(LINKFLAGS=[
        '-Wl,-undefined,dynamic_lookup', "-miphoneos-version-min=" + env["ios_min_version"]
    ])

elif env["platform"] == "android":
    libfmod = 'libfmod%s.so' % lfix
    libfmodstudio = 'libfmodstudio%s.so' % lfix
    fmod_info_table = {
        "armv7": "armeabi-v7a",
        "arm64": "arm64-v8a",
        "x86": "x86",
        "x86_64": "x86_64"
    }
    arch_dir = fmod_info_table[env["arch"]]

    env.Append(CPPPATH=[env['fmod_lib_dir'] + 'android/core/inc/', env['fmod_lib_dir'] + 'android/studio/inc/'])
    env.Append(LIBPATH=[env['fmod_lib_dir'] + 'android/core/lib/' + arch_dir, env['fmod_lib_dir'] + 'android/studio/lib/' + arch_dir])
    env.Append(LIBS=[libfmod, libfmodstudio])

# --- Cache-safe MSVC debug info -------------------------------------------------
# Kept byte-identical with the copies in sim/SConstruct and in the sibling game
# repo. If you change one, change all of them.
#
# A build cache (SCons CacheDir / ccache) stores the .obj but NOT the separate .pdb
# that /Zi writes debug info into. A cache-HIT object therefore links with LNK4099
# ("PDB not found"), and godot-cpp's linker /WX promotes that to a fatal LNK1218
# with no output file. /Z7 EMBEDS debug info in the .obj, so a cached object is
# self-contained and links clean WITH full symbols.
#
# This rewrites only the debug-info FORMAT, never whether symbols are emitted: if
# nothing asked for debug info, none is added. No codegen or ABI impact, so every
# consumer links exactly as before. Idempotent. No-op off MSVC — GCC/Clang already
# embed DWARF in the .o, which is why Linux never hit this.
def _cache_safe_debug(e):
    if not (bool(e.get("is_msvc", False)) or str(e.get("CC", "")) == "cl"):
        return e
    wants_symbols = any(str(f) in ("/Zi", "/ZI", "/Z7") for f in e.get("CCFLAGS", []))
    # /FS is meaningless without /Zi; /ZI (edit-and-continue) also splits out a .pdb.
    e["CCFLAGS"] = [f for f in e.get("CCFLAGS", []) if str(f) not in ("/Zi", "/ZI", "/FS")]
    if wants_symbols:
        e.AppendUnique(CCFLAGS=["/Z7"])
    # Net for objects still carrying /Zi from an older cache entry.
    e.AppendUnique(LINKFLAGS=["/ignore:4099"])
    return e


# Placement here is load-bearing, and differs from sim/SConstruct on purpose: the
# windows block above re-appends /FS + /Zi on top of godot-cpp's, so this has to run
# after the whole platform chain. Keep it directly ahead of the target below.
_cache_safe_debug(env)

#Output is placed in the addons directory of the demo project directly
target = "{}{}/{}.{}.{}".format(
    target_path, env["platform"], target_name, env["platform"], env["target"]
) if env["platform"] != "android" else "{}{}/{}/{}.{}.{}".format(
    target_path, env["platform"], env["arch"], target_name, env["platform"], env["target"]
)

if env["platform"] == "macos":
    target = "{}.framework/{}.{}.{}".format(
        target,
        target_name,
        env["platform"],
        env["target"]
    )
else:
    target = "{}.{}{}".format(
        target,
        env["arch"],
        env["SHLIBSUFFIX"]
    )

library = env.SharedLibrary(target=target, source=sources)


def sys_exec(args):
    proc = subprocess.Popen(args, stdout=subprocess.PIPE, text=True)
    (out, err) = proc.communicate()
    return out.rstrip("\r\n").lstrip()


if env["platform"] == "ios":
    xcframework_path = "{}{}/{}.{}.{}.xcframework".format(
        target_path,
        env["platform"],
        target_name,
        env["platform"],
        env["target"]
    )

    def create_xcframework(self, arg, env, executor = None):
        sys_exec(["xcodebuild", "-create-xcframework", "-library", target, "-output", xcframework_path])
        sys_exec(["rm", target])
        sys_exec(["/usr/libexec/PlistBuddy", "-c", "Add :MinimumOSVersion string " + env["ios_min_version"], "{}/Info.plist".format(xcframework_path)])

    create_xcframework_action = Action('', create_xcframework)

    AddPostAction(library, create_xcframework_action)


def copy_fmod_libraries(self, arg, env, executor = None):
    fmod_core_lib_dir = ""
    fmod_studio_lib_dir = ""

    addon_fmod_libs_output = "{}{}/".format(
        target_path, env["platform"]
    ) if env["platform"] != "android" else "{}{}/{}/".format(
        target_path, env["platform"], env["arch"]
    )

    if env["platform"] == "macos":
        fmod_core_lib_dir = env['fmod_lib_dir'] + 'osx/core/lib/'
        fmod_studio_lib_dir = env['fmod_lib_dir'] + 'osx/studio/lib/'
    elif env["platform"] == "linux":
        fmod_core_lib_dir = env['fmod_lib_dir'] + 'linux/core/lib/' + env["arch"]
        fmod_studio_lib_dir = env['fmod_lib_dir'] + 'linux/studio/lib/' + env["arch"]
    elif env["platform"] == "windows":
        fmod_core_lib_dir = env['fmod_lib_dir'] + 'windows/core/lib/' + arch_suffix_override + '/'
        fmod_studio_lib_dir = env['fmod_lib_dir'] + 'windows/studio/lib/' + arch_suffix_override + '/'
    elif env["platform"] == "ios":
        fmod_core_lib_dir = env['fmod_lib_dir'] + 'ios/core/lib/'
        fmod_studio_lib_dir = env['fmod_lib_dir'] + 'ios/studio/lib/'
    elif env["platform"] == "android":
        fmod_core_lib_dir = env['fmod_lib_dir'] + 'android/core/lib/' + arch_dir
        fmod_studio_lib_dir = env['fmod_lib_dir'] + 'android/studio/lib/' + arch_dir

    source_files = [env.Glob(os.path.join(source_dir, '*.*')) for source_dir in [fmod_core_lib_dir, fmod_studio_lib_dir]]
    [[shutil.copy(str(file), addon_fmod_libs_output) for file in files] for files in source_files]


copy_fmod_libraries_action = Action('', copy_fmod_libraries)
AddPostAction(library, copy_fmod_libraries_action)

Default(library)