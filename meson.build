project('libfprint', [ 'c', 'cpp' ],
    version: '1.90.1',
    license: 'LGPLv2.1+',
    default_options: [
        'buildtype=debugoptimized',
        'warning_level=1',
        'c_std=gnu99',
    ],
    meson_version: '>= 0.49.0')

gnome = import('gnome')

libfprint_conf = configuration_data()

cc = meson.get_compiler('c')
cpp = meson.get_compiler('cpp')
host_system = host_machine.system()
glib_min_version = '2.56'

glib_version_def = 'GLIB_VERSION_@0@_@1@'.format(
    glib_min_version.split('.')[0], glib_min_version.split('.')[1])
common_cflags = cc.get_supported_arguments([
    '-Wall',
    '-Wcast-align',
    '-Wformat-nonliteral',
    '-Wformat-security',
    '-Wformat=2',
    '-Wignored-qualifiers',
    '-Wlogical-op',
    '-Wmissing-declarations',
    '-Wmissing-format-attribute',
    '-Wmissing-include-dirs',
    '-Wmissing-noreturn',
    '-Wpointer-arith',
    '-Wshadow',
    '-Wtype-limits',
    '-Wundef',
    '-Wunused',
    '-Werror=address',
    '-Werror=array-bounds',
    '-Werror=empty-body',
    '-Werror=init-self',
    '-Werror=int-to-pointer-cast',
    '-Werror=main',
    '-Werror=missing-braces',
    '-Werror=nonnull',
    '-Werror=redundant-decls',
    '-Werror=return-type',
    '-Werror=sequence-point',
    '-Werror=trigraphs',
    '-Werror=write-strings',
    '-fno-strict-aliasing',
    '-DGLIB_VERSION_MIN_REQUIRED=' + glib_version_def,
    '-DGLIB_VERSION_MAX_ALLOWED=' + glib_version_def,
    '-D_GNU_SOURCE',
    '-DG_LOG_DOMAIN="@0@"'.format(meson.project_name()),
])
c_cflags = cc.get_supported_arguments([
    '-Wimplicit-function-declaration',
    '-Wmissing-prototypes',
    '-Wnested-externs',
    '-Wold-style-definition',
    '-Wstrict-prototypes',
    '-Werror=implicit',
    '-Werror=pointer-to-int-cast',
])
add_project_arguments(common_cflags + c_cflags, language: 'c')
add_project_arguments(common_cflags, language: 'cpp')

# maintaining compatibility with the previous libtool versioning
# current = binary - interface
# revision = interface
soversion = 2
current = 0
revision = 0
libversion = '@0@.@1@.@2@'.format(soversion, current, revision)
versioned_libname = meson.project_name() + '-' + soversion.to_string()

# Dependencies
glib_dep = dependency('glib-2.0', version: '>=' + glib_min_version)
gio_dep = dependency('gio-unix-2.0', version: '>=' + glib_min_version)
gobject_dep = dependency('gobject-2.0', version: '>=' + glib_min_version)
gusb_dep = dependency('gusb', version: '>= 0.2.0')
mathlib_dep = cc.find_library('m', required: false)

# The following dependencies are only used for tests
cairo_dep = dependency('cairo', required: false)

# Drivers
drivers = get_option('drivers').split(',')
virtual_drivers = [ 'virtual_image' ]
default_drivers = [
    'upektc_img',
    'vfs5011',
    'aes3500',
    'aes4000',
    'aes1610',
    'aes1660',
    'aes2660',
    'aes2501',
    'aes2550',
    'vfs101',
    'vfs301',
    'vfs0050',
    'vfs0090',
    'etes603',
    'vcom5s',
    'synaptics',
    'elan',
    'uru4000',
    'upektc',
    'upeksonly',
    'upekts',
]

all_drivers = default_drivers + virtual_drivers

if drivers == [ 'all' ]
    drivers = all_drivers
endif

if drivers == [ 'default' ]
    drivers = default_drivers
endif

if drivers.length() == 0 or drivers[0] == ''
    error('Cannot build libfprint without drivers, please specify a valid value for the drivers option')
endif

nss_dep = dependency('', required: false)
openssl_dep = dependency('', required: false)
imaging_dep = dependency('', required: false)
libfprint_conf.set10('HAVE_PIXMAN', false)
foreach driver: drivers
    if driver == 'uru4000'
        nss_dep = dependency('nss', required: false)
        if not nss_dep.found()
            error('NSS is required for the URU4000/URU4500 driver')
        endif
    endif
    if driver == 'aes3500' or driver == 'aes4000'
        imaging_dep = dependency('pixman-1', required: false)
        if not imaging_dep.found()
            error('pixman is required for imaging support')
        endif

        libfprint_conf.set10('HAVE_PIXMAN', true)
    endif
    if driver == 'vfs0090'
        nss_dep = dependency('nss', required: false)
        if not nss_dep.found()
            error('NSS is required for the Validity VFS009x drivers')
        endif
        openssl_dep = dependency('openssl', required: false)
        if not openssl_dep.found()
            error('OpenSSL is required for the Validity VFS009x drivers')
        endif
        imaging_dep = dependency('pixman-1', required: false)
        if not imaging_dep.found()
            error('pixman is required for imaging support')
        endif

        git = find_program('git', required: false)
        if git.found()
            message('Updating the vfs0090 submodule')
            c = run_command(git, ['submodule', 'update', '--init', '--recursive'])

            if c.returncode() != 0
                error('Impossible to update the vfs0090 submodule')
            endif
        else
            message('Ensure you updated the vfs0090 submodule!')
        endif
    endif
    if not all_drivers.contains(driver)
        error('Invalid driver \'' + driver + '\'')
    endif
endforeach

# Export the drivers' types to the core code
drivers_type_list = []
drivers_type_func = []
drivers_type_list += '#include <glib-object.h>'
drivers_type_list += '#include "fpi-context.h"'
drivers_type_list += ''
drivers_type_func += 'GArray *'
drivers_type_func += 'fpi_get_driver_types (void)'
drivers_type_func += '{'
drivers_type_func += '  GArray *drivers = g_array_new (TRUE, FALSE, sizeof (GType));'
drivers_type_func += '  GType t;'
drivers_type_func += ''
foreach driver: drivers
    drivers_type_list += 'extern GType (fpi_device_' + driver + '_get_type) (void);'
    drivers_type_func += '  t = fpi_device_' + driver + '_get_type ();'
    drivers_type_func += '  g_array_append_val (drivers, t);'
    drivers_type_func += ''
endforeach
drivers_type_list += ''
drivers_type_func += '  return drivers;'
drivers_type_func += '}'

root_inc = include_directories('.')

if get_option('udev_rules')
    udev_rules_dir = get_option('udev_rules_dir')

    if udev_rules_dir == 'auto'
        udev_dep = dependency('udev')
        udev_rules_dir = udev_dep.get_pkgconfig_variable('udevdir') + '/rules.d'
    endif
endif

if get_option('gtk-examples')
    gnome = import('gnome')

    gtk_dep = dependency('gtk+-3.0', required: false)
    if not gtk_dep.found()
        error('GTK+ 3.x is required for GTK+ examples')
    endif
endif

configure_file(output: 'config.h', configuration: libfprint_conf)

subdir('libfprint')
subdir('examples')
if get_option('doc')
    gnome = import('gnome')
    subdir('doc')
endif
if get_option('gtk-examples')
    subdir('demo')
endif

subdir('tests')

pkgconfig = import('pkgconfig')
pkgconfig.generate(
    name: versioned_libname,
    description: 'Generic C API for fingerprint reader access',
    version: meson.project_version(),
    libraries: libfprint,
    requires: [gio_dep, gobject_dep],
    subdirs: versioned_libname,
    filebase: versioned_libname,
)
