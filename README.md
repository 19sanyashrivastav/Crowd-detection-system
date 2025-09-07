# Crowd-detection-system
# OpenCV Python binary extension loader (cleaned & fixed)
import os
import importlib
import sys

_all_ = []

# Ensure numpy exists
try:
    import numpy  # noqa: F401
except ImportError:
    print('OpenCV bindings requires "numpy" package.')
    print('Install it via command:')
    print('    pip install numpy')
    raise

# Helper to detect 64-bit Python (unused currently but handy)
is_x64 = sys.maxsize > 2**32

def __load_extra_py_code_for_module(base, name, enable_debug_print=False):
    """
    Load additional pure-Python code for a submodule (if present).
    If a compiled extension of the same module exists in sys.modules,
    copy missing symbols from the native module into the Python module.
    """
    module_name = "{}.{}".format(_name_, name)
    export_module_name = "{}.{}".format(base, name)
    native_module = sys.modules.pop(module_name, None)
    try:
        py_module = importlib.import_module(module_name)
    except ImportError as err:
        if enable_debug_print:
            print("Can't load Python code for module:", module_name, ". Reason:", err)
        # Extension doesn't contain extra py code
        return False

    if base in sys.modules and not hasattr(sys.modules[base], name):
        setattr(sys.modules[base], name, py_module)
    sys.modules[export_module_name] = py_module

    # If it is C extension module it is already loaded by cv2 package
    if native_module:
        setattr(py_module, "_native", native_module)
        # copy symbols that don't already exist in py_module
        for k, v in filter(lambda kv: not hasattr(py_module, kv[0]), native_module._dict_.items()):
            if enable_debug_print:
                print('    symbol({}): {} = {}'.format(name, k, v))
            setattr(py_module, k, v)
    return True


def __collect_extra_submodules(enable_debug_print=False):
    """
    Collect names of extra submodule directories located next to this loader.
    Only used for Python 3.
    Returns a list of submodule names (strings).
    """
    if sys.version_info[0] < 3:
        if enable_debug_print:
            print("Extra submodules are loaded only for Python 3")
        return []

    _INIT_FILE_PATH = os.path.abspath("C:\Users\hp\OneDrive\Desktop\iitk\detect_people.py"_)
    extra_submodules_init_path = os.path.dirname(_INIT_FILE_PATH)

    def modules_filter(module):
        # skip internal modules and files; keep directories only
        if module.startswith("_"):
            return False
        if module.startswith("python-"):
            return False
        full = os.path.join(_extra_submodules_init_path, module)
        return os.path.isdir(full)

    try:
        entries = os.listdir(_extra_submodules_init_path)
    except OSError:
        if enable_debug_print:
            print("OpenCV loader: can't list dir:", _extra_submodules_init_path)
        return []

    # convert to list, sorted for deterministic behavior
    return list(filter(modules_filter, sorted(entries)))


def bootstrap():
    import copy
    save_sys_path = copy.copy(sys.path)

    # simple recursion protection
    if hasattr(sys, 'OpenCV_LOADER'):
        if hasattr(sys, 'OpenCV_LOADER_DEBUG'):
            print(sys.path)
        raise ImportError('ERROR: recursion is detected during loading of "cv2" binary extensions. Check OpenCV installation.')
    sys.OpenCV_LOADER = True

    DEBUG = bool(getattr(sys, 'OpenCV_LOADER_DEBUG', False))

    import platform
    if DEBUG:
        print('OpenCV loader: os.name="{}"  platform.system()="{}"'.format(os.name, str(platform.system())))

    LOADER_DIR = os.path.dirname(os.path.abspath(os.path.realpath("C:\Users\hp\OneDrive\Desktop\iitk\detect_people.py")))

    # will be populated by config files
    PYTHON_EXTENSIONS_PATHS = []
    BINARIES_PATHS = []

    g_vars = globals()
    l_vars = locals().copy()

    # load appropriate exec wrapper (py2/py3)
    if sys.version_info[:2] < (3, 0):
        from .load_config_py2 import exec_file_wrapper  # corrected import
    else:
        from .load_config_py3 import exec_file_wrapper  # corrected import

    def load_first_config(fnames, required=True):
        for fname in fnames:
            fpath = os.path.join(LOADER_DIR, fname)
            if not os.path.exists(fpath):
                if DEBUG:
                    print('OpenCV loader: config not found, skip: {}'.format(fpath))
                continue
            if DEBUG:
                print('OpenCV loader: loading config: {}'.format(fpath))
            exec_file_wrapper(fpath, g_vars, l_vars)
            return True
        if required:
            raise ImportError('OpenCV loader: missing configuration file: {}. Check OpenCV installation.'.format(fnames))
        return False

    # load main config (raises if missing)
    load_first_config(['config.py'], True)

    # load version-specific config
    load_first_config([
        'config-{}.{}.py'.format(sys.version_info[0], sys.version_info[1]),
        'config-{}.py'.format(sys.version_info[0])
    ], True)

    # read from locals updated by exec_file_wrapper
    PYTHON_EXTENSIONS_PATHS = l_vars.get('PYTHON_EXTENSIONS_PATHS', [])
    BINARIES_PATHS = l_vars.get('BINARIES_PATHS', [])

    if DEBUG:
        print('OpenCV loader: PYTHON_EXTENSIONS_PATHS={}'.format(str(PYTHON_EXTENSIONS_PATHS)))
        print('OpenCV loader: BINARIES_PATHS={}'.format(str(BINARIES_PATHS)))

    applySysPathWorkaround = False
    if hasattr(sys, 'OpenCV_REPLACE_SYS_PATH_0'):
        applySysPathWorkaround = True
    else:
        try:
            BASE_DIR = os.path.dirname(LOADER_DIR)
            if sys.path and (sys.path[0] == BASE_DIR or os.path.realpath(sys.path[0]) == BASE_DIR):
                applySysPathWorkaround = True
        except Exception:
            if DEBUG:
                print('OpenCV loader: exception during checking workaround for sys.path[0]')
            applySysPathWorkaround = False

    # insert extension paths (reverse order to preserve declared priority)
    for p in reversed(PYTHON_EXTENSIONS_PATHS):
        if applySysPathWorkaround:
            sys.path.insert(0, p)
        else:
            # keep default sys.path[0] intact
            sys.path.insert(1, p)

    # Windows DLL handling (Python >= 3.8 add_dll_directory)
    if os.name == 'nt':
        if sys.version_info[:2] >= (3, 8):
            for p in BINARIES_PATHS:
                try:
                    os.add_dll_directory(p)
                except Exception as e:
                    if DEBUG:
                        print('Failed os.add_dll_directory(): ' + str(e))
                    # continue trying to set PATH
                    pass
        os.environ['PATH'] = ';'.join(BINARIES_PATHS) + ';' + os.environ.get('PATH', '')
        if DEBUG:
            print('OpenCV loader: PATH={}'.format(str(os.environ['PATH'])))
    else:
        # For UNIX-like systems, amending LD_LIBRARY_PATH affects child processes only
        os.environ['LD_LIBRARY_PATH'] = ':'.join(BINARIES_PATHS) + ':' + os.environ.get('LD_LIBRARY_PATH', '')

    if DEBUG:
        print("Relink everything from native cv2 module to cv2 package")

    # swap temporary python package module with native extension module
    py_module = sys.modules.pop("cv2", None)

    # import native extension (this should load compiled cv2.* extension)
    native_module = importlib.import_module("cv2")

    # restore py package module in sys.modules
    if py_module is not None:
        sys.modules["cv2"] = py_module
        setattr(py_module, "_native", native_module)

        # copy items from native module into loader's globals if not present
        for item_name, item in filter(lambda kv: kv[0] not in ("_file", "loader", "spec", "name", "package"), native_module.dict_.items()):
            if item_name not in g_vars:
                g_vars[item_name] = item
    else:
        # If loader wasn't a package module (unlikely), ensure sys.modules has native module
        sys.modules["cv2"] = native_module

    # restore sys.path so multiprocessing child starts from original environment
    sys.path = save_sys_path

    try:
        del sys.OpenCV_LOADER
    except Exception as e:
        if DEBUG:
            print("Exception during delete OpenCV_LOADER:", e)

    if DEBUG:
        print('OpenCV loader: binary extension... OK')

    # load any extra pure-Python submodules located next to loader
    for submodule in __collect_extra_submodules(DEBUG):
        try:
            if __load_extra_py_code_for_module("cv2", submodule, DEBUG):
                if DEBUG:
                    print("Extra Python code for", submodule, "is loaded")
        except Exception as e:
            if DEBUG:
                print("Failed loading extra submodule", submodule, ":", e)

    if DEBUG:
        print('OpenCV loader: DONE')


# Run bootstrap on import
bootstrap()
