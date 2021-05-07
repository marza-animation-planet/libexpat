import os
import sys
import excons
import excons.cmake
import SCons.Script # pylint: disable=import-error


env = excons.MakeBaseEnv()


def ExpatName(static=True):
   if static and sys.platform == "win32":
      return "libexpat"
   else:
      return "expat"

def ExpatPath(static=True):
   name = ExpatName(static)
   if sys.platform == "win32":
      libname = name + ".lib"
   else:
      libname = "lib" + name + (".a" if static else excons.SharedLibraryLinkExt())
   return excons.OutputBaseDirectory() + "/lib/" + libname

def RequireExpat(env, static=True):
   if static:
      env.Append(CPPDEFINES=["XML_STATIC"])
   env.Append(CPPPATH=[excons.OutputBaseDirectory() + "/include"])
   env.Append(LIBPATH=[excons.OutputBaseDirectory() + "/lib"])
   excons.Link(env, ExpatPath(static), static=static, force=True, silent=True)


_static = (excons.GetArgument("expat-static", 1, int) != 0)

if sys.platform != "win32":
   envcf = os.environ.get("CFLAGS", "")
   if not "-fPIC" in envcf:
      envcf += " -fPIC"
      os.environ["CFLAGS"] = envcf

prjs = [
   {
      "name": "expat",
      "type": "cmake",
      "cmake-root": "expat",
      "cmake-opts": {"EXPAT_BUILD_TOOLS": 0,
                     "EXPAT_BUILD_EXAMPLES": 0,
                     "EXPAT_BUILD_TESTS": 0,
                     "EXPAT_BUILD_DOCS": 0,
                     "EXPAT_ENABLE_INSTALL": 1,
                     "EXPAT_BUILD_PKGCONFIG": 0,
                     "EXPAT_SHARED_LIBS": 0 if _static else 1,
                     "EXPAT_LARGE_SIZE": 1,
                     "CMAKE_INSTALL_LIBDIR": "lib"},
      "cmake-cfgs": excons.CollectFiles(".", patterns=["CMakeLists.txt"], recursive=True),
      "cmake-srcs": excons.CollectFiles("expat/src", patterns=["*.c"]),
      "cmake-outputs": ["include/expat.h",
                        "include/expat_config.h",
                        "include/expat_external.h",
                        ExpatPath(static=_static)]
   }
]

excons.AddHelpOptions(expat="""EXPAT OPTIONS
  expat-static=0|1   : Toggle between static and shared library build [1]""")

excons.DeclareTargets(env, prjs)

SCons.Script.Export("ExpatName ExpatPath RequireExpat")
